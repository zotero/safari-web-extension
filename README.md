# Zotero Safari Web Extension Connector

The Xcode wrapper that packages the Zotero Connector as a [Safari Web
Extension](https://developer.apple.com/documentation/safariservices/safari-web-extensions).
This replaces the old Safari *App* Extension (`safari-app-extension`).

The actual Connector code (shared with the Chrome and Firefox Connectors) lives in
[zotero-connectors](https://github.com/zotero/zotero-connectors); this project is just a thin
container app + `SafariWebExtensionHandler` appex that bundles the Connector's built `safari` web
extension. It was generated once with `xcrun safari-web-extension-converter` and is maintained by hand.

## Building for Development

1. Clone this repository.
1. Clone https://github.com/zotero/zotero-connectors so that it is reachable as `../zotero-connectors`
   from this repo (a `~/zotero-connectors` symlink to your checkout works).
1. Build the Connector:
   ```sh
   cd ../zotero-connectors && ./build.sh      # produces build/safari (among the other flavors)
   ```
   A "Copy Connector build" run-script phase rsyncs `../../zotero-connectors/build/safari` into the
   extension's `Resources` on every build, so the built web-extension files are always picked up. This
   needs script sandboxing off, which the project sets.
1. Open the project in Xcode, select the top-level **Zotero Connector** item, and under each target set a
   valid Team under Signing & Capabilities.
1. Build and run, then enable the extension in Safari ▸ Settings ▸ Extensions. For an unsigned/dev build,
   also enable Safari ▸ Settings ▸ Advanced ▸ "Show features for web developers", then Develop ▸ "Allow
   Unsigned Extensions" (resets each Safari launch).

Because the run-script phase re-copies and Xcode re-signs every build, you don't need to clean between
Connector rebuilds (rebuild + run is enough). The extension files don't appear in
`Copy Bundle Resources` — the script handles bundling. This sidesteps the three ways the declarative
approaches fall short:
- A single **folder reference** nests everything under a `safari/` subfolder, but Safari wants
  `manifest.json` at the `Resources` root.
- **Per-file references** (listing each file) drift out of sync with the Connector build — newly added
  files are silently omitted (this is how the new icon sizes first went missing).
- A **synchronized folder group** (or any flat resource list) flattens the subdirectories, so same-named
  files collide (e.g. `images/unix/…png` vs `images/win/…png`, or gdocs `api.js` vs the top-level one).

## Bundle identifiers

- **This dev wrapper:** app `org.zotero.SafariWebExtensionApp`, extension
  `org.zotero.SafariWebExtensionApp.SafariExtension` — a development identifier so it doesn't collide
  with an installed `Zotero.app` (`org.zotero.zotero`) in Launch Services.
- **Production:** when shipped inside the Zotero desktop app, the extension uses
  `org.zotero.zotero.SafariExtension` (the same id as the old App Extension, so it's an in-place upgrade).

## Regenerating

Day-to-day you should **not** need to regenerate: the rsync build phase is structure-agnostic, so
changes to `zotero-connectors/build/safari` (new files, new subdirectories) are picked up automatically.
Only regenerate if the scaffold itself (the container app, the `SafariWebExtensionHandler`, build
settings) needs resetting:

```sh
xcrun safari-web-extension-converter --project-location <tmp> --app-name "Zotero Connector" \
  --bundle-identifier org.zotero.SafariWebExtensionApp --macos-only --no-open --force \
  ../zotero-connectors/build/safari
```

Then re-apply the local adjustments the converter doesn't make:
1. Set up `build/safari` bundling: a "Copy Connector build" run-script phase that rsyncs
   `../../zotero-connectors/build/safari` into the appex `Resources`, with
   `ENABLE_USER_SCRIPT_SANDBOXING = NO` so it can read the external build dir.
2. Set matching bundle ids (the converter leaves the app target name-derived and the extension
   `…App.Extension`): app `org.zotero.SafariWebExtensionApp`, extension
   `org.zotero.SafariWebExtensionApp.SafariExtension`.

## Architecture

See the main Connector README in `zotero-connectors` for the shared architecture. Unlike the old App
Extension (which had a Swift `GlobalPage`/`SafariExtensionHandler` background and injected scripts via
`SFSafariContentScript`), this is a standard Web Extension: the Connector's MV2 build runs as-is, with
a minimal `SafariWebExtensionHandler` as the only native code.
