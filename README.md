# Wardrobe (Fabric 1.21.11)

Adds a rotating 3D skin preview + a "Wardrobe" button to the pause (Esc) menu.
The Wardrobe screen lets you import any number of local skin PNGs and pick
which one your character wears. This is a **client-side cosmetic override
only** — it does not touch your real Mojang account skin, so people without
the mod (or without this skin's file) still see your normal skin.

# Wardrobe (Fabric 1.21.11)

Adds a rotating 3D skin preview + a "Wardrobe" button to the pause (Esc) menu.
The Wardrobe screen lets you import any number of local skin PNGs and pick
which one your character wears. This is a **client-side cosmetic override
only** — it does not touch your real Mojang account skin, so people without
the mod (or without this skin's file) still see your normal skin.

## Easiest way to get a jar: let GitHub build it for you (no Java/Gradle needed)

1. Go to github.com, sign in (or make a free account), click the **+** in
   the top right → **New repository**. Name it anything, keep it Public,
   create it.
2. On the new repo's page click **"uploading an existing file"** (or
   **Add file → Upload files**). Drag this whole unzipped `wardrobe-mod`
   folder in - GitHub keeps the folder structure - and commit.
3. Click the **Actions** tab at the top of the repo. A workflow called
   "Build mod jar" should already be running (it starts automatically on
   upload). Wait for the green checkmark - first run takes a few minutes.
4. Click into that finished run, scroll to **Artifacts**, and download
   **wardrobe-mod-jar**. Unzip that download and you'll have the real
   `wardrobe-1.0.0.jar`.
5. Install Fabric Loader 0.16.10+ for Minecraft 1.21.11 from fabricmc.net,
   install Fabric API 0.141.5+1.21.11 from Modrinth into your `.minecraft/mods`
   folder, then drop `wardrobe-1.0.0.jar` in that same folder and launch.

If the Actions build fails red instead of green, click into it and read the
error - most likely it hit one of the two version-fragility spots called out
below, and the log will show exactly which line to fix.

## Building locally instead (more setup, but works offline once configured)

0. **One-time fix**: this zip includes `gradle/wrapper/gradle-wrapper.properties`
   but not the actual `gradle-wrapper.jar` binary (it can't be produced
   offline). Before step 4 below, either:
   - open the project folder in IntelliJ IDEA with the Gradle plugin - it
     regenerates the wrapper jar automatically on import, or
   - if you already have Gradle installed some other way, run `gradle wrapper`
     once inside this folder to generate it, or
   - download `gradle-wrapper.jar` from any other Fabric mod template repo's
     `gradle/wrapper/` folder (it's identical for a given Gradle version) and
     drop it next to `gradle-wrapper.properties`.
   After that, `gradlew`/`gradlew.bat` work normally for good.
1. Install a JDK 21.
2. `./gradlew genSources` (or genSourcesWithVineflower, whatever your Loom
   version calls it) so your IDE can decompile Minecraft and you can jump to
   real method/field names.
3. `./gradlew runClient` to launch a dev client with the mod loaded.
4. `./gradlew build` produces the distributable jar in `build/libs/`.

## What to check first if it doesn't compile

Fabric mods are pinned to exact obfuscation mappings, and a couple of the
classes this mod touches are genuinely new/renamed as of very recent 1.21.x
snapshots, so two spots are flagged in comments and worth double-checking in
your IDE (right click the class → "Copy target reference" with the Minecraft
Development plugin's Mixin support installed):

- **`GameMenuScreenMixin`** targets a method called `initWidgets` to append
  the Wardrobe button after the vanilla pause-menu buttons are built. If your
  mappings name it differently, update the `@Inject(method = "...")` string.
- **`AbstractClientPlayerMixin`** overrides `getSkinTextures()` and rebuilds
  a `SkinTextures` record. Around 1.21.9 Mojang reworked this into a
  differently-shaped `PlayerSkin` record built from `ClientAsset.Texture`
  values instead of plain `Identifier`s. If your decompiled sources show that
  newer shape, adjust the constructor call in this file to match — the
  compiler error will point at exactly which accessor names changed.

Everything else (screens, buttons, JSON persistence, texture loading, the
drag-to-rotate math, the native file picker) uses stable, long-standing
Minecraft/Fabric APIs and shouldn't need changes.

## How it works

- `SkinManager` copies imported PNGs into
  `.minecraft/config/wardrobe/skins/`, tracks them (+ which one is selected)
  in `wardrobe.json`, and registers each as a GPU texture.
- `AbstractClientPlayerMixin` makes the local player report the selected
  wardrobe texture instead of its normal skin whenever one is selected.
- `PlayerPreviewRenderer` temporarily rotates the *actual* client player
  entity's yaw/pitch, renders it via vanilla's own
  `InventoryScreen.renderEntityInInventory` (the same call the normal
  inventory screen uses to draw you), then restores its real orientation so
  gameplay isn't affected.
- `PreviewRotation` just accumulates click-and-drag mouse deltas into a free
  360° yaw and a clamped pitch — that's the "spin the model" behaviour, used
  by both the pause-screen preview and the bigger one in the Wardrobe screen.
- `WardrobeScreen` lists saved skins as buttons, a native OS file dialog
  (LWJGL's `TinyFileDialogs`) for importing new PNGs, and a delete button
  per entry.

## Ideas for later

- Detect slim vs. classic arm model automatically when applying the skin to
  the render layer (currently detected on import and stored, but the actual
  `PlayerEntityModel` swap between wide/slim arms isn't wired up yet — you'd
  hook `PlayerModelType`/`getModel()` similarly to the texture override).
- Add capes: read the classic-vs-newer skin record's cape field the same way.
- Sync a chosen skin to a resource pack or a server-side mod so *other*
  players with the same mod can see it too (true synced custom skins need a
  server-side component; this client-only version only changes what you see
  of yourself, plus what appears in these preview panels).
