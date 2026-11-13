# Firetower as a Flatpak

`com.usefiretower.Firetower.yml` builds the desktop app from source, offline,
on the GNOME 51 runtime — the way Flathub builds it. It is not on Flathub
yet; this directory is the manifest that submission will start from.

## Build and run it

```sh
flatpak install --user flathub org.gnome.Sdk//51 \
  org.freedesktop.Sdk.Extension.rust-stable//26.08 \
  org.freedesktop.Sdk.Extension.node22//26.08

cd desktop/flatpak
flatpak-builder --user --install --force-clean build-dir com.usefiretower.Firetower.yml
flatpak run com.usefiretower.Firetower
```

`build-dir/`, `repo/` and `.flatpak-builder/` are build output and ignored.

## The two generated files

Flathub builds without network, so every dependency is a source listed in
advance. `cargo-sources.json` is `src-tauri/Cargo.lock` and
`node-sources.json` is `pnpm-lock.yaml`, each turned into a list by
[flatpak-builder-tools](https://github.com/flatpak/flatpak-builder-tools).
**When either lockfile changes, regenerate its list**, or the build fails
looking for a crate or a tarball it was never given:

```sh
git clone https://github.com/flatpak/flatpak-builder-tools /tmp/fbt
uv run --script /tmp/fbt/cargo/flatpak-cargo-generator.py \
  desktop/src-tauri/Cargo.lock -o desktop/flatpak/cargo-sources.json
uvx --from /tmp/fbt/node flatpak-node-generator pnpm --pnpm-store-version v11 \
  desktop/pnpm-lock.yaml -o desktop/flatpak/node-sources.json
```

## What is different inside the sandbox

* **No self-update.** Flathub updates the app; `/app` is read-only to it in
  any case. The shell does not load the updater when `/.flatpak-info` exists,
  and `updates_itself` tells the renderer so, which keeps the rail quiet and
  has the Updates screen say the system does it (`src/update.ts`).
* **The binary is named after the app id.** GTK takes the window's Wayland
  app_id from the program name, and the desktop matches a window to its
  `.desktop` file and icon by it. Tauri's `enableGTKAppId` does not help:
  it names the GApplication, which Tao's windows are not attached to.
* **Permissions.** Network, the display, the GPU, the Secret Service (for the
  tokens) and the notification daemon. Links and the save dialog go through
  portals and need nothing.

## Before Flathub

* The app id assumes `usefiretower.com`: Flathub verifies it by fetching a
  token from `https://usefiretower.com/.well-known/org.flathub.VerifiedApps.txt`.
* The `dir` source becomes a `git` source pinned to a `desktop-v*` tag and its
  commit, and the manifest moves to its own `flathub/com.usefiretower.Firetower`
  repository.
* The metainfo needs screenshots and a release entry per version.
