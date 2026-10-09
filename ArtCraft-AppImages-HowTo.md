# How to Add and Update AppImages (ArtCraft Suite)

This guide documents the process used to install and integrate the ArtCraft
Crafting Apps (PhotoCraft, VectorCraft, FilmCraft, PrintCraft, EffectCraft) as
AppImages on an `~/Applications`-based Linux desktop running a Wayland
compositor (Niri) with the DankMaterialShell launcher.

## Prerequisites

- The AppImages are stored in `~/Applications` (i.e. `/home/USERNAME/Applications`).
- The launcher finds applications through standard FreeDesktop `.desktop` entry
  files, which are stored in `~/.local/share/applications/`.
- App icons are stored in `~/.local/share/icons/hicolor/scalable/apps/`.

Change every `/home/USERNAME` path below to your own username.

---

## 1. Make each AppImage executable

A freshly downloaded AppImage often lacks the executable bit and will not
launch. Set it:

```bash
chmod +x ~/Applications/*.AppImage
```

To confirm, run `ls -l`; the permission string should contain an `x`
(e.g. `-rwxrwxr-x`).

---

## 2. Name the AppImages

Use a stable, unversioned name so the `.desktop` entries and icons never need
to change when the app updates. The version number is not part of the filename:

```bash
cd ~/Applications
mv photocraft-0.3.0-linux-x86_64.AppImage photocraft.AppImage
mv vectorcraft-0.4.0-linux-x86_64.AppImage vectorcraft.AppImage
mv filmcraft-0.2.1-linux-x86_64.AppImage  filmcraft.AppImage
mv printcraft-0.2.1-linux-x86_64.AppImage printcraft.AppImage
mv effectcraft-0.6.0-linux-x86_64.AppImage effectcraft.AppImage
```

Adjust the old filenames to match whatever version you downloaded.

---

## 3. Extract and install each app icon

AppImages bundle their own icons. Extract the app's contents to a temporary
directory, copy the icon out as a `.png`, place it in the icon theme, then
clean up.

```bash
tmp=$(mktemp -d)
cd "$tmp"
~/Applications/effectcraft.AppImage --appimage-extract >/dev/null 2>&1
cp "$tmp/squashfs-root/usr/share/icons/hicolor/512x512/apps/ai.storyteller.effectcraft.png" \
   ~/.local/share/icons/hicolor/scalable/apps/effectcraft.png
cd ~
rm -rf "$tmp"
```

Repeat for each app, changing both the AppImage name and the output icon name.

### Finding the right icon file

The icon is not always named the way you expect. If the `cp` fails with
"cannot stat", list what was actually extracted first:

```bash
tmp=$(mktemp -d)
cd "$tmp"
~/Applications/effectcraft.AppImage --appimage-extract >/dev/null 2>&1
find "$tmp/squashfs-root" \( -name '*.png' -o -name '*.svg' -o -name '.DirIcon' -o -name '*.DirIcon' \) 2>/dev/null
cd ~
rm -rf "$tmp"
```

Choose the largest `.png` (or the `.svg`), then update the `cp` source path
accordingly. For the ArtCraft apps, the internal icon name is often
reverse-DNS style (`ai.storyteller.effectcraft`), while the installed file must
be named with the simple app name (`effectcraft.png`) to match the `.desktop`
entry.

### Verify the icon is a real PNG

```bash
file ~/.local/share/icons/hicolor/scalable/apps/effectcraft.png
```

Expected output contains `PNG image data` (e.g. `512 x 512, 8-bit/color RGBA`).
If it says `SVG Scalable Vector Graphics image`, rename the extension to `.svg`
and update the `.desktop` `Icon=` path accordingly.

---

## 4. Create the `.desktop` entries

Each app needs one file in `~/.local/share/applications/`. This is what makes
it appear in the launcher. The `Exec=` line points at the unversioned AppImage,
and `Icon=` points at the installed icon.

```bash
cat > ~/.local/share/applications/effectcraft.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=EffectCraft
Comment=Open-source motion graphics and visual effects
Exec=/home/USERNAME/Applications/effectcraft.AppImage
Icon=/home/USERNAME/.local/share/icons/hicolor/scalable/apps/effectcraft.png
Terminal=false
Categories=AudioVideo;Video;Graphics;
EOF
```

The `Name=` value is what you type to find the app. `Categories=` determines
which menu group it appears under. Create one block per app, changing `Name`,
`Comment`, `Exec`, `Icon`, and `Categories` to suit.

Reference categories for the other ArtCraft apps:

| App | Categories |
|---|---|
| PhotoCraft | `Graphics;2DGraphics;RasterGraphics;` |
| VectorCraft | `Graphics;VectorGraphics;` |
| FilmCraft | `AudioVideo;Video;` |
| PrintCraft | `Office;Viewer;` |
| EffectCraft | `AudioVideo;Video;Graphics;` |

After creating the files, the launcher will usually pick them up on its own.
If an entry does not appear, restart the shell or log out and back in.

---

## 5. Updating an app manually

The developer does not consistently embed self-update metadata in the
AppImages, so the reliable method is to download the newest release from the
project's GitHub releases page, then swap files while preserving the stable
name.

For each app:

1. Go to the app's releases page (e.g. `https://github.com/storytold/photocraft/releases`).
2. Download the latest `*-linux-x86_64.AppImage`.
3. In `~/Applications`, rename the current file to include its old version
   number, then rename the freshly downloaded file to the clean name.

```bash
cd ~/Applications
# keep the old version, tagged with its version number
mv photocraft.AppImage photocraft-0.3.0.AppImage

# give the newly downloaded file the stable name
mv ~/Downloads/photraft-0.5.0-linux-x86_64.AppImage photocraft.AppImage
chmod +x photocraft.AppImage
```

Because the `.desktop` entry keeps pointing at `photocraft.AppImage` and the
icon path never changes, no other files need updating.

### Notes

- Keep the old versioned file if you want a rollback copy, or delete it once
  you are happy with the new version.
- Make sure the new file is still executable after renaming (run `chmod +x`
  if needed).
- The icon is bundled inside the AppImage, so a simple version swap does not
  change the icon; no need to re-extract unless the developer changed the icon.