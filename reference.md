# Omarchy plugin reference

Read this when the workflow in [SKILL.md](SKILL.md) needs the manifest,
runtime, or marketplace submission contract.

Local checks win over this file when they disagree. Read
`/usr/bin/omarchy-plugin-validate` and
`$OMARCHY_PATH/shell/services/PluginRegistry.qml` on the machine you are
developing on. First-party examples are `$OMARCHY_PATH/shell/plugins/`.

## Manifest

Required by `omarchy plugin validate`:

| Field | Rule |
|-------|------|
| `schemaVersion` | JSON number `1`. The string `"1"` is rejected. |
| `id` | Non-empty. Must match `^[A-Za-z0-9][A-Za-z0-9._-]*$`, must not contain `..`, and must not start with `omarchy.`. |
| `name` | Human-readable name. |
| `version` | Current version. Marketplace display allows up to 64 characters. |
| `kinds` | Non-empty array. |
| `entryPoints` | Object. Each value is a non-empty relative path with no `..`, no leading `/`, no newline, and the file must exist. |

Marketplace listings also expect `author`, `description`, and `license`.
`license` in the manifest does not replace a root license file.

Kind to entry-point key, from the validator:

| Kind | Required key |
|------|----------------|
| `bar` | `entryPoints.bar` |
| `bar-widget` | `entryPoints.barWidget` |
| `menu` | `entryPoints.menu` |
| `overlay` | `entryPoints.overlay` |
| `panel` | `entryPoints.panel` |
| `service` | `entryPoints.service` |

An unknown kind is not guessed. A known kind without its key fails validation.

`barWidget`, when present:

- `displayName` — label used by bar placement UI.
- `category` — grouping such as `Time` or `Media`.
- `allowMultiple` — `false` for a normal widget. `true` only when two
  instances on one bar make sense.
- `defaultSection` — `left`, `center`, or `right`. Omitted means `center`.
- `defaults` — inline setting values applied to a new bar entry.
- `schema` — setting fields, each with `key`, `type`, and `label`.

`keepLoaded: true` keeps the instance mounted between summons and across
plugin hot-reload. Use it for a service or panel that must not be destroyed
when another plugin's files change. Its own code updates on shell restart.

`omarchy.clonedFrom` is clone metadata. Keep it on a local clone so the
built-in id still routes to your copy and removal restores the original.
Delete it, and `omarchy.clonePaths` if present, before publishing.

Development manifest for a clock clone:

```json
{
  "schemaVersion": 1,
  "id": "yourname.clock",
  "name": "Custom Clock",
  "version": "1.0.0",
  "author": "Your name",
  "license": "MIT",
  "description": "A small clock with a details panel for the Omarchy bar.",
  "kinds": ["bar-widget"],
  "entryPoints": { "barWidget": "BarWidget.qml" },
  "barWidget": {
    "displayName": "Custom Clock",
    "category": "Time",
    "allowMultiple": false,
    "defaultSection": "center"
  },
  "omarchy": { "clonedFrom": "omarchy.clock" }
}
```

Publish id shape: `io.github.<user>.<plugin-name>`. Plugin ids are permanent
across the marketplace, including retired listings.

## Runtime

Entry points are `Item`s. The host injects `omarchyPath`, `shell`,
`manifest`, and `pluginRegistry` or `barWidgetRegistry`.

Built-ins receive host objects. Third-party plugins receive capability-scoped
facades: own service and lifecycle, detached scalar bar state, and a narrower
configuration surface for a clone of a built-in. A full-bar plugin also
receives detached bar configuration, a widget catalog snapshot, and lifecycle
control for configured non-authentication UI. These facades are not a
same-process sandbox. A visual widget shares the bar scene and can walk
ordinary parent objects. Do not treat the facade as isolation, and do not
expose authentication services through a third-party property.

A replacement bar can draw installed widgets, but those widgets do not get
their companion service there. Service-backed widgets keep that integration
on `omarchy.bar`.

Enablement in `~/.config/omarchy/shell.json`:

- A third-party plugin is enabled when its id is `bar.id`, a `bar.layout`
  entry, or an entry in `plugins[]`.
- A full bar has no off switch. Enabling one sets `bar.id` and replaces the
  active bar. The built-in returns if the selected bar cannot load.
- Bar-widget placement is the bar layout. `omarchy bar move <id> --section
  center` moves one. Sections are `left`, `center`, and `right`.
- Settings sit inline on the layout entry (`{ "id": "...", "format": "HH:mm" }`).
  There is no `config` sub-object.

Install by git only when the user asks to install that repository:

```bash
omarchy plugin add https://github.com/<user>/<repo>.git --enable
```

Add refuses to guess and confirms in a terminal. `--yes` skips those prompts;
use it only when the user explicitly wants a non-interactive install. The
command clones, validates, and enables. It does not run an install hook and
does not use sudo.

`omarchy plugin update <id>` shows a diff and fast-forwards. It refuses dirty
checkouts and rolls back a revision that fails validation. `omarchy plugin
remove <id>` disables first. A git checkout is deleted, a symlink is
unlinked, and a hand-made folder is moved to a timestamped backup.

## QML lifecycle

Panel, overlay, and menu entry points:

```qml
function open(payloadJson) { }
function close() { }
```

`omarchy-shell shell summon <id> '<json>'` calls `open`. `hide` calls
`close`. `toggle` summons when closed and hides when open.

A `bar-widget` that opens a nested panel keeps one kind. `BarWidget.qml`
loads the panel itself and forwards:

- `opened`
- `popoutSwitchClosing`
- `open()`
- `close()`
- `toggle()`
- `closeForPopoutSwitch()`

Use the same `moduleName` as the manifest `id` in the widget and the panel.
The built-in clock at `$OMARCHY_PATH/shell/plugins/panels/clock/` is the
pattern for a bar button plus a keyboard panel. Match its base types and
property names when the shell version differs from a tutorial.

Shell IPC for development goes through `omarchy-shell shell <method>`.
There is no `bar` IPC target. Useful methods: `ping`, `summon`, `hide`,
`toggle`, `call`, `rescanPlugins`, `reloadConfig`, `listPlugins`,
`setPluginEnabled`. `setPluginEnabled` takes a string; only `"true"` enables.

## Troubleshooting

| Symptom | Check |
|---------|--------|
| Plugin folder not found | Use the id printed by `omarchy plugin clone`. The folder is `~/.config/omarchy/plugins/<id>/`. |
| `entry point file not found` | `entryPoints` values must match filename and capitalization. |
| Validates but is not listed | `omarchy-shell shell rescanPlugins`, then `omarchy plugin list --json`. |
| Listed but not visible | Kind, enabled state, and `qs log -p "$OMARCHY_PATH/shell" --tail 100`. |
| Panel opens once | The bar entry point must forward `opened`, `open()`, and `close()`. |

## Marketplace submission

Prepare the repository using the publish steps in [SKILL.md](SKILL.md).
Validation checks structure and Quattro compatibility. It does not review
security. Publication needs a fresh exact-commit scan and an
`approved-and-verified` maintainer decision.

One category, copied exactly:

`Appearance`, `Desktop`, `Developer Tools`, `Hardware`, `Kids`,
`Productivity`, `System`, `Widgets`, `Other`.

One to three tags:

`ai`, `bar`, `education`, `games`, `hyprland`, `kids`, `launcher`, `media`,
`power-management`, `quickshell`, `security`, `system`, `vpn`, `workspaces`.

Create the issue only after the owner has approved the title, body, and all
five checklist statements. Preserve the headings, their order, and the
checklist text.

Title: `[Plugin]: <human-readable name>`

Repository: `omacom/omarchy-plugin-marketplace`

Body:

```markdown
### Repository URL

https://github.com/<user>/<repo>

### Category

Widgets

### Tags

bar

### Suggest a missing tag

_No response_

### Maintainer notes

_No response_

### Submission checklist

- [x] The repository is public and contains installation and removal instructions.
- [x] I have documented the plugin license and any external dependencies.
- [x] I confirm that I own or have permission to submit this plugin and its preview assets.
- [x] The plugin does not overwrite user configuration without explicit consent.
- [x] I understand that approval is for listing and is not a security review.
```

The repository URL is the GitHub root, with no trailing slash and no
`/tree/main`. Keep all six headings. Replace `_No response_` only when the
owner wants a tag suggestion or a note.

```bash
gh issue create \
  --repo omacom/omarchy-plugin-marketplace \
  --title "[Plugin]: Plugin name" \
  --body-file /tmp/omarchy-plugin-submission.md
```

If validation does not comment, edit the same issue. Do not open a duplicate.
The title must start with `[Plugin]: `, the six headings must stay in order,
the category must match exactly, and all five boxes must be checked.

To publish a newer commit of a listed plugin, use the verification form
`verify-plugin.yml` and choose "Verify and publish a newer upstream commit".
The plugin id set must stay the same. Do not resubmit the original issue for
an update.
