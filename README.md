# UECAST-BO7-FIX

Fixes UECast's broken semantic-based material/texture import for BO7 models in Unreal Engine 5.

## The problem

[UECast](https://github.com/o-Astral-o/UECast) imports `.cast` files (exported via [Saluki](https://github.com/Scobalula/Saluki)) directly into Unreal, auto-binding texture parameters and setting texture import/compression settings based on each texture's *semantic index* — a number encoded per-texture that's supposed to say "this one's the albedo map," "this one's the normal map," and so on.

For Black Ops 7, that semantic table changed upstream, and UECast's own table (hardcoded in `UECast.cpp`) hasn't been updated to match. The result:

- Real color/albedo textures end up bound to the wrong material parameter (often `nogMap`), while the actual normal/gloss/occlusion (NOG) texture ends up somewhere else entirely (often `emissiveMap`)
- `colorMap` is frequently left empty
- Textures can also get imported with the wrong compression settings / sRGB flag, since UECast decides those from the same stale table

None of this is a bug in Saluki's export — it labels what it doesn't know generically (`unk_semantic_X`, `extra0`...`extraN`) rather than guessing wrong. The mislabeling happens entirely on UECast's side. Confirmed with the UECast author: this isn't planned to be fixed upstream, so this repo works around it instead.

## The fix

Rather than trying to patch UECast's semantic table (which turned out to be inconsistent across different shader techsets — the same numeric semantic means different things depending on which shader a given part uses), this takes a different approach:

1. **Read ground truth, not the semantic table.** UECast's mislabeling is wrong but *consistent* on a given project: the real color source reliably ends up in `nogMap`, and the real NOG source reliably ends up in `emissiveMap`. This script reads directly off those bindings instead of trusting what UECast *named* them.
2. **Re-derive the actual channel data from source**, rather than trying to reassign already-imported (and possibly already-mis-compressed) textures. The real NOG/albedo source files are hash-matched back to their raw exports on disk and run through a faithful Python port of [GameImageUtil](https://github.com/Scobalula/GameImageUtil)'s CoD-specific channel-split algorithms (ported directly from GameImageUtil's own source, with attribution — see [Credits](#credits)).
3. **Convert every material to a simpler parent** (`CoDBase`, exposing just `colorMap`/`normalMap`) so nothing ever depends on UECast's semantic guessing again.

## What it does, step by step

Given a SkeletalMesh already imported via UECast:

1. Scans every Material Instance on the mesh, reads whatever's bound to `nogMap` (→ real color source) and `emissiveMap` (→ real NOG source), and copies the matching raw export files (matched by hash, tolerant of minor filename differences) into `_gameimageutil_staging/_color/` and `_gameimageutil_staging/_nog/` next to the model's export folder.
2. Runs those staged files through the actual NOG-decode and specular/albedo-split math (hemi-octahedron normal reconstruction, metalness-based color/spec separation) — no GameImageUtil GUI required.
3. Imports the results, sets correct compression/sRGB on the normal map, swaps every material's parent to `CoDBase`, and wires up `colorMap`/`normalMap`.

Everything after selecting the mesh is automatic — one folder picker for the model's disk export location, then it runs end to end.

## Requirements

- Unreal Engine 5 with [UECast](https://github.com/o-Astral-o/UECast) installed
- A `CoDBase` material (or your own equivalent) exposing `colorMap` and `normalMap` texture parameters
- Python 3 with `numpy` and `Pillow` installed and available as `python` in a normal terminal on the same machine:
  ```
  pip install numpy pillow
  ```
- Windows (the folder picker uses a native Windows dialog via PowerShell)

## Setup

1. Put `UECAST-BO7-FIX.py` and `cod_texture_split.py` in the same folder.
2. Open `UECAST-BO7-FIX.py` and edit the config block near the top:
   ```python
   PYCONVERT_DIR = r"C:\path\to\the\folder\with\both\scripts"
   CODBASE_MATERIAL = "/Game/Path/To/Your/CoDBase.CoDBase"
   DEFAULT_BROWSE_ROOT = r"C:\path\to\your\Saluki\exports"  # just where the folder picker starts, not a hard requirement
   ```

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

## Known limitations

- Assumes the `nogMap` → real color / `emissiveMap` → real NOG pattern holds. This was consistent across every model tested, but a different shader techset could theoretically break that assumption — if a converted material comes out wrong, check what's actually bound to those two slots before assuming the script is at fault.
- Only wires up `colorMap`/`normalMap`. Gloss, occlusion, and specular are computed as part of the split (and written to disk) but not currently used downstream, since `CoDBase` doesn't expose them.
- If a texture's alpha channel carries no real per-pixel data (some exports have it flattened to a constant), the specular/albedo split falls back to passing RGB straight through rather than applying the metalness math, which would otherwise force the whole texture to black.
- Tested specifically on BO7 exports. Other post-IW8 titles weren't verified.

## Credits

- [GameImageUtil](https://github.com/Scobalula/GameImageUtil) by Scobalula — `cod_texture_split.py` is a direct Python port of GameImageUtil's `CoDNOGProcessor` and `CoDFusedCSProcessor`, GPL-3.0 licensed. This repo is GPL-3.0 for that reason.
- [UECast](https://github.com/o-Astral-o/UECast) by Astral
- [Saluki](https://github.com/Scobalula/Saluki) by Scobalula / echo000

## License

GPL-3.0. See [LICENSE](LICENSE).
