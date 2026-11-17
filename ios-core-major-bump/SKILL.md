---
name: ios-core-major-bump
description: Propagate a breaking (major) release of an Infomaniak iOS Swift package, usually ios-core, through every dependent Infomaniak package (ios-core-ui, ios-core-uikit, ios-notifications, ios-version-checker, ios-create-account, ios-bug-tracker, ios-features…) in dependency order, tag the new majors, then update the app (Calendar, Mail, kDrive, SwissTransfer, Euria, Authenticator). Use when the user released a new major of ios-core (or another shared package) and wants an app to target it.
---

# ios-core major bump cascade

When a shared package such as `ios-core` gets a new major, SwiftPM cannot resolve an app until **every** Infomaniak package depending on it, directly or transitively, has a new major that requires it. This skill releases those packages layer by layer, then updates the app.

## Inputs

- **Root package and new version**, e.g. `ios-core` `20.0.0`. Check that the tag exists: `gh api repos/Infomaniak/ios-core/git/ref/tags/20.0.0`.
- **Target app repository**, e.g. `ios-calendar`, `ios-kMail` or `ios-kDrive`. Its dependencies are in `Tuist/Package.swift`.
- Ask the user once, with `ask_user`, before changing anything:
  1. **How to land changes**: open one PR per repo and wait for the user to merge it (default), or push directly to the default branch.
  2. **Who to notify** about the app PR, if anyone (for example a kChat username).

Do not rely on a list of packages supplied by the user. Always compute the graph, because users often forget transitive dependents such as `ios-core-uikit`, which only depends on `ios-core-ui`.

## 1. Compute the dependency graph and next versions

Run this script, passing the root package as the argument. It reads every non-archived Infomaniak repo's `Package.swift` or `Tuist/Package.swift` on its default branch and prints:

- the affected packages, grouped by layer (all packages in a layer can be done in parallel)
- the latest semver tag and the next major for each package
- the apps to update last

```bash
cat > /tmp/ik-graph.py <<'EOF'
import json, re, subprocess, sys
from concurrent.futures import ThreadPoolExecutor
ROOT, ORG = (sys.argv[1] if len(sys.argv) > 1 else "ios-core"), "Infomaniak"
def gh(*a):
    r = subprocess.run(["gh", *a], capture_output=True, text=True)
    return r.stdout if r.returncode == 0 else ""
repos = json.loads(gh("repo", "list", ORG, "--limit", "1000", "--no-archived", "--json", "name,defaultBranchRef"))
def manifest(repo):
    n, b = repo["name"], (repo.get("defaultBranchRef") or {}).get("name", "main")
    for p in ("Package.swift", "Tuist/Package.swift"):
        raw = gh("api", "-H", "Accept: application/vnd.github.raw", f"repos/{ORG}/{n}/contents/{p}?ref={b}")
        if raw:
            return n, p, b, dict(re.findall(rf'github\.com/{ORG}/([\w.-]+?)(?:\.git)?",\s*([^)]*\))', raw))
    return n, None, b, {}
with ThreadPoolExecutor(16) as pool:
    ms = [m for m in pool.map(manifest, repos) if m[1]]
deps, info = {n: d for n, _, _, d in ms}, {n: (p, b) for n, p, b, _ in ms}
aff, changed = {ROOT}, True
while changed:
    changed = False
    for n, d in deps.items():
        if n not in aff and aff & d.keys(): aff.add(n); changed = True
aff.discard(ROOT)
def latest(n):
    t = [x for x in gh("api", f"repos/{ORG}/{n}/tags", "--paginate", "--jq", ".[].name").split() if re.fullmatch(r"\d+\.\d+\.\d+", x)]
    return max(t, key=lambda x: tuple(map(int, x.split("."))), default=None)
pk = sorted(n for n in aff if info[n][0] == "Package.swift")
apps = sorted(n for n in aff if info[n][0] != "Package.swift")
lvl = {}
def depth(n):
    if n not in lvl: lvl[n] = 1 + max((depth(d) for d in deps[n] if d in pk), default=0)
    return lvl[n]
print("Packages (release layer by layer):")
for n in sorted(pk, key=lambda n: (depth(n), n)):
    t = latest(n); nx = f"{int(t.split('.')[0]) + 1}.0.0" if t else "?"
    print(f"  L{depth(n)} {n:24} latest={t} next={nx} branch={info[n][1]} bumps: {', '.join(sorted(d for d in deps[n] if d in aff | {ROOT}))}")
print("Apps (Tuist/Package.swift):")
for n in apps: print(f"  {n:24} uses: {', '.join(sorted(d for d in deps[n] if d in aff | {ROOT}))}")
EOF
python3 /tmp/ik-graph.py ios-core
```

Check the output against these rules before continuing:

- **Versions come from tags, not GitHub Releases.** Most packages publish annotated tags only, so `releases/latest` is stale.
- **Check whether a package has already been bumped.** If its latest tag already requires the new root major (`gh api "repos/Infomaniak/<repo>/contents/Package.swift?ref=<tag>"`), skip it.
- Default branches differ between repos (`ios-bug-tracker` and `ios-core` use `master`). Always read the default branch from the output, never assume `main`.

Track every repo in the session SQL `todos` table, with `todo_deps` mirroring the layers.

## 2. Identify the source migrations

Diff the root package between its previous tag and the new one, to find changes that require code edits in dependents:

```bash
gh repo clone Infomaniak/ios-core /tmp/ik-bump/ios-core -- -q
git -C /tmp/ik-bump/ios-core --no-pager log --oneline <previous>..<new>
git -C /tmp/ik-bump/ios-core --no-pager diff <previous>..<new> -- Package.swift Sources
```

Look for renamed or removed public API, and for changes to products or transitive modules. For example, in ios-core 20.0.0, `Sentry-Dynamic` was replaced by `SentrySPM`, so every consumer has to change `import Sentry` to `import SentrySwift`. Grep each affected repo and the app for the affected symbols or imports, then fix them in the same PR as the version bump.

## 3. Release each layer

Work in fresh clones under `/tmp/ik-bump/`. **Never modify the user's working checkout**, even if it contains in-progress edits. Do not use `.package(path:)` dependencies.

For each repo in the current layer:

```bash
cd /tmp/ik-bump && gh repo clone Infomaniak/<repo> -- -q && cd <repo>
git checkout -q -b chore/bump-ios-core-<major>
# Bump the root and every package released in earlier layers (repeat -e for each one)
sed -i '' \
  -e 's#\(Infomaniak/ios-core", .upToNextMajor(from: "\)[0-9.]*"#\120.0.0"#' \
  -e 's#\(Infomaniak/ios-core-ui", .upToNextMajor(from: "\)[0-9.]*"#\128.0.0"#' \
  Package.swift
git --no-pager diff Package.swift   # each bumped line must change; the sed is a silent no-op if the pattern doesn't match
swift package resolve
```

Build for the iOS Simulator and rely on the **exit code**:

```bash
xcodebuild -list            # pick the scheme
xcodebuild build -scheme <scheme> -destination 'generic/platform=iOS Simulator' \
  -skipPackagePluginValidation -skipMacroValidation -quiet
echo "exit=$?"
```

- **Choosing the scheme:** use `<PackageName>-Package` when it exists, otherwise use the package's own library scheme (for example `VersionChecker` or `InfomaniakNotifications`). Never pick the first scheme alphabetically, because it is usually a third-party dependency such as `Alamofire` or `DesignSystem`, and building it proves nothing.
- `error: the following command failed with exit code 0 but produced no further output` is xcodebuild noise. If the exit code is 0, the build passed.

Commit `Package.swift`, `Package.resolved` and any source migrations. Use a signed Conventional Commit, with no AI attribution unless the user's commit conventions say otherwise:

```bash
git add Package.swift Package.resolved <migrated files>
git commit -S -m "chore: Update ios-core to 20.0.0"
git push -q -u origin HEAD
gh pr create -B <default-branch> -t "chore: Update ios-core to 20.0.0" \
  -b "Bump InfomaniakCore to 20.0.0 (<one-line summary of the breaking change>). Requires a new major release."
```

For a repo that only bumps another package (such as `ios-core-uikit` → `ios-core-ui`), use a title that names that package, for example `chore: Update ios-core-ui to 28.0.0`.

Give the user every PR link of the layer and **stop until they confirm the merge**. Then, for each repo:

```bash
git checkout -q <default-branch> && git pull -q
grep -q '"20.0.0"' Package.swift            # make sure the merge contains the bump
git cat-file -t <previous-tag>              # existing tags are annotated ("tag"); match that style
git tag -s <next-major> -m "<next-major>"   # plain semver, no "v" prefix
git push -q origin <next-major>
```

- Tags are protected by a ruleset. `Bypassed rule violations for refs/tags/...` means the push **succeeded**, using the maintainer's bypass. If the push is rejected instead, ask the user to push the tag.
- Only create a GitHub Release if the repo's latest tag already has one.
- A package may move to the next layer as soon as **all of its own** dependencies are tagged. It doesn't need to wait for unrelated packages in the same layer. For example, `ios-features` doesn't depend on `ios-bug-tracker`.

## 4. Update the app

Bump each affected package in `Tuist/Package.swift` to its new major. Leave packages the app doesn't reference alone. Apply the source migrations from step 2, then resolve and build:

```bash
cd /tmp/ik-bump && gh repo clone Infomaniak/<app> -- -q && cd <app>
git checkout -q -b chore/bump-ios-core-<major>
# sed bumps on Tuist/Package.swift, plus the source migrations
mise trust -q . && mise install
mise exec -- tuist install && mise exec -- tuist generate --no-open
mise exec -- tuist build <AppScheme>        # tuist prints compile errors prefixed with [x]
```

If the build fails, find out whether the bump caused it before blaming it:

1. **Compare with a baseline.** Stash the changes, rebuild `main` the same way, then `git stash pop`. Errors that also occur on `main` (for example `Multiple commands produce …DotLottiePlayer.framework`) already existed. Report them; don't fix them in this PR.
2. **Isolate the edited modules.** Build the framework scheme that contains the migrated files with xcodebuild against the generated workspace. That proves the bump compiles even when the full app target is broken.
3. **Missing KMP types.** If an error says a type is missing from a Kotlin Multiplatform module (such as `MultiplatformCalendar`), the xcframework may only have an arm64 simulator slice. Rebuild with `ARCHS=arm64 ONLY_ACTIVE_ARCH=YES`.

Commit with the same message style, open the PR, and send its link to the reviewer the user named. For example, with the kChat MCP: `kchat_send_direct_message(username: "<user>", text: "…could you review…? <PR link>")`.

## Exit criteria

- Every affected package has a merged PR and a signed tag at its next major. Verify the tags by running the graph script again: each `latest` shows the new major.
- The app PR is open, `Package.resolved` contains the new root version, and the build either passes or fails only on errors that also occur on `main`.
- Final report: one line per repo with its PR and tag, the app PR link, pre-existing build issues, and confirmation that the user's local checkout was not touched.

## Worked example: ios-core 20.0.0 (November 2026)

| Layer | Repo | Tag | Notes |
|-------|------|-----|-------|
| 1 | ios-core-ui | 28.0.0 | |
| 1 | ios-notifications | 17.0.0 | |
| 2 | ios-version-checker | 20.0.0 | |
| 2 | ios-create-account | 27.0.0 | |
| 2 | ios-core-uikit | 4.0.0 | Transitive only, through ios-core-ui. It was missing from the user's list. |
| 3 | ios-bug-tracker | 21.0.0 | Default branch `master` |
| 3 | ios-features | 12.0.0 | |
| app | ios-calendar | — | `import Sentry` → `import SentrySwift` in 3 files |
