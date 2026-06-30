---
name: infomaniak-uikit
description: Infomaniak house conventions for UIKit iOS apps built with Tuist (flat App/Core/Resources target layout, UI organized by domain under UI/Controller & UI/View, MVVM view models with Combine, AppRouter coordinator, DI via InfomaniakDI/CoreTargetAssembly, Realm via Transactionable + schema migrations, UIConstants paddings). Use when creating, structuring, scaffolding or reviewing UIKit/Tuist code in Infomaniak iOS repos (kDrive…).
---

# Infomaniak — UIKit + Tuist house style

This skill encodes the architecture and code conventions of Infomaniak's **UIKit** iOS apps.
The reference is **ios-kDrive** (UIKit ViewControllers with occasional SwiftUI integration,
Tuist, Realm). For pure-SwiftUI apps (authenticator, SwissTransfer, kMail) use the companion
skill **infomaniak-swiftui-tuist** instead.

> **Priority on divergence: follow kDrive.** When kDrive itself is inconsistent, prefer the
> most recent area (the `AGENTS.md` Context Map is authoritative for placement).

Generate code that blends into the existing module: match the comment density, naming and
idioms of the neighbouring file.

---

## 1. Architecture (flat multi-target, UIKit)

Unlike the SwiftUI apps, kDrive does **not** use a per-feature module (`Feature`) split. It has
a small number of targets, and the **app target holds all the UI**, organized by domain folders:

```
kDrive/                         # App target (product .app) — UIKit, app lifecycle, all UI
├── AppDelegate.swift           #   lifecycle, background tasks, root init
├── AppDelegate+Scene.swift, AppDelegate+BGNSURLSession.swift   # focused extensions
├── SceneDelegate.swift         #   UIWindowSceneDelegate
├── AppRouter.swift             #   navigation & deep-linking coordinator (see §4)
├── AppRouter+PublicShare.swift, AppRouter+SharedWithMe.swift…  # routing extensions
├── UI/
│   ├── Controller/<Domain>/    #   UIViewControllers grouped by domain (Files/, Menu/, Home/, Photo/, SwiftUI/)
│   └── View/<Domain>/          #   custom UIViews, cells, XIBs grouped by domain
├── Resources/                  #   Assets.xcassets, *.lproj, Info.plist, entitlements, storyboards
├── Utils/                      #   helpers
├── Data/                       #   app-level data handling
└── IAP/                        #   in-app purchase

kDriveCore/                     # Business-logic framework — NO app UI (shared UI components allowed under UI/)
├── Data/
│   ├── Api/                    #   DriveApiFetcher (+Upload/+Listing/+Share…), Endpoint.swift (+Files/+Share…)
│   ├── Models/                 #   Realm-compatible models (File.swift, Drive/, Upload/…)
│   ├── Cache/                  #   *Manager (AccountManager, DriveFileManager/, DriveInfosManager/…)
│   ├── Database/               #   RealmAccessor.swift, Realm configuration
│   ├── DownloadQueue/, Upload/ #   background session managers
│   └── MQService/              #   MQTT real-time sync
├── Services/                   #   AppContextService, BackgroundTasksService…
├── UI/                         #   shared UI (Alert/, Scan/, UIConstants.swift)
├── Utils/                      #   PHAsset/, FileProvider/, Deeplinks/, Sentry/…
└── DI/                         #   CoreTargetAssembly.swift (Factory registration)

kDriveResources/                # Resources framework (localizations, assets) → KDriveResourcesAsset / *Strings
kDriveFileProvider/             # NSFileProviderExtension (Enumerators/…)
kDriveShareExtension/           # share sheet extension
kDriveActionExtension/          # action sheet extension
kDriveTests/, kDriveAPITests/, kDriveUITests/, kDriveTestShared/

Tuist/
├── Package.swift               # SPM declarations
└── ProjectDescriptionHelpers/
    ├── Constants.swift              #   versions, settings, deployment target, scripts
    ├── ExtensionTarget.swift        #   `Target.extensionTarget(...)` factory for app extensions
    └── SettingsDictionary+Extension.swift  # marketingVersion / bridgingHeader / compilationConditions / appIcon

Project.swift                   # declares all targets (app, Core, Resources, extensions, tests)
.swift-version, .mise.toml, .swiftformat, .swiftlint.yml, file-header-template.txt
```

**Placement rules:**
- A screen → `kDrive/UI/Controller/<Domain>/`. A custom view/cell/XIB → `kDrive/UI/View/<Domain>/`.
- A view model → next to its controller (`<Name>ViewModel.swift` beside `<Name>ViewController.swift`).
- Business logic, a manager, a service, the API layer, models → `kDriveCore/` (no app UI).
- UI shared with extensions / cross-cutting (alerts, `UIConstants`) → `kDriveCore/UI/`.
- Long files split into focused extensions: `Type+Aspect.swift` (e.g. `AppRouter+PublicShare`,
  `DriveApiFetcher+Upload`, `FileListViewController+DragAndDrop`).

---

## 2. Tuist + mise setup (initializing a cloned repo)

Always in this order:

```sh
mise install      # installs the right tuist version (+ swiftlint, swiftformat…)
tuist install     # resolves the project's SPM dependencies
tuist generate    # generates the .xcodeproj / .xcworkspace
```

> **Never** commit the generated `.xcodeproj`: it is regenerated by `tuist generate`.
> After any dependency or target-structure change, re-run `tuist generate`.
> Lint before every PR: `scripts/lint.sh`. Never run anything under `ci_scripts/` locally.

---

## 3. Declaring targets in Tuist

Targets are declared explicitly in `Project.swift` (no `Feature` helper). The app target lists
`sources: "kDrive/**"` plus its resources (storyboards, xcassets, strings, xibs) and depends on
the framework targets + external SPM packages.

```swift
// Project.swift — adapted from kDrive
.target(name: "kDrive",
        destinations: Constants.destinations,
        product: .app,
        bundleId: "com.infomaniak.drive",
        deploymentTargets: Constants.deploymentTarget,
        infoPlist: .file(path: "kDrive/Resources/Info.plist"),
        sources: "kDrive/**",
        resources: [
            "kDrive/**/*.storyboard", "kDrive/**/*.xib", "kDrive/**/*.xcassets",
            "kDrive/**/*.strings", "kDrive/**/*.stringsdict", "kDrive/**/PrivacyInfo.xcprivacy"
        ],
        entitlements: "kDrive/Resources/kDrive.entitlements",
        scripts: [Constants.swiftlintScript],
        dependencies: [
            .target(name: "kDriveCore"),
            .target(name: "kDriveResources"),
            .target(name: "kDriveFileProvider"),
            .external(name: "RealmSwift"), .external(name: "InfomaniakDI"),
            .external(name: "FloatingPanel"), .external(name: "DifferenceKit") // …
        ],
        settings: .settings(base: Constants.baseSettings))
```

- App **extensions** (FileProvider, Share, Action) are built with the
  `Target.extensionTarget(...)` factory in `ProjectDescriptionHelpers`, which re-shares the
  relevant app controllers/XIBs and applies `ISEXTENSION` compilation conditions.
- Always reuse `Constants.baseSettings`, `Constants.destinations`,
  `Constants.deploymentTarget` and the `SettingsDictionary+` helpers
  (`.compilationConditions(...)`, `.bridgingHeader(...)`, `.appIcon(...)`) rather than literals.
- Code shared with an extension is guarded with `#if !ISEXTENSION` / `#if ISEXTENSION`.
- Every source file carries the GPL header (`file-header-template.txt`).

---

## 4. Navigation: the `AppRouter` coordinator

Navigation and deep linking go through a single coordinator, `AppRouter`, a `struct` conforming
to an `AppNavigable` protocol, with its dependencies injected and split into focused extensions
(`AppRouter+PublicShare`, `AppRouter+SharedWithMe`, …). ViewControllers navigate via the
injected router, **not** by reaching into `UIApplication`/window themselves.

```swift
public struct AppRouter: AppNavigable {
    @LazyInjectService private var driveInfosManager: DriveInfosManager
    @LazyInjectService var accountManager: AccountManageable
    @LazyInjectService var deeplinkService: DeeplinkServiceable

    @MainActor var window: UIWindow? { /* resolve from the active SceneDelegate */ }
    // navigation methods (push/present/showX) live here and in AppRouter+*.swift
}

// In a controller:
@LazyInjectService var router: AppNavigable
```

---

## 5. Dependency injection (`InfomaniakDI`)

Registration happens in `kDriveCore/DI/CoreTargetAssembly.swift`, an `open class` subclassing
`TargetAssembly`. Factories are grouped by category (`networkingServices`, …) and merged.

```swift
open class CoreTargetAssembly: TargetAssembly {
    private static var networkingServices: [Factory] {
        [
            Factory(type: AccountManageable.self) { _, _ in AccountManager() },
            Factory(type: BackgroundUploadSessionManager.self) { _, _ in BackgroundUploadSessionManager() },
            // protocol → concrete resolution:
            Factory(type: PhotoLibraryQueryable.self) { _, resolver in
                try resolver.resolve(type: PhotoLibraryUploader.self,
                                     forCustomTypeIdentifier: nil, factoryParameters: nil, resolver: resolver)
            }
        ]
    }
}
```

### Accessing services
- `@LazyInjectService` for a controller/router/manager property.
- `@InjectService` **locally** inside a function when the call is one-off.
- Declare an injected service **as close to its use as possible** — if a single
  controller/function needs it, don't hoist it into a shared base type.

---

## 6. UIKit screens: ViewController + ViewModel (MVVM)

A screen is a `UIViewController` (often `UICollectionViewController`) paired with a `ViewModel`.
The view model owns state and business calls; the controller owns UIKit and binds to the view
model (Combine + `DifferenceKit` for diffable updates).

```swift
@MainActor
class FileListViewModel: SelectDelegate {
    typealias FileListUpdatedCallback = ([Int], [Int], [Int], [(source: Int, target: Int)], Bool, Bool) -> Void

    struct Configuration {            // per-screen config as a nested struct with defaults
        var isMultipleSelectionEnabled = true
        var rootTitle: String?
        // …
    }
    // exposes published state + callbacks; calls managers / DriveFileManager
}

class FileListViewController: UICollectionViewController, SceneStateRestorable {
    @LazyInjectService var router: AppNavigable
    @LazyInjectService var matomo: MatomoUtils

    // MARK: - Properties
    let viewModel: FileListViewModel
    private var bindStore = Set<AnyCancellable>()
    var driveFileManager: DriveFileManager { viewModel.driveFileManager }

    // MARK: - View lifecycle
    // viewDidLoad: register cells, set up collection view, bind view model
}
```

Conventions:
- **`@MainActor`** on view models and any UI-bound type.
- Group members with `// MARK: -` (Properties / View lifecycle / Actions …); swiftformat enforces marks.
- Keep controllers thin: push logic into the view model or a `*Manager` in `kDriveCore`.
- Bindings stored in a `Set<AnyCancellable>` (`bindStore`); cancel/replace long tasks via stored `Task`/`ObservationToken`.
- Register cells/headers and dequeue with typed helpers; XIB-backed cells live in `kDrive/UI/View/<Domain>/`.

---

## 7. Persistence: Realm via `Transactionable`

kDrive uses RealmSwift through the in-house **`Transactionable`** abstraction
(`InfomaniakCoreDB`). **Never** open a `Realm` by hand in feature code: go through the
`database` / `uploadsDatabase` transactionable exposed by a manager (`DriveFileManager`, upload
operations…). `RealmAccessor` (in `kDriveCore/Data/Database/`) centralizes opening/retry.

```swift
// Read — fetchResults(ofType:) with a filter on the lazy collection; returns frozen results
let files = database.fetchResults(ofType: File.self) { lazyCollection in
    lazyCollection.filter("rawType = %@", "dir")
}

// Single object
let frozenFile = database.fetchObject(ofType: File.self, forPrimaryKey: fileId)

// Write — writeTransaction { writableRealm in ... }
try database.writeTransaction { writableRealm in
    let obsolete = writableRealm.objects(File.self).filter("isObsolete == true")
    writableRealm.delete(obsolete)
    writableRealm.add(parsedFiles, update: .modified)
}
```

**Realm rules (kDrive-specific):**
- Every model defines a proper **primary key** for thread safety; pass **frozen** objects across
  threads/actors, re-fetch a live object inside the write block before mutating.
- **Schema migrations:** any schema-affecting change to a Realm `Object`/`EmbeddedObject` must
  bump the matching `RealmSchemaVersion` (`drive` or `upload`) in
  `kDriveCore/Data/Cache/DriveFileManager/DriveFileManagerConstants.swift`, and update the
  migration block when existing data needs migrating.
- Identify a transactionable instance with the `kDriveDBID` constants (`uploads`, `driveInfo`).

---

## 8. API layer

- Extend `DriveApiFetcher` for API operations, in focused extensions:
  `DriveApiFetcher+Upload.swift`, `+Listing.swift`, `+Share.swift`.
- Define endpoints in `Endpoint.swift` and extend per domain: `Endpoint+Files.swift`,
  `Endpoint+Share.swift`, …
- Networking is Alamofire-based; use `async/await` and structured concurrency.
- Managers follow the `*Manager` naming (`AccountManager`, `DriveFileManager`,
  `DriveInfosManager`, `AvailableOfflineManager`).

---

## 9. Conventions (naming, paddings, resources)

### 9.1 Naming
- Types in **PascalCase**; variables/functions in **camelCase**.
- Identifiers: `newId`, **never** `newID` (same for `userId`, `fileId`). The `Id` stays
  camelCase; the acronym is not uppercased (swiftformat does not enable the acronyms rule).
- Max line width **130**, 4-space indentation, LF endings.
- Imports alphabetically grouped, blank line after imports (swiftformat).

### 9.2 Paddings & metrics
Use the **`UIConstants`** tokens (in `kDriveCore/UI/UIConstants.swift`) rather than magic
numbers — e.g. `UIConstants.List.paddingBottom`, `UIConstants.Padding.…`:

```swift
collectionView.contentInset = UIEdgeInsets(top: 0, left: 0, bottom: UIConstants.List.paddingBottom, right: 0)
```
Never hardcode raw paddings/insets. (`IKPadding` exists for the SwiftUI-integration corners but
`UIConstants` is the UIKit default here.)

### 9.3 Resources
Strings, colors and images always via the generated `kDriveResources` accessors
(`KDriveResourcesStrings.Localizable.…`, `KDriveResourcesAsset.…`) — never a raw string/color
literal. Localize every user-facing string.

### 9.4 SwiftUI integration
When a SwiftUI view is embedded (`UI/Controller/SwiftUI/`), property wrappers must be `private`
(`@State`, `@StateObject`, `@ObservedObject`, …), same as the SwiftUI skill.

---

## 10. Concurrency (Swift, strict concurrency)

- Use **`async/await`** and structured concurrency for API/IO.
- `@MainActor` on view models and UI-bound objects; keep `kDriveCore` business types off the
  main actor and `Sendable` where simple.
- Pass **frozen** Realm objects across concurrency boundaries; never share live Realm objects.
- For non-strict libs, `@preconcurrency import …` is acceptable.

---

## 11. Pre-delivery checklist

- [ ] File in the right place (`kDrive/UI/Controller|View/<Domain>` for UI, `kDriveCore/…` for logic).
- [ ] GPL header present (template `file-header-template.txt`).
- [ ] Imports alphabetically grouped, blank line after imports.
- [ ] Controller stays thin; logic in the view model or a `*Manager`.
- [ ] PascalCase/camelCase naming, identifiers `…Id` (not `…ID`).
- [ ] Paddings/insets via `UIConstants`, no magic numbers.
- [ ] `@LazyInjectService` / `@InjectService` as close to its use as possible; services registered in `CoreTargetAssembly`.
- [ ] Navigation via `AppRouter` / `AppNavigable`, not direct window/`UIApplication` access.
- [ ] Realm only via `Transactionable` (`database`/`uploadsDatabase`: `fetchResults`/`fetchObject`/`writeTransaction`); frozen objects across threads.
- [ ] Realm schema change → matching `RealmSchemaVersion` bumped + migration block updated.
- [ ] API work extends `DriveApiFetcher` / `Endpoint` in focused `+Aspect` extensions.
- [ ] Strings/colors/images via `kDriveResources`; all user-facing strings localized.
- [ ] `@MainActor` on UI/view-model types; `async/await` for IO.
- [ ] `scripts/lint.sh` clean; ran `tuist generate` after target/dependency changes.

---

## Installing this skill

This skill lives in `infomaniak-uikit/`. To make it discoverable by opencode,
place it (or symlink it) under a skills directory, with the folder name matching the `name`
field in the frontmatter:
- personal: `~/.config/opencode/skills/infomaniak-uikit/SKILL.md`
- project: `<repo>/.opencode/skills/infomaniak-uikit/SKILL.md`
