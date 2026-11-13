# Tomba! 2 decomp

A decompilation of **Tomba! 2: The Evil Swine Return** (USA, `SCUS-94454`), set up with
[splat](https://github.com/ethteck/splat). It currently holds the build-matched split configuration for every
code image on the disc, a symbol map from our own reverse engineering, and the RE notes. No C is matched yet.

The PC port [psx-tomba](https://github.com/SomeoneIsWorking/psx-tomba) consumes this repository as a submodule and
reads its guest addresses from the symbol map.

## Bring your own disc

No game bytes are ever committed here: not the executables, not overlays, not the split assembly or data. Copy the
files from your own disc into `disc/` (gitignored), keeping the disc's layout:

```
disc/SCUS_944.54
disc/MAIN.EXE
disc/BIN/START.BIN  DEMO.BIN  GAME.BIN  SOP.BIN  OPN.BIN  CRD.BIN  A00.BIN .. A0L.BIN
```

Each config carries the file's SHA-1 and splat refuses a mismatch. The images are listed in `docs/overlays.md`.

## Split

```
uv run --frozen splat split config/main.yaml
uv run --frozen splat split config/scus_944.54.yaml
uv run --frozen splat split config/overlays/a00.yaml     # one config per overlay
```

Output goes to `asm/<image>/` and `build/`, both gitignored. splat is pinned in `pyproject.toml`/`uv.lock`.

## Layout

| path | what |
|---|---|
| `config/*.yaml`, `config/overlays/*.yaml` | splat configs, one per image |
| `config/symbols/<image>.txt` | the symbol map, splat `symbol_addrs` format; overlay configs also read `main.txt` |
| `docs/overlays.md` | the image set, load slots and segment boundaries |
| `docs/re-frontier.md` | the ordered RE evidence chain (`re_frontier.py`) |
| `docs/notes/` | behaviour notes on guest routines |
| `docs/unplaced-overlay-names.md` | named overlay routines whose image is not yet recorded |
| `docs/prior-art.md` | other Tomba! 2 work, licences and what we take |

Notes cite psx-tomba paths (`game/...`, `docs/issues/...`) where a port owner or an issue holds the evidence.

## Licence

MIT (`LICENSE`), covering this repository's own files only. Tomba! 2 is © Whoopee Camp / Sony Computer
Entertainment; nothing of it is distributed here.

Built with AI assistance (Claude and other LLMs).
