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

Every file currently over the 2MB budget, and why it is accepted. If you add
to this list, say why — an undocumented file over budget should read as "not
yet optimised", not "fine".

| File | Size | Dimensions | Why |
| --- | --- | --- | --- |
| `breaklight_prototype_web.png` | 4.65MB | 1920x1079 | Photograph in a lossless container; see below. |
| `spacebeaver_prototype_web.png` | 4.46MB | 1920x1326 | Photograph in a lossless container; see below. |
| `spacebeaver-draft.png` | 2.96MB | 1354x1241 | Artwork, already well under the dimension cap. |
| `transflag_patch.png` | 2.55MB | 1365x962 | Merch artwork, already well under the dimension cap. |
| `cyberpunktits_transcolours.png` | 2.53MB | 1302x1380 | Merch artwork, already well under the dimension cap. |

The bottom three are flat-colour artwork rather than photographs, and they are
already smaller than the 2560px cap, so resizing is the only lever PNG offers
and it would cost real quality on files that may serve as print sources. Left
at full quality deliberately.

The two `_web` PNGs are different: they are photographs, and each has a `.jpg`
twin (`breaklight_prototype.jpg` at 1.16MB, `spacebeaver_prototype.jpg` at
0.83MB) that is the same picture — verified by pixel comparison — at a quarter
of the size. **Reference the `.jpg`, not the `.png`.** The PNGs survive only
because deleting from an asset archive is a deliberate decision rather than a
cleanup; with the masters now in `originals/`, dropping them is safe whenever
someone wants to.

Two things that are *not* exceptions, in case they look like it:

- `2122_hi_res.png` is 1,999,869 bytes — just inside the budget, despite the
  name. Nothing to do, but it has no headroom if it is ever re-encoded.
- `*.xcf` — GIMP sources, all under 1MB. They are not images the site serves,
  and they are well within budget anyway.

## History

`images/` was 126MB before the July 2026 pass, almost all of it a handful of
files stored at capture resolution (`generic-bg.jpg` alone was a 14100x3456,
50MB panorama). It is now 46MB, with every touched master preserved in
`originals/`. Restoring those masters cost the repository nothing: the same
blobs were already in history, and git stores content once.
