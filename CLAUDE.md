# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Nextpad++ is a native macOS port of Notepad++ written in Objective-C++ (`.mm`) against Cocoa. It isn't a Wine wrapper or a rewrite. It keeps Notepad++'s data formats (langs/stylers/theme XML, nativeLang localization XML, shortcuts.xml, functionList parsers, UDL files) and its plugin message numbering, so behaviour should match Windows Notepad++ unless macOS conventions call for something else. Bug fixes often cite GitHub issue numbers (`#266`) in commit messages and in code comments.

## Build & run

CMake builds a Universal (arm64 + x86_64) app with a minimum of macOS 11. The build directories `build/`, `build-release/` and `build-release-tahoe/` are gitignored.

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
open "build/Nextpad++.app"            # or run build/Nextpad++.app/Contents/MacOS/Nextpad++ for stdout logs
build/Nextpad++.app/Contents/MacOS/Nextpad++ -nosession -noPlugin file.txt   # CLI flags: see src/main.mm printHelp
```

- Prerequisites are the Xcode Command Line Tools (full Xcode isn't needed) and `brew install cmake`. A clean parallel build takes about 20 s, and the only warnings are deprecations.
- A dev build shares `~/Library/Application Support/Nextpad++/` and its preferences with any installed `/Applications/Nextpad++.app`. Use `-nosession` so the dev build doesn't touch the real session.
- Builds are ad-hoc signed by default, so macOS privacy (TCC) grants are tied to the binary's hash and the "access Downloads/Documents" prompts return after every rebuild. Two copies with different hashes running at once also revoke each other's grant on every "Allow". To avoid this, configure with a local signing certificate: `-DNPP_CODESIGN_IDENTITY="Nextpad Dev"`, where "Nextpad Dev" is a self-signed code-signing certificate in the login keychain.
- New source files have to be added by hand to `APP_SRCS`/`APP_HEADERS` in `CMakeLists.txt`, because there is no glob for `src/`. All app sources compile with `-fobjc-arc`.
- Post-build steps copy the `resources/` data (themes, localization, functionList, UDLs, the default shortcuts/contextMenu/toolbar XML) into the bundle and then ad-hoc codesign it. When you edit a resource file, rebuild. Don't hand-edit the bundle.
- The version lives in `CMakeLists.txt` (`MACOSX_BUNDLE_*_VERSION`). Release signing, notarization and DMG tooling live in `tools/` and `signing-config.sh`, which are local only and never committed.

## Tests

The project has no unit-test target for the app. The standalone harnesses are:

```bash
bash regex/test/run.sh         # NppRegexSearch (default std::regex backend) against real Scintilla Document
bash regex/test/run_boost.sh   # Boost.Regex backend
cmake -S test_plugins -B test_plugins/build && cmake --build test_plugins/build
./test_plugins/build/test_plugins [plugins_dir]   # dlopen/dlsym smoke test of installed plugins
```

Everything else is verified by running the app.

## Layout

- `src/` holds all the app code (flat, one class per `.h`/`.mm` pair). `src/LexUser.cxx` is a macOS-compatible copy of Lexilla's UDL lexer. CMake swaps it in for `lexilla/lexers/LexUser.cxx`.
- `scintilla/` and `lexilla/` are vendored from the Notepad++ upstream with RTL patches applied in place. Only `scintilla/src`, `scintilla/cocoa` and the Lexilla sources get built. Avoid editing these unless the fix really belongs in the editor component.
- `regex/` is the `SCI_OWNREGEX` implementation (the counterpart of Windows' `boostregex/`). Both backends get compiled in: `NppRegexSearch.cxx` (per-line `std::regex`, the default) and `BoostRegExSearch.cxx` (whole-buffer, header-only Boost vendored under `regex/boost/`). `RegexBackendSelect.cxx` chooses one at runtime from `gNppUseBoostRegex`, which is set by the Preferences "Use Boost Regex" toggle.
- `resources/` contains the Notepad++-format data files, `Info.plist`, entitlements and icons. `en.lproj/InfoPlist.strings` is the template that CMake stamps into roughly 135 `<lang>.lproj` dirs so that macOS registers every UI language.
- `downloads/` holds release DMGs (gitignored).

## Architecture

- **Startup:** `main.mm` parses Notepad++-style CLI flags (`NppCommandLineParams`), then `NppApplication` (an NSApplication subclass that intercepts menu actions for macro recording), then `AppDelegate`. The menu bar is built in code by `MenuBuilder` (there are no nibs), and `NppLocalizer` then translates it.
- **Windows and editors:** `MainWindowController` (about 11k lines, the hub that corresponds to Windows' `Notepad_plus`) owns one or two `TabManager`s (main and secondary view for split view). Each `TabManager` owns an `NppTabBar` plus a set of `EditorView`s. `EditorView` (about 7k lines) wraps one Scintilla `ScintillaView` per buffer and handles file I/O, encoding/EOL/BOM, auto-backup, per-tab state, macros and Scintilla notifications. Components talk to each other through `NSNotification`s (`EditorView*Notification`, `NPPLocalizationChanged`) and delegates.
- **Panels:** docking side panels (Document Map, Function List, Project, Folder as Workspace, Git, Clipboard History, Character panel, and plugin panels) are hosted by `SidePanelHost`/`PanelFrame` and can float in a `FloatingPanelWindow`. Find, Replace, Find in Files, Mark and Replace in Files all go through `FindWindow` + `SearchEngine` → `SearchResultsPanel`. `FindInFilesPanel.mm` is dead code and is excluded from the build.
- **Plugins:** `NppPluginInterfaceMac.h` is the public plugin ABI. It mirrors Windows `PluginInterface.h` and `Notepad_plus_msgs.h` with the same `NPPM_*`/`NPPN_*` integer values, but strings are UTF-8 `char*`, handles are opaque `uintptr_t`, and `NppData._sendMessage` stands in for Win32 `SendMessage`. `NppPluginManager` dlopens the `.dylib`s from the user plugins dir and dispatches `NPPM_*` in a large switch. Scintilla handles route `SCI_*` directly. Changes to that header are ABI changes for already-ported plugins. `PluginsAdminWindowController` installs plugins from the nppPluginList registry.
- **Languages & styling:** `NppLangsManager`, `NppBuiltinLanguages` and `NppThemeManager` read `langs.model.xml`, `stylers.model.xml` and `themes/*.xml`. `UserDefineLangManager`, `UserDefineDialog` and `UDLStylerDialog` handle UDLs, which are lexed by `src/LexUser.cxx`.
- **Settings and user data:** preferences live in `NSUserDefaults` (keys are `kPref*` constants, mostly in `PreferencesWindowController`). File-based config (shortcuts.xml, contextMenu.xml, plugins, backups, sessions, localization overrides, UDLs) lives under `~/Library/Application Support/Nextpad++/`. Always resolve those paths through `NppConfigDir()`/`NppConfigSubpath()` from `NppPaths.h`, which also migrates the legacy `~/.nextpad++`. On first run, the defaults are copied there from the bundle.
- **Shortcuts:** `ShortcutMapperWindowController` edits `shortcuts.xml`, which covers menu commands, macros, run commands, plugin commands and Scintilla key overrides.

## Fork decisions (this repo is a personal fork that no longer tracks upstream releases)

- The GitHub update check and the "Check for Updates…" menu item were removed. Don't reintroduce them.
- Option+Shift+Left/Right select by word, as they do elsewhere on macOS. The Windows column extend they used to do is now on Ctrl+Option+Shift+Left/Right. Option+Shift+Up/Down still extend a column block. The binding lives in `macMapDefault` (`scintilla/cocoa/ScintillaCocoa.mm`), and `kSciDefaultKeys` in `EditorView.mm` has to be kept in sync with it.
- Clickable links (`kPrefClickableLinkEnable`) default to off.

## Conventions and gotchas

- **Never match UI text in English.** The UI is translated at runtime, so look up top-level menus by tag (`kMenuTagMacro`, `kMenuTagPlugins`, and so on in `MenuBuilder.h`) rather than by title. User-visible strings go through `[NppLocalizer.shared translate:]`, and the code has to re-translate when it observes `NPPLocalizationChanged`. Several past bugs came from alert buttons or shortcuts breaking once their buttons were translated.
- **Byte offsets are UTF-8.** Scintilla positions are UTF-8 byte offsets, not `NSString` UTF-16 indices. Many past crash fixes involved text handling on non-ASCII documents, so convert between the two explicitly.
- **Protect auto-backups.** An unsaved buffer's auto-backup is the only copy of its text. Claim backup filenames through `+[EditorView uniqueBackupPathInDirectory:filename:]` and never delete the last good backup when a write fails.
- **Rebuild after editing icons.** Icons are synced into the bundle with `rsync -a` on every build, because `cmake -E copy_directory` doesn't overwrite stale files.
