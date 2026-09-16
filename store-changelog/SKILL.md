---
name: store-changelog
description: Generate concise, user-oriented App Store or Google Play release notes between two Git tags. Infer the target platform from the repository, account for platform-specific technologies, and output English, French, German, Spanish, and Italian. Use when the user asks for a store changelog, release notes, or a What's New text from a tag or release range.
---

# Store Changelog

Generate paste-ready release notes for Apple App Store, Google Play, or both from the changes between two Git tags.

## Inputs

Identify:

- The older/base tag.
- The newer/target tag.
- The target store or platform when explicitly stated.
- Any languages explicitly requested by the user.

If the user does not specify languages, output:

1. English
2. French
3. German
4. Spanish
5. Italian

If tag order is ambiguous, determine it from Git history. Ask the user only if neither tag is an ancestor of the other.

## Workflow

### 1. Detect the project platform

Infer the supported platform from repository evidence before writing the changelog.

Apple indicators include:

- `.xcodeproj`, `.xcworkspace`, `Package.swift`, `Project.swift`, or `Tuist/`.
- Swift or Objective-C application targets.
- `Info.plist`, entitlements, App Store metadata, or TestFlight notes.
- Apple frameworks and features such as SwiftUI, UIKit, AppKit, Catalyst, Siri, Shortcuts, Spotlight, App Intents, WidgetKit, watchOS, tvOS, or visionOS.

Google/Android indicators include:

- `build.gradle`, `build.gradle.kts`, `settings.gradle`, or `settings.gradle.kts`.
- `AndroidManifest.xml`, Android application modules, or Play metadata.
- Kotlin or Java Android sources.
- Android features such as Jetpack Compose, App Actions, Google Assistant integrations, Android widgets, Wear OS, Android Auto, or platform SDK changes.

Cross-platform indicators include:

- Both native Apple and Android projects.
- Flutter, React Native, Kotlin Multiplatform, .NET MAUI, Unity, or another framework with multiple platform targets.

Use repository structure, build manifests, target configuration, changed files, and existing release metadata together. Do not classify a project from programming language alone.

If only Apple indicators are present, produce App Store notes. If only Android indicators are present, produce Google Play notes. If both are present, determine which platforms the compared changes affect.

If the user explicitly names a store, use it when supported; if it conflicts with repository evidence, explain the mismatch and ask before switching stores. Infer the target store only when the user did not specify one.

### 2. Validate the range

Verify both tags exist:

```bash
git rev-parse --verify "<tag>^{commit}"
```

Determine their relationship:

```bash
git merge-base --is-ancestor "<older-tag>" "<newer-tag>"  # older -> newer
git merge-base --is-ancestor "<newer-tag>" "<older-tag>"  # newer -> older

Use the range:

```text
<older-tag>..<newer-tag>
```

### 3. Gather release evidence

Inspect all of the following:

```bash
git log --first-parent --pretty=format:'%h%x09%s' "<older-tag>..<newer-tag>"
git log --no-merges --pretty=format:'%h%x09%s' "<older-tag>..<newer-tag>"
git diff --stat "<older-tag>..<newer-tag>"
git diff --name-only "<older-tag>..<newer-tag>"
```

Also inspect existing release-note files when present, including:

- `TestFlight/WhatToTest.*`
- Fastlane or App Store Connect metadata
- Google Play or Gradle Play Publisher metadata
- `fastlane/metadata/android/*/changelogs/`
- `whatsnew`, release-note, or localized store metadata
- `CHANGELOG*`
- Release configuration or localized release-note files

Read focused diffs when a commit title does not establish the actual user impact. Do not infer features from filenames alone.

### 4. Classify changes by platform

For each candidate user-visible change, determine whether it applies to:

- Apple platforms only.
- Android/Google platforms only.
- Both platforms.
- A narrower platform such as Mac, iPad, Apple Watch, Wear OS, or Android Auto.

Trace the implementation through changed targets, manifests, build guards, availability checks, imports, and directly called helpers. Examples:

- Siri, Shortcuts, Spotlight, App Intents, SwiftUI, UIKit, Catalyst, and Apple framework changes are Apple-specific unless a separate Android implementation is present.
- Google Assistant, App Actions, Jetpack Compose, Android intents, Play Services, and Android framework changes are Android-specific unless a separate Apple implementation is present.
- Shared business logic is not automatically user-visible on every platform; confirm that each application target consumes the changed behavior.
- A dependency or shared-module change may affect both stores only when both shipped applications use it.

Never translate a platform-specific technology into the other platform's nearest equivalent. For example, do not turn a Siri improvement into a Google Assistant improvement unless the range contains a Google Assistant change.

When producing notes for one store, omit changes exclusive to the other store. When producing notes for both stores, create separate store sections if their user-visible changes differ.

### 5. Select user-visible changes

Include:

- New capabilities users can directly access.
- Meaningful improvements to existing workflows.
- User-visible bug fixes.
- Platform-specific improvements when relevant.
- Translation, accessibility, performance, or stability improvements when supported by the changes.

Exclude:

- CI, build, dependency, lint, formatting, and release automation changes.
- Refactors without observable behavior changes.
- Debug-only or internal developer features.
- Telemetry and logging changes unless they fix a user-visible privacy or reliability issue.
- Implementation details, class names, frameworks, ticket numbers, PR numbers, and commit hashes.

Merge related commits into one benefit-oriented item. Prefer concrete wording over generic “bug fixes,” but do not expose technical details that are irrelevant to users.

### 6. Write the source changelog

Write English first:

- Use three to five bullets unless the release genuinely needs fewer or more.
- Lead with the most important improvement.
- Keep each bullet to one sentence.
- Use clear, neutral, user-oriented language.
- Name platforms only when the change is platform-specific.
- Mention an OS version only when the feature actually depends on it.
- Avoid marketing exaggeration and unsupported claims.
- Do not add a heading such as “What’s New” unless the user requests one.
- Respect the destination store's limits when repository metadata defines one.

### 7. Translate by meaning

Translate the final English meaning naturally, not word-for-word.

When no languages were requested, use these headings; otherwise include only the explicitly requested languages with their native headings:

```markdown
### English
### Français
### Deutsch
### Español
### Italiano
```

Preserve:

- The same number and order of bullets in every language.
- Product and platform names such as Spotlight, Siri, iOS, Mac, Android, Google Play, and Wear OS.
- Feature scope and qualifications.

Use established platform terminology:

- French: `Raccourcis`
- German: `Kurzbefehle`
- Spanish: `Atajos`
- Italian: `Comandi Rapidi`

Use native punctuation, grammar, and idiomatic store-release phrasing. Do not introduce claims absent from the English source.

## Output

For a single store, return only the paste-ready localized changelog.

For different Apple and Google Play changelogs, use:

```markdown
## Apple App Store

### English
...

## Google Play

### English
...
```

Do not include Git analysis, commit lists, implementation notes, validation details, or a recap unless the user explicitly asks for them.

## Quality Check

Before responding, verify:

- Both tags were included in the comparison.
- The target store was inferred from concrete repository evidence.
- Every bullet is supported by the inspected range.
- Every platform-specific claim is present in that platform's shipped target.
- No technology was incorrectly generalized across Apple and Android.
- Internal-only changes were removed.
- Each language communicates the same user benefit.
- The copy is concise enough for the target store's release notes.
