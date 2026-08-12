<h1 align="center">
  <br>
  UECAST BO7 FIX
  <br>
</h1>

<h4 align="center">Fixes UECast's broken semantic-based material/texture import for BO7 models in Unreal Engine 5.</h4>

<div align="center">
  <a href="https://github.com/Politohh/UECAST-BO7-FIX/releases"><img src="https://img.shields.io/github/downloads/Politohh/UECAST-BO7-FIX/total"></a>
  <a href="https://paypal.me/politoggs"><img src="https://img.shields.io/badge/Donate-Paypal-orange?style=flat-square"></a>
</div>

<p align="center">
  <a href="#the-problem">The problem</a> •
  <a href="#eli5">ELI5</a> •
  <a href="#preview">Preview</a> •
  <a href="#how-it-works">How it works</a> •
  <a href="#requirements">Requirements</a> •
  <a href="#installation">Installation</a> •
  <a href="#usage">Usage</a> •
  <a href="#known-limitations">Known limitations</a> •
  <a href="#credits">Credits</a>
</p>

---

## The problem

[UECast](https://github.com/o-Astral-o/UECast) imports `.cast` files (exported via [Saluki](https://github.com/Scobalula/Saluki)) directly into Unreal, auto-binding texture parameters and setting texture import/compression settings based on each texture's *semantic index* — a number encoded per-texture that's supposed to say "this one's the albedo map," "this one's the normal map," and so on.

For Black Ops 7, that semantic table changed upstream, and UECast's own table (hardcoded in `UECast.cpp`) hasn't been updated to match. The result:

- Real color/albedo textures end up bound to the wrong material parameter (often `nogMap`), while the actual normal/gloss/occlusion (NOG) texture ends up somewhere else entirely (often `emissiveMap`)
- `colorMap` is frequently left empty
- Textures can also get imported with the wrong compression settings / sRGB flag, since UECast decides those from the same stale table

None of this is a bug in Saluki's export — it labels what it doesn't know generically (`unk_semantic_X`, `extra0`...`extraN`) rather than guessing wrong. The mislabeling happens entirely on UECast's side. Confirmed with the UECast author: this isn't planned to be fixed upstream, hence this repo.

---

## ELI5

BO7 broke a tool called UECast. When you import a BO7 character into Unreal Engine, the textures get put in the wrong places — the color texture ends up somewhere it shouldn't, the normal map (the bumpy detail texture) ends up somewhere else, and the actual color slot is just... empty. Black.

This happens because UECast decides where each texture goes by reading a little tag baked into the file that says "I'm the color one" or "I'm the normal one." For BO7, those tags got scrambled, and the person who makes UECast isn't planning to fix it.

So this tool fixes it for you automatically:

1. It looks at your messed-up materials and figures out where the *real* color and normal textures actually ended up (they're always in the same wrong spots, just consistently wrong, so it can find them).
2. It grabs the original texture files from your export folder and runs the correct math on them to split out the real color and normal maps properly.
3. It puts everything back where it's supposed to go, on a clean simple material.

You select your character in Unreal, run one script, pick a folder once, and it does all of that by itself. That's it.

---

## Preview

**Before** — freshly imported via UECast, textures scrambled

<img src="https://raw.githubusercontent.com/Politohh/UECAST-BO7-FIX/main/before.png">

**After** — same mesh, after running `UECAST-BO7-FIX.py`

<img src="https://raw.githubusercontent.com/Politohh/UECAST-BO7-FIX/main/after.png">

---

## How it works

Rather than patching UECast's semantic table (which turned out to be inconsistent across shader techsets — the same numeric semantic means different things depending on which shader a given part uses), this takes a different approach:

- **Reads ground truth, not the semantic table.** UECast's mislabeling is wrong but *consistent* on a given project: the real color source reliably ends up bound to `nogMap`, and the real NOG source reliably ends up bound to `emissiveMap`. This reads directly off those bindings instead of trusting what UECast *named* them.
- **Re-derives the actual channel data from source**, rather than reassigning already-imported (and possibly already-mis-compressed) textures. The real NOG/albedo source files are hash-matched back to their raw exports on disk and run through a faithful Python port of [GameImageUtil](https://github.com/Scobalula/GameImageUtil)'s CoD-specific channel-split algorithms — ported directly from GameImageUtil's own source, with attribution (see [Credits](#credits)).
- **Converts every material to a simpler parent** (`CoDBase`, exposing just `colorMap`/`normalMap`) so nothing ever depends on UECast's semantic guessing again.

Given a SkeletalMesh already imported via UECast, the full pipeline:

1. Scans every Material Instance on the mesh, reads whatever's bound to `nogMap` (→ real color source) and `emissiveMap` (→ real NOG source), and copies the matching raw export files (matched by hash, tolerant of minor filename differences) into `_gameimageutil_staging/_color/` and `_gameimageutil_staging/_nog/` next to the model's export folder.
2. Runs those staged files through the actual NOG-decode and specular/albedo-split math (hemi-octahedron normal reconstruction, metalness-based color/spec separation) — no GameImageUtil GUI required.
3. Imports the results, sets correct compression/sRGB on the normal map, swaps every material's parent to `CoDBase`, and wires up `colorMap`/`normalMap`.

Everything after selecting the mesh is automatic — one folder picker for the model's disk export location, then it runs end to end, in the background so the editor stays responsive.

---

## Requirements

- **Unreal Engine 5** with [UECast](https://github.com/o-Astral-o/UECast) installed
- A **CoDBase** material (or your own equivalent) exposing `colorMap` and `normalMap` texture parameters
- **Python 3.x** with `numpy` and `Pillow`, available as `python` in a normal terminal on the same machine:
  ```
  pip install numpy pillow
  ```
- **Windows** (the folder picker uses a native Windows dialog via PowerShell)

---

## Installation

1. Download the [latest release](https://github.com/Politohh/UECAST-BO7-FIX/releases)
2. Place these files all in the same folder:
   - `UECAST-BO7-FIX.py`
   - `cod_texture_split.py`
3. Open `UECAST-BO7-FIX.py` and edit the config block near the top:
   ```python
   PYCONVERT_DIR = r"C:\path\to\the\folder\with\both\scripts"
   CODBASE_MATERIAL = "/Game/Path/To/Your/CoDBase.CoDBase"
   DEFAULT_BROWSE_ROOT = r"C:\path\to\your\Saluki\exports"  # just where the folder picker starts
   ```

---

## Usage

1. Export a model from Saluki, import the `.cast` file(s) into UE via UECast as usual.
2. Select the imported **SkeletalMesh** in the Content Browser.
3. In UE's console command bar (dropdown set to **Cmd**, not Python):
   ```
   py "C:/path/to/UECAST-BO7-FIX.py"
   ```
4. A folder picker opens — navigate to that model's export folder (containing `_images`, `_mat_info`, and the `.cast` file) and select it.
5. Watch the Output Log. The editor stays responsive while the texture-split step runs in the background.
6. Check the result in the viewport.

`cod_texture_split.py` also runs standalone if you want to process a staging folder without going through the full pipeline:
```
python cod_texture_split.py "<path to a _gameimageutil_staging folder>"
```

---

## Known limitations

- **Specular output looks wrong when actually used.** The split math also produces a `_s` (specular) file for every color/albedo texture, ported faithfully from GameImageUtil's own algorithm — but wiring that into a material here produces an overly glossy, "wet-looking" result. The exact cause hasn't been tracked down (could be something specific to how `CoDBase` or the wider material setup handles specular, rather than the split math itself being wrong). Because of this, the pipeline deliberately never wires up `specularMap` — only `colorMap`/`normalMap` — and the `_s` files just sit unused on disk. If you want to use specular, treat it as experimental and expect to need extra tuning.
- Assumes the `nogMap` → real color / `emissiveMap` → real NOG pattern holds. This was consistent across every model tested, but a different shader techset could theoretically break that assumption — if a converted material comes out wrong, check what's actually bound to those two slots before assuming the script is at fault.
- Only wires up `colorMap`/`normalMap`. Gloss and occlusion are computed as part of the split (and written to disk) but not currently used downstream, since `CoDBase` doesn't expose them.
- If a texture's alpha channel carries no real per-pixel data (some exports have it flattened to a constant), the specular/albedo split falls back to passing RGB straight through rather than applying the metalness math, which would otherwise force the whole texture to black.
- Tested specifically on BO7 exports.

---

## Credits

- **[Scobalula](https://github.com/Scobalula)** — [GameImageUtil](https://github.com/Scobalula/GameImageUtil) and [Saluki](https://github.com/Scobalula/Saluki). `cod_texture_split.py` is a direct Python port of GameImageUtil's `CoDNOGProcessor` and `CoDFusedCSProcessor`, GPL-3.0 licensed — this repo is GPL-3.0 for that reason.
- **[Astral](https://github.com/o-Astral-o)** — [UECast](https://github.com/o-Astral-o/UECast)

---

**Please contact me if you find any bugs or have any suggestions.**
#### Twitter: @thatkidpolito
#### Discord: polito#2491
