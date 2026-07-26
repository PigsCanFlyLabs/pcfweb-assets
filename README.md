# pcfweb-assets

Image assets for [pcfweb](https://github.com/pigsCanFlyLabs/pcfweb). Kept out
of that repo so its checkout stays small; `pcfweb/build.sh` expects this one
cloned as a sibling directory and copies the web set into
`main/static/assets/images/` before building the container image.

## ⚠️ Git LFS is now required on any build host

New and changed images in this repository are stored in [Git LFS](https://git-lfs.com/).
A plain `git clone` gives you ~130-byte **pointer files** where those images
should be, unless LFS is installed and the objects have been fetched.

**Before running `pcfweb/build.sh`, on every build host:**

```sh
git lfs install     # once per machine/user
git lfs pull        # in this repository, after clone/fetch/checkout
```

### Why this matters more than it looks

`pcfweb/build.sh` does:

```sh
rm -rf main/static/assets/images
cp -af ../pcfweb-assets/images main/static/assets/
```

It copies whatever is on disk. If the objects were never materialised, it
copies the pointer files into the Docker image and **the site ships with
broken images**.

Nothing catches this today. `build.sh` validates that assets are not too
*large* (`ASSET_MAX_BYTES=5000000`) — a 130-byte pointer sails straight
through a maximum-size check. There is no minimum, no content-type check, and
no LFS check. The failure is completely silent:

- the build succeeds,
- the container image builds and pushes,
- the deploy is green,
- and the pages render with broken image icons, with no error in any log.

A companion guard is being added to `pcfweb/build.sh` to detect pointer files
in `images/` and fail the build. Until that lands, `git lfs pull` is the only
thing standing between a stale checkout and a broken production site.

### Scope: which files are affected today

Only files stored in LFS can turn into pointers, and today that is a **small
and growing** subset:

- `.gitattributes` tracks `*.jpg`, `*.jpeg`, `*.png`, `*.gif`, `*.webp`,
  `*.xcf` (and their uppercase spellings), so **every image added or modified
  from now on** goes into LFS.
- Images already committed before LFS was enabled remain ordinary git blobs.
  They were deliberately left alone — converting them would require rewriting
  history and force-pushing `main`, which breaks every existing clone.

So a build host that forgets `git lfs pull` currently ships most images fine
and a handful broken — which is *harder* to notice than a wholesale failure,
not easier. That set grows with every image touched.

Side effect of the mixed state: in a checkout whose index stat cache has gone
stale, `git status` may report every pre-LFS image as modified even though the
bytes on disk are unchanged. Git is comparing the stored raw blob against what
the LFS clean filter would now produce. **Do not "fix" this with `git commit -a`
or `git add -A`** — that would convert the entire back catalogue to LFS in one
unreviewed commit. A fresh clone does not show it.

### Storage and bandwidth cost

GitHub Free and Pro accounts include **10 GiB of LFS storage and 10 GiB of LFS
bandwidth** per billing cycle, billed as metered usage (the old pre-paid data
packs were retired); overage runs $0.07/GB stored and $0.0875/GiB downloaded.
Both are pooled per account owner, not per repository.
Source: [Git Large File Storage billing](https://docs.github.com/en/billing/concepts/product-billing/git-lfs)
and [About storage and bandwidth usage](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-storage-and-bandwidth-usage).

All of `images/` is ~47MB. Even if every file in it eventually lived in LFS,
that is under 0.5% of the 10 GiB storage allowance, and a full fetch would be
~47MB against a 10 GiB monthly bandwidth allowance — on the order of 200 full
fetches per month before anything is billable. Today the LFS set is a single
file under 1MB.

Bandwidth is also spent far less often than it first appears: the images are
baked into the Docker image at build time, so LFS is touched **once per build,
on the build host**. Deploys, pod restarts, replica scale-ups and rollbacks
pull the container image, not LFS. `originals/` is never copied into the
container at all.

Nothing here is close to the quota. This section exists so the correctness
problem above does not get mistaken for a cost problem — the risk of LFS in
this repository is silently broken images, not the bill.

## Layout

| Directory | Ships in the container | What lives here |
| --- | --- | --- |
| `images/` | **Yes** | Web-ready copies. This is the whole directory `build.sh` copies. |
| `originals/` | No | Full-resolution masters. Archive only. |

`build.sh` copies `../pcfweb-assets/images` and nothing else, so anything in
`originals/` is safe to keep at whatever size it was captured at — it never
reaches the image.

### Book covers

Book covers live in `images/book_covers/`, and their masters — where one
exists — in `originals/book_covers/`, under the same filename as the
`images/` copy. Only `distributed_computing_4_kids` has a master
(`.webp`, because that is genuinely what the master is); the four O'Reilly
covers ship at their native 2100x2756 and were never downscaled, so there is
nothing to archive for them.

pcfweb references covers by a path **relative to `images/`**, so a cover at
`images/book_covers/x.jpg` is referenced as `book_covers/x.jpg`.

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
