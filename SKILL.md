---
name: omarchy-plugin-development
description: >
  Develop, validate, and publish Omarchy Quattro shell plugins. Use when
  creating or editing an Omarchy plugin, manifest.json, or plugin QML
  (bar-widget, panel, overlay, menu, service, or full bar), when running
  `omarchy plugin clone`, `omarchy plugin validate`, or qmllint against the
  shell, or when listing a plugin on plugins.omarchy.org. Do not use for
  end-user bar layout, shell.json, themes, Hyprland, or other desktop config.
---

# Omarchy plugin development

Build a third-party plugin for the Omarchy Quattro shell. Plugins are QML
hosted by the one long-running `omarchy-shell` process.

The installed shell is the contract. Before writing a component, read the
closest built-in under `$OMARCHY_PATH/shell/plugins/` and the checks in
`omarchy plugin validate --help`. Marketplace pages describe the workflow;
they do not replace the local validator.

Details that do not fit the workflow live in [reference.md](reference.md).

## Boundaries

- Edit a user-owned plugin: `~/.config/omarchy/plugins/<id>/` while developing,
  or the plugin's own git repository. Never edit `$OMARCHY_PATH`
  (`/usr/share/omarchy`). Package updates overwrite it.
- Do not start a second Quickshell process. `omarchy-shell` only forwards IPC
  to the running shell.
- A plugin runs unsandboxed, with the user's permissions. Review every
  dependency and command. Document external dependencies, setup steps,
  privilege boundaries, services, installers, and remote builds in the README.
- Do not add download-to-shell execution, unpinned external git execution,
  passwordless sudoers rules, or privileged process control that reads
  predictable shared temporary paths. Marketplace listing is not a security review.
- This skill is for authoring a plugin. End-user bar layout, `shell.json`,
  themes, and Hyprland config belong to the Omarchy desktop skill.

## Choose a kind

Match a built-in with the same kind and interaction. A manifest may declare
more than one kind. `omarchy.media` is `service` plus `bar-widget`.

| Kind | `entryPoints` key | Use it for |
|------|-------------------|------------|
| `bar-widget` | `barWidget` | Item in the active bar |
| `panel` | `panel` | Floating surface |
| `overlay` | `overlay` | Fullscreen surface |
| `menu` | `menu` | Summoned menu |
| `service` | `service` | Headless singleton |
| `bar` | `bar` | Full bar that replaces `omarchy.bar` |

A panel, calendar, or detail view loaded by `BarWidget.qml` stays inside the
`bar-widget`. Do not add a second kind for that nested file.

## Develop

1. Clone the closest built-in when one exists. The clone copies the plugin
   into `~/.config/omarchy/plugins/<username>.<source>/`, enables it, and
   replaces the built-in. Calls to the original id route to the clone while
   `omarchy.clonedFrom` is set.

   ```bash
   omarchy plugin clone omarchy.clock --edit
   ```

   Keep the printed id and the generated `omarchy.clonedFrom` until the plugin
   is ready to publish. Removing the clone restores the built-in.

2. For a plugin with no built-in equivalent, create
   `~/.config/omarchy/plugins/<id>/manifest.json`. Use a development id such
   as `<username>.<name>`. Switch to the permanent id only in the publish step.

3. Write the manifest with `schemaVersion` as the number `1`, and with `id`,
   `name`, `version`, `kinds`, and `entryPoints`. Every declared kind needs
   its entry-point key from the table above. Entry-point values are safe
   relative paths to files that exist. See [reference.md](reference.md) for
   the `barWidget` block and id rules.

4. Implement entry points as QML `Item`s, not `ShellRoot`. Set `moduleName`
   to the manifest `id` everywhere that property exists. Read the cloned or
   built-in QML and follow its base types (`BarWidget`, `Panel`,
   `KeyboardPanel`, and so on) instead of inventing a parallel API.

   Panel, overlay, and menu entry points expose `open(payloadJson)` and
   `close()` so `summon` and `hide` work. A bar widget that owns a nested
   panel forwards `opened`, `popoutSwitchClosing`, `open()`, `close()`,
   `toggle()`, and `closeForPopoutSwitch()` to that panel.

5. Import the palette as `qs.Commons as Commons` and use `Commons.Color`.
   Qt 6.12 has its own `Color` type that shadows an unqualified `Color`.
   Keep a plain `import qs.Commons` when the file also uses `Style`, `Util`,
   or `Border`. Prefer `Style.space()` and `Style.font.*` over raw pixels.

6. Prefer a change signal over a timer: a Quickshell service, an existing
   event stream, a `FileView` watch, or an IPC refresh after the command that
   changes the state. A widget is built once per monitor, so a polling
   `Process` runs on every monitor for the life of the session. If polling is
   the only option, keep one shared timer and the longest tolerable interval.

Saving a file under `~/.config/omarchy/plugins/` reloads that plugin.
`keepLoaded: true` keeps a service or panel mounted across that reload; those
code changes apply on `omarchy restart shell`. Force a rescan only when the
new plugin is not discovered:

```bash
omarchy-shell shell rescanPlugins
```

## Validate and exercise

```bash
PLUGIN_DIR="$HOME/.config/omarchy/plugins/<id>"
omarchy plugin validate "$PLUGIN_DIR"
qmllint -I "$OMARCHY_PATH/shell" "$PLUGIN_DIR"/*.qml
```

Both commands must exit 0. `omarchy plugin validate` checks JSON,
`schemaVersion`, required fields, id shape, the reserved `omarchy.*`
namespace, kind-to-entry-point keys, relative paths that exist,
`barWidget.defaultSection`, and symlinks. Symlinks are rejected everywhere
except `.git`.

Then confirm discovery and the summon route:

```bash
omarchy plugin list --json | jq --arg id "<id>" '.[] | select(.id == $id)'
omarchy-shell shell summon "<id>" '{}'
omarchy-shell shell hide "<id>"
```

Before sharing, test click, Escape, summon, hide, disable, re-enable,
`omarchy restart shell`, and removal. If QML fails to load:

```bash
qs log -p "$OMARCHY_PATH/shell" --tail 100
```

Common failures are in [reference.md](reference.md).

## Publish

Do this only when the user wants the plugin listed.

1. Put the plugin in a public GitHub repository, one plugin, `manifest.json`
   at the repository root.
2. Replace the clone id with a permanent namespaced id, preferably
   `io.github.<user>.<plugin-name>`. Search the marketplace first. Retired
   ids stay reserved. Update `moduleName`, commands, and docs to the new id.
3. Remove `omarchy.clonedFrom`. Do not publish the tutorial's id, author,
   URL, or description unchanged.
4. Add a root README with install and removal commands, a root license, and
   notes for every external dependency. An optional root preview may be
   `preview.png`, `preview.jpg`, `preview.jpeg`, `preview.webp`, or
   `preview.avif`.
5. Run `omarchy plugin validate` on the repository root.

Submitting to [plugins.omarchy.org](https://plugins.omarchy.org/publish.html)
creates an issue on `omacom/omarchy-plugin-marketplace`. Follow
[reference.md](reference.md) for the exact title, headings, category, and
tags. Show the title and body to the owner and create the issue only after
they explicitly approve every checklist statement.

Sources: [Develop a Plugin](https://plugins.omarchy.org/develop.html),
[Publish a Plugin](https://plugins.omarchy.org/publish.html),
[submission guide](https://github.com/omacom/omarchy-plugin-marketplace/blob/main/SUBMISSION.md).
