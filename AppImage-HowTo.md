# How to Add and Update an AppImage

A distributor-agnostic guide for integrating any AppImage into a Linux
desktop with stable naming, FreeDesktop icons, `.desktop` launcher entries,
and version updates. Written for AI agents to produce terminal commands for
the user to run.

The examples assume a Wayland compositor with a launcher that reads standard
FreeDesktop `.desktop` files (e.g. Niri + DankMaterialShell). Adjust the paths
to your own setup.

---

## Triggers → sections

WHEN this document is loaded → READ "Agent operating mode" first, then only
the section(s) your trigger matches below.

- WHEN the owner provides app-info URLs → READ "Sourcing the app's details".
- WHEN a newly downloaded AppImage has nothing set up → READ §1, then §2, then
  "Combined".
- WHEN an AppImage only needs to be made executable → READ §1.
- WHEN an AppImage only needs its stable name → READ §2.
- WHEN only the icon is needed, for one or more named AppImages → READ §3.
- WHEN only the `.desktop` entry is needed, for one or more named AppImages →
  READ §4.
- WHEN both the icon and the `.desktop` entry are needed, for one or more named
  AppImages → READ "Combined".
- WHEN updating one or more already-integrated apps → READ §5.
- WHEN the task matches more than one trigger → READ each matched section in
  the order listed above, then render one command block that covers everything.

---

## Agent operating mode

This guide is written for collaboration where the **owner runs the terminal
commands** and the **Agent produces copy-paste output**.

- WHEN rendering output → emit one copy-paste command block for the whole
  task. Never run a command yourself, and never emit more than one block.
- WHEN a command acts on AppImages → use the exact AppImage filenames the
  owner gave. Never use `*.AppImage` or any wildcard over `~/Applications`.
  Never list, scan or enumerate that folder.
- WHEN app details (Name, Comment, Categories) are needed → take them from the
  owner's URL (see [Sourcing the app's details](#sourcing-the-apps-details)).
  Never open the source repository, build files or YAML/config.
- WHEN a file's presence is uncertain → let the command check it. Never ask
  the owner to verify it.
- WHEN adding a best-practice step → guard it with `command -v`, make it safe
  to repeat, and add a short `# comment` saying what it does. Never add an
  unguarded or unexplained step.

---

## Placeholders

Throughout this guide, replace these tokens with the real values for each app:

| Placeholder | Meaning | Example |
|---|---|---|
| `<file>.AppImage` | The exact AppImage filename the owner gave | `someapp.AppImage` |
| `<app>` | Stable lower-case name for the `.desktop`/icon files (strip `.AppImage`) | `someapp` |
| `<AppImage>` | The downloaded AppImage filename including its version | `someapp-1.2.0-linux-x86_64.AppImage` |
| `<AppName>` | Human-readable display name shown in the launcher | `SomeApp` |
| `<Comment>` | Short one-line description shown under the name | `Open-source widget toolkit` |
| `<Categories>` | Semicolon-separated FreeDesktop categories (see the [categories rule](#choosing-categories)) | `Graphics;2DGraphics;` |
| `<user>` | Your Linux username, used only in the absolute `Exec=` path | `you` |

The rest of this guide relies on `~/` (your home directory), which works on
any machine without editing.

---

## Sourcing the app's details

WHEN the owner provides an app-info URL (e.g. a `get…/apps/<name>` page) →
fetch that URL and pull the values from it. Do **not** chase links into the
source repository.

- `<AppName>` — the page's product name (the main heading).
- `<Comment>` — the page's one-line tagline/description (e.g. "Open-source PDF
  workbench").
- `<Categories>` — read the page's feature list, pick one Main Category plus
  optional Additional Categories from the [categories rule](#choosing-categories).

WHEN the owner gave no URL → ask only for the app names and the info-page
URLs. Never ask about files on disk.

---

## 1. Make the AppImage executable

A freshly downloaded AppImage often lacks the executable bit and will not
launch. For one file:

```bash
chmod +x ~/Applications/<file>.AppImage
```

For several files list each exact name in the same `chmod` line, e.g.
`chmod +x ~/Applications/foo.AppImage ~/Applications/bar.AppImage`. To check,
run `ls -l`; the permission string should contain an `x`
(e.g. `-rwxrwxr-x`).

---

## 2. Name the AppImage

Use a stable, unversioned name so the `.desktop` entry and icon never need to
change when the app updates. The version number is not part of the filename:

```bash
cd ~/Applications
mv <AppImage> <app>.AppImage
```

Adjust the old filename to match the version you downloaded.

---

## 3. Install the app icon

Extract the icon from each named AppImage and install it into the icon theme.
The script below picks the best icon automatically (largest `.png` in
`hicolor/*/apps`, then `.svg`, then `.DirIcon`) and verifies the
result.

Replace `file1.AppImage file2.AppImage` on the first line with the exact
AppImage filenames the owner gave.

```bash
mkdir -p ~/.local/share/icons/hicolor/scalable/apps

for file in <file1>.AppImage <file2>.AppImage; do
  app="${file%.AppImage}"
  AIMG="$HOME/Applications/$file"
  [ -f "$AIMG" ] || { echo "$app: SKIP — $AIMG not found"; continue; }
  [ -x "$AIMG" ] || chmod +x "$AIMG"

  TMP=$(mktemp -d)
  (cd "$TMP" && "$AIMG" --appimage-extract >/dev/null 2>&1)
ICON= EXT=
  # Prefer largest PNG from hicolor size dirs
  PNGS=$(find "$TMP/squashfs-root/usr/share/icons/hicolor/"*/apps -maxdepth 1 -name '*.png' 2>/dev/null)
  [ -n "$PNGS" ] && ICON=$(echo "$PNGS" | xargs ls -S 2>/dev/null | head -1) && EXT="png"
  # Fall back to SVG
  if [ -z "$ICON" ]; then
    for DIR in "$TMP/squashfs-root/usr/share/icons/hicolor/scalable/apps" \
               "$TMP/squashfs-root/usr/share/icons/hicolor/"*/apps; do
      [ -d "$DIR" ] || continue
      ICON=$(ls "$DIR"/*.svg 2>/dev/null | head -1)
      [ -n "$ICON" ] && { EXT="svg"; break; }
    done
  fi
  # Last resort: .DirIcon
  if [ -z "$ICON" ]; then
    ICON=$(find "$TMP/squashfs-root" \( -name '.DirIcon' -o -name '*.DirIcon' \) 2>/dev/null | head -1)
    [ -n "$ICON" ] && EXT="png"
  fi

  if [ -n "$ICON" ]; then
   cp "$ICON" "$HOME/.local/share/icons/hicolor/scalable/apps/$app.$EXT"
    echo "$app: icon OK ($(file -b "$HOME/.local/share/icons/hicolor/scalable/apps/$app".* 2>/dev/null | head -1))"
  else
    echo "$app: WARNING — no icon found inside the AppImage"
  fi
  rm -rf "$TMP"
done
```

---

## 4. Create the `.desktop` entry

Each app needs one file in `~/.local/share/applications/` to appear in the
launcher.

Replace `file1.AppImage file2.AppImage` on the first line with the exact
AppImage filenames the owner gave. Fill in `<AppName>`, `<Comment>` and
`<Categories>` on the four marked lines using the
[sourcing rules](#sourcing-the-apps-details) and
[categories rule](#choosing-categories).

The unquoted heredoc (`<<DESKTOP`) expands `$HOME` so the file on disk always
gets a literal absolute path. Never quote that heredoc.

```bash
mkdir -p ~/.local/share/applications

for file in <file1>.AppImage <file2>.AppImage; do
  app="${file%.AppImage}"
  case "$app" in
# ── one case arm per app: name, comment, categories ──
    app1) name="<AppName>"; comment="<Comment>"; cats="<Categories>" ;;
    app2) name="<AppName>"; comment="<Comment>"; cats="<Categories>" ;;
# ───────────────────────────────────────────────────────
  esac

  # Pick the right icon extension (whatever was installed by Section 3)
  if [ -f "$HOME/.local/share/icons/hicolor/scalable/apps/$app.png" ]; then
    iconp="$HOME/.local/share/icons/hicolor/scalable/apps/$app.png"
  elif [ -f "$HOME/.local/share/icons/hicolor/scalable/apps/$app.svg" ]; then
    iconp="$HOME/.local/share/icons/hicolor/scalable/apps/$app.svg"
  else
    iconp="$HOME/.local/share/icons/hicolor/scalable/apps/$app.png"
  fi

  cat > "$HOME/.local/share/applications/$app.desktop" <<DESKTOP
[Desktop Entry]
Type=Application
Name=$name
Comment=$comment
Exec=$HOME/Applications/$file
Icon=$iconp
Terminal=false
Categories=$cats
DESKTOP
  echo "$app: .desktop written"
done

# Refresh caches if the tools exist (standard practice, harmless when absent)
command -v gtk-update-icon-cache >/dev/null 2>&1 && gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor 2>/dev/null
command -v update-desktop-database >/dev/null 2>&1 && update-desktop-database ~/.local/share/applications 2>/dev/null

echo "Done"
```

The `Name=` value is what you type to find the app. `Categories=` determines
which menu group it appears under.

`Icon=` must be an absolute path. The code block detects whether the installed
icon is `.png` or `.svg` and writes the full path. Never use a bare theme name
(e.g. `Icon=deckcraft`) — the file must contain
`Icon=/home/<user>/.local/share/icons/hicolor/scalable/apps/<app>.png` (or
`.svg`).

### Choosing `Categories=`

`Categories=` must contain exactly one **Main Category** from the FreeDesktop
spec's closed list:

`AudioVideo` `Audio` `Video` `Development` `Education` `Game`
`Graphics` `Network` `Office` `Science` `Settings` `System` `Utility`

After the main category you may add **Additional Categories** from
[the full registry](https://specifications.freedesktop.org/menu-spec/latest/category-registry.html)
that match the app's stated purpose. Each entry ends with a `;`.
Examples: `Office;Viewer;`, `Graphics;2DGraphics;RasterGraphics;`,
`AudioVideo;Video;`.

### `Exec=` must be an absolute path

The value written into the `.desktop` file must be a literal absolute path
(e.g. `/home/you/Applications/<file>.AppImage`). A literal `$HOME` or `~` in
the file will not be expanded by most launchers.

The unquoted heredoc in the command block handles this: `$HOME` is expanded by
the shell at write time, so the file on disk contains the absolute path.

After creating the `.desktop` file the launcher will pick it up on its own. If
it does not, restart the shell or log out and back in.

---

## 5. Updating an app manually

Many AppImages do not self-update reliably, so download the newest release
from the project's release or download page, then swap files while preserving
the stable name.

For one app:

```bash
cd ~/Applications
# keep the old version, tagged with its version number
mv <app>.AppImage <app>-<old-version>.AppImage

# give the newly downloaded file the stable name
mv ~/Downloads/<AppImage> <app>.AppImage
chmod +x <app>.AppImage
```

For several apps, repeat the three `mv`/`chmod` lines for each app inside one
`cd ~/Applications` block, using the exact filenames the owner gave. Then
because the `.desktop` entry keeps pointing at the stable `<file>.AppImage` and
the icon path never changes, no other files need updating.

### Notes

- Keep the old versioned file if you want a rollback copy, or delete it once
  you are happy with the new version.
- Make sure the new file is still executable after renaming (`chmod +x`
  handles this).
- The icon is bundled inside the AppImage, so a simple version swap does not
  change the icon. Re-extract only when the developer changed the icon.

---

## Combined: icon and .desktop entry

Use when the owner needs the icon **and** the `.desktop` entry for one or more
AppImages whose filename is already the stable name.

The Agent fills in only the data rows inside the `DATA` block at the
bottom — one `file.AppImage|Name|Comment|Categories` row per AppImage the
owner gave. The rest of the block is copied verbatim.

The script acts only on the files listed in `DATA`. Any other file in
`~/Applications` is never touched.

```bash
mkdir -p ~/.local/share/applications ~/.local/share/icons/hicolor/scalable/apps

PASS=0 TOTAL=0
while IFS='|' read -r file name comment cats; do
  case "$file" in \#*|"") continue ;; esac
  TOTAL=$((TOTAL + 1))
  app="${file%.AppImage}"

  AIMG="$HOME/Applications/$file"
  if [ ! -f "$AIMG" ]; then
    echo "$app: SKIP — $AIMG not found"
    continue
  fi
  [ -x "$AIMG" ] || chmod +x "$AIMG"

  TMP=$(mktemp -d)
  (cd "$TMP" && "$AIMG" --appimage-extract >/dev/null 2>&1)

  ICON= EXT=
  # Prefer largest PNG from hicolor size dirs
  PNGS=$(find "$TMP/squashfs-root/usr/share/icons/hicolor/"*/apps -maxdepth 1 -name '*.png' 2>/dev/null)
  [ -n "$PNGS" ] && ICON=$(echo "$PNGS" | xargs ls -S 2>/dev/null | head -1) && EXT="png"
  # Fall back to SVG
  if [ -z "$ICON" ]; then
    for DIR in "$TMP/squashfs-root/usr/share/icons/hicolor/scalable/apps" \
               "$TMP/squashfs-root/usr/share/icons/hicolor/"*/apps; do
      [ -d "$DIR" ] || continue
      ICON=$(ls "$DIR"/*.svg 2>/dev/null | head -1)
      [ -n "$ICON" ] && { EXT="svg"; break; }
    done
  fi
  # Last resort: .DirIcon
  if [ -z "$ICON" ]; then
    ICON=$(find "$TMP/squashfs-root" \( -name '.DirIcon' -o -name '*.DirIcon' \) 2>/dev/null | head -1)
    [ -n "$ICON" ] && EXT="png"
  fi

  if [ -n "$ICON" ]; then
    cp "$ICON" "$HOME/.local/share/icons/hicolor/scalable/apps/$app.$EXT"
  fi
  rm -rf "$TMP"

  # Unquoted heredoc expands $HOME/$name/$comment/$file/$app/$EXT to absolute values
  cat > "$HOME/.local/share/applications/$app.desktop" <<DESKTOP
[Desktop Entry]
Type=Application
Name=$name
Comment=$comment
Exec=$HOME/Applications/$file
Icon=$HOME/.local/share/icons/hicolor/scalable/apps/$app.$EXT
Terminal=false
Categories=$cats
DESKTOP

  if [ -f "$HOME/.local/share/icons/hicolor/scalable/apps/$app.png" ] || \
     [ -f "$HOME/.local/share/icons/hicolor/scalable/apps/$app.svg" ]; then
    echo "$app: OK"
    PASS=$((PASS + 1))
  else
    echo "$app: WARNING — no icon installed"
  fi
done <<'DATA'
# Replace every row below — one `file.AppImage|Name|Comment|Categories`
# row per app the owner named. Remove the leading # on each real row.
# file1.AppImage|AppOne|Short tagline.|AudioVideo;
# file2.AppImage|AppTwo|Another tagline.|Office;
DATA

# Refresh caches if the tools exist (standard practice, harmless when absent)
command -v gtk-update-icon-cache >/dev/null 2>&1 && gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor 2>/dev/null
command -v update-desktop-database >/dev/null 2>&1 && update-desktop-database ~/.local/share/applications 2>/dev/null

echo "$PASS/$TOTAL installed"
```

### How the Agent fills this block

1. Fetch each URL the owner gave (see [Sourcing the app's
   details](#sourcing-the-apps-details)) and extract `<AppName>` and
   `<Comment>`.
2. Use the exact AppImage filename the owner gave as `<file>.AppImage`. Strip
   `.AppImage` to get `<app>` for the icon and `.desktop` name.
3. Pick `<Categories>` using the [categories rule](#choosing-categories).
4. Replace the comment lines inside the `DATA` block with one
   `file.AppImage|Name|Comment|Categories` row per app the owner named.
   Remove the leading `#`.
5. Copy the rest of the block verbatim. Render the whole thing as the single
   answer.