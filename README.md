# vencord-customstream-build

Builder repo for a custom Vencord build. It holds no Vencord source; a
GitHub Actions workflow clones everything, builds once, and publishes
the result as a rolling `latest` release.

## What gets built

- Upstream Vencord, `main` branch — https://github.com/Vendicated/Vencord
- customStream (replaces Discord's stream preview with your own image,
  profiles, slideshow, gallery) — https://github.com/MrTopQ/customStream-Vencord
- CustomUpdater (the updater plugin) — https://github.com/MLG-SERBUR/vencord-custom-updater

Build command: `pnpm build --standalone --disable-updater`

- `--standalone` makes a universal build, not tied to the Linux runner,
  so the same files run on Windows Discord desktop and Vesktop.
- `--disable-updater` turns Vencord's stock updater off entirely, so it
  can never replace this custom build with the official one. CustomUpdater
  is the only updater in the build.

## Release format

Rolling release, tag `latest`, in this repo:

- `version.json` — `version` is
  `<vencordShort>-<customStreamShort>-<updaterShort>`, plus the full SHAs
  and the build time. The version only changes when one of the three
  inputs changes, so a daily rebuild with no upstream change is skipped
  by the updater instead of re-downloaded.
- `patcher.js`, `preload.js`, `renderer.js`, `renderer.css` — Discord desktop.
- `vencordDesktopMain.js`, `vencordDesktopPreload.js`,
  `vencordDesktopRenderer.js`, `vencordDesktopRenderer.css` — Vesktop.

Release notes carry the full SHAs plus the upstream Vencord commit list
since the previous build, the release-notes equivalent of the stock
updater's changelog tab.

Dist files are uploaded first, `version.json` last, so CustomUpdater
only sees a new version once every file for that version is in place.

## Triggers

- Daily schedule, 07:00 UTC — picks up upstream and plugin commits.
- Manual: Actions tab > Build custom Vencord > Run workflow.
- Push to the workflow file itself — a pin change rebuilds.

There is no trigger on upstream/plugin pushes: this repo does not own
those repos, so it cannot receive their webhooks. The daily run is the
trade-off for that.

## First install (once, on your PC)

Easiest: run the standalone installer, which switches an existing
regular Vencord install to this build (backup, download, enable both
plugins, launch Discord):

https://github.com/MLG-SERBUR/vencord-custom-installer/releases/latest/download/CustomVencordInstaller.exe

The updater plugin only updates an existing install; it does not do the
first inject. The installer above does that first inject, or do it
manually from a Vencord checkout:

```sh
git clone https://github.com/Vendicated/Vencord
cd Vencord
git clone https://github.com/MrTopQ/customStream-Vencord src/userplugins/customStream
git clone https://github.com/MLG-SERBUR/vencord-custom-updater src/userplugins/customUpdater.desktop
pnpm install
pnpm build --standalone --disable-updater
pnpm inject
```

Restart Discord, enable **CustomStream-Preview** and **CustomUpdater**
in Vencord settings > Plugins. From then on, CustomUpdater checks this
repo's `latest` release, downloads new builds, and (with `autoRestart`
off, the default) tells you to restart Discord to apply them.

Alternative first install without a source checkout: download the eight
dist files plus `version.json` from the `latest` release into your
existing Vencord dist folder and restart Discord. That folder is the one
`pnpm inject` created; if you do not know where it is, use the source
checkout route above instead.

## Changing what it builds

- Pin a plugin or Vencord to a SHA: change the `ref:` in the workflow
  from `main` to that SHA, push the workflow, and the push trigger
  rebuilds. Unpinned `main` means the latest third-party code lands on
  your PC automatically — that is the point of this setup, and also the
  trust trade-off. Pin if you want control instead of zero clicks.
- Add another userplugin: add another checkout step into
  `src/userplugins/<name>` and it is included in the next build. If the
  plugin needs a desktop-only gate, use a `.desktop` folder suffix, the
  way CustomUpdater does.
- Stock Vencord settings and cloud sync are untouched by all of this:
  settings live outside the dist folder, and a rebuild/inject does not
  wipe them. Cloud sync moves settings, not plugin code — another device
  needs this same build to run the same plugins.
