# pcfweb-assets

Image assets for [pcfweb](https://github.com/pigsCanFlyLabs/pcfweb). Kept out
of that repo so its checkout stays small; `pcfweb/build.sh` expects this one
cloned as a sibling directory and copies the web set into
`main/static/assets/images/` before building the container image.

## Layout

| Directory | Ships in the container | What lives here |
| --- | --- | --- |
| `images/` | **Yes** | Web-ready copies. This is the whole directory `build.sh` copies. |
| `originals/` | No | Full-resolution masters. Archive only. |

`build.sh` copies `../pcfweb-assets/images` and nothing else, so anything in
`originals/` is safe to keep at whatever size it was captured at — it never
reaches the image.

## Sizing rules for `images/`

Everything in `images/` is copied into the container **and** duplicated by
`collectstatic`, so a file here costs roughly twice its size in an artifact
that Kubernetes re-pulls on every rollout. Budgets:

- **Long edge ≤ 2560px.** Retina-generous for a full-width hero; nothing on
  the site displays larger.
- **≤ 2MB per file**, with a couple of documented exceptions below.

If you have something bigger, put the master in `originals/` and a resized
copy in `images/` under the same filename. `pcfweb/build.sh` fails the build
if a file in `images/` exceeds a hard ceiling, so an accidentally-committed
master gets caught before it ships rather than after.

Re-encoding settings used for the current set: JPEG at quality 82,
progressive, ICC profile preserved; PNG re-saved with `optimize=True`. PNG has
no quality knob, so an oversized PNG photograph can only be fixed by
dimensions — those are capped at 1920px instead.

### Known exceptions

- `2122_hi_res.png` (2.0MB) — the filename says the resolution is the point.
- `breaklight_prototype_web.png` (4.7MB) and `spacebeaver_prototype_web.png`
  (4.5MB) — photographs in a lossless container. Each has a `.jpg` twin
  (`breaklight_prototype.jpg`, `spacebeaver_prototype.jpg`) that is the same
  picture at ~1MB and is what any page should reference. The PNGs are kept
  only because removing files from an asset archive is a deliberate decision,
  not a cleanup; if nothing grows to need them, they can go.
- `*.xcf` — GIMP sources. They are small, and are not images the site serves.

## History

`images/` was 126MB before the July 2026 pass, almost all of it a handful of
files stored at capture resolution (`generic-bg.jpg` alone was a 14100x3456,
50MB panorama). It is now 46MB, with every touched master preserved in
`originals/`. Restoring those masters cost the repository nothing: the same
blobs were already in history, and git stores content once.
