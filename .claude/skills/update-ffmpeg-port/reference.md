# Reference: what this registry adds to upstream ffmpeg

Read this before resolving a conflict or hand-editing anything. Everything here
was derived from the last clean round (`master` on top of its subtree merge);
regenerate it any time with `scripts/resolume-delta.sh`.

## Branch shape

| branch | contents |
| --- | --- |
| `master` | previous round: subtree merge + one `resolume customizations` commit |
| `<label>-ffmpeg-port` | this round, upstream `ports/ffmpeg` only, nothing resolume |
| `<label>-ffmpeg-update` | this round, the PR branch: port branch + the resolume commit |
| `resolume-split/<label>` | in the *vcpkg* clone: `ports/ffmpeg` history split out |

`<label>` is the vcpkg date tag (`2026.06.01`) or the short commit sha.
`master^1` (the README's wording) is the previous subtree merge; the scripts find
it as the last merge on the first-parent chain, which is the same commit.

## The resolume delta

### `ports/ffmpeg/portfile.cmake` - four hunks, nothing else

1. The resolume patches, appended to the `PATCHES` list of `vcpkg_from_github`,
   after upstream's, separated by a blank line:

   ```cmake
           # Resolume patches
           1000-resolume-small-memory-allocations.patch
           1001-max_chunk_size-from-format-context-in-build_chunks.patch
           1003-avformat_index_get_entry_const_correctness.patch
           1004-h264-videotoolbox-arm64-bframe-size.patch
   ```

2. Inside `if(VCPKG_TARGET_IS_OSX)`, before the `VCPKG_OSX_ARCHITECTURES` lines:

   ```cmake
       # we want our libs with @rpath for development
       string(APPEND OPTIONS " --install_name_dir=@rpath")
   ```

3. At the very end of the file, after `vcpkg_install_copyright(...)`:

   ```cmake
   # Disable fixup of rpaths by vcpkg
   set(VCPKG_FIXUP_MACHO_RPATH OFF)
   ```

4. At the top of upstream's `if ("vaapi" IN_LIST FEATURES)` block, before its
   `--enable-vaapi`: the pkg-config probe for the system `libva` + `libva-drm`
   that fails with an `apt install libva-dev` hint. VAAPI must use the system
   libva: a vcpkg-built one looks for drivers in its own build tree, so every
   VAAPI open fails at runtime.

### `ports/ffmpeg/vcpkg.json` - version fields only

`version` is upstream's, `port-version` is upstream's + 100 (upstream `1` -> 101,
upstream `2` -> 102). The only other line that is ours is the `vaapi` feature:
it has no `dependencies` (upstream's depends on the vcpkg `libva` port) and its
description says it uses the system libva. Keep that when taking upstream's
manifest. `finish-update.sh` rewrites only the two version lines.

### The patches themselves

| patch | what it does |
| --- | --- |
| `1000-resolume-small-memory-allocations.patch` | small-allocation handling |
| `1001-max_chunk_size-from-format-context-in-build_chunks.patch` | take max chunk size from the format context in the mov writer |
| `1003-avformat_index_get_entry_const_correctness.patch` | const correctness on index entries |
| `1004-h264-videotoolbox-arm64-bframe-size.patch` | b-frame DTS offset of 2 on arm64 for H264 |

There is no `1002`; the gap is deliberate, do not renumber. These are the only
files in `ports/ffmpeg` this repo owns. `0xxx` patches are upstream's - never
touch them, and do not delete an unused one (upstream leaves e.g.
`0042-fix-arm64-linux.patch` behind unlisted; `verify.sh` only warns about it).

A new ffmpeg release usually moves the context of the `1xxx` patches, so they
need refreshing - `1001` was refreshed for 8.1.1. `check-patches.sh` catches it.

### `versions/`

* `versions/f-/ffmpeg.json` - exactly one entry, the version being published:
  `{"git-tree", "version", "port-version"}`, 4-space indent, no trailing newline.
  `git-tree` is `git rev-parse HEAD:ports/ffmpeg` of the finished commit.
* `versions/baseline.json` - `baseline` = version, `port-version` = port-version
  of that entry.

The versions this registry shipped itself are not kept. Consumers point their
`vcpkg-configuration.json` at a branch or tag of this repo and always resolve the
newest, and those entries do not survive re-pointing that ref, so naming them
buys nothing. The trade-off: an exact pin to an old resolume port-version no
longer resolves. `verify.sh` fails the round if the file holds more than one
entry.

Both are folded into the resolume commit with `git commit --amend`. Amending
cannot change the recorded `git-tree`, because `versions/` lives outside
`ports/ffmpeg` - that is why the README's order (git-tree first, then version
files, then amend) is safe.

### Commits that upstream has made redundant

A resolume commit whose change upstream has since absorbed shrinks, after the
rebase, to a port-version bump and nothing else. `finish-update.sh` drops those:
it rebuilds each replayed commit with its bookkeeping neutralised (`versions/` as
it is on the port branch, the manifest's version fields set to what this round
publishes, the manifest's trailing newline restored) and skips any whose tree then
matches its parent's. Authors, dates and messages of the survivors are preserved.

The 2026.07.29 round is the worked example: `ffmpeg: add vaapi feature` and
`ffmpeg: support GNU-driver clang targeting windows-msvc` both went that way -
upstream took the clang work as vcpkg #52810 and ffmpeg 8.1.2 took the configure
half, while upstream's own `vaapi` feature (which depends on the vcpkg `libva`
port) replaced ours (which probed for system libva). That replacement broke
VAAPI at runtime (see hunk 4), so the system-libva variant was restored on top.

Commits on `master` that are not part of the port delta (this skill, README
edits) get replayed along with the resolume commit. That is harmless - vcpkg
resolves a version by its `git-tree`, wherever the version files are committed -
but keep `resolume customizations` as master's tip so it stays the commit that
carries both the delta and the version files. `finish-update.sh` warns when the
tip is something else.

## Resolving the rebase conflicts

The rebase replays the resolume commit onto the new upstream, so:

* **ours** = upstream (`<label>-ffmpeg-port`), **theirs** = resolume commit.

Do not merge the two sides line by line. Take upstream and re-apply the small
delta:

```bash
F=ports/ffmpeg/portfile.cmake
scripts/resolume-delta.sh -o /tmp/resolume-delta.patch   # previous round's delta
git checkout --ours "$F" && git add "$F"                 # take upstream, stage it
git apply -3 --include="$F" /tmp/resolume-delta.patch    # re-apply the resolume hunks
git add "$F"                                             # after fixing any markers
```

The `git add` before `git apply -3` matters: the three-way apply reads the
staged (upstream) file as its "ours" side, and a conflicted path has nothing
staged.

`git apply -3` usually places two of the three hunks silently and leaves a small
conflict in the `PATCHES` list, because upstream adds and drops patches there
every release. Keep upstream's list exactly as it is and append the resolume
block after it - that is the whole resolution. Where a hunk cannot be placed at
all, add it by hand from the three hunks listed above.

For `ports/ffmpeg/vcpkg.json`, always take upstream and let `finish-update.sh`
set the two version fields:

```bash
git checkout --ours ports/ffmpeg/vcpkg.json && git add ports/ffmpeg/vcpkg.json
```

Then `git rebase --continue` and run `scripts/verify.sh`. Its
"only portfile.cmake, vcpkg.json and 1xxx patches differ from upstream" check is
what proves the resolution kept upstream intact.

## How the upstream history is extended

`git subtree split` walks every commit in vcpkg's history and shells out two or
three times per commit, so on a ~30k-commit vcpkg it runs for hours. It is also
mostly wasted work: this registry already holds the split history up to
`master^1^2`, and only the handful of commits after it are missing.

`split-upstream.sh` therefore copies just those commits. For each upstream commit
that touches `ports/ffmpeg` (in `git rev-list --reverse --topo-order
<boundary>..<ref> -- ports/ffmpeg`) it writes a commit whose tree is that
commit's `ports/ffmpeg` tree, whose parent is the previous copy, and whose
author, committer, both dates and message are copied verbatim - which is exactly
what git-subtree's `copy_commit` does. The result is byte-identical: replaying
the 22 commits between the 7.1.1 and 8.1.1 rounds reproduces `d257160da`, the
sha `git subtree split` produced for that state.

Two details matter if you ever touch this code:

* the message must come from `--pretty=format:%B`, not `--format=%B` - the
  latter appends a newline and changes every sha;
* `--no-show-signature` keeps signature output out of the message.

The boundary is found by tree identity: the newest upstream commit whose
`ports/ffmpeg` tree equals the merged split tip's tree. Because the copies
descend from that tip, `git subtree pull` fast-forwards instead of conflicting.
`split-upstream.sh` still asserts the ancestry and the final tree, and stops if
either does not hold - that means the requested ref is older than what is already
merged, or the histories are unrelated.

`--full-split` runs the real `git subtree split` instead. It yields the same
shas, takes hours, and exists only for cross-checking.

## Troubleshooting

| symptom | cause / fix |
| --- | --- |
| `split-upstream.sh` exits 3 | requested ref adds nothing new - nothing to do |
| "cannot find the upstream commit matching what master already merged" | the ref is from an unrelated history, or `master`'s port content is not upstream's; needs a human |
| "does not contain the upstream ffmpeg state already merged" | ref is older than `master`'s; pick a newer tag |
| `subtree pull` conflicts | the port branch was not started from the previous subtree merge; re-run `start-update.sh --force` |
| huge `portfile.cmake` conflict | expected once the file was reformatted in a past round; resolve with the recipe above and keep upstream formatting |
| `verify.sh`: "the port looks reformatted" | whitespace-only churn crept in; re-do the resolution from upstream |
| `check-patches.sh`: fuzz or FAIL on a `1xxx` patch | refresh it with the recipe the script prints, `git commit --amend`, re-run `finish-update.sh` |
| version files look wrong | never hand-edit; re-run `finish-update.sh` (idempotent) |
| a replayed commit is left with only a port-version bump | expected once upstream absorbs it; `finish-update.sh` drops it |

`vcpkg x-add-version` is the upstream way to write the version files, but it
wants a full vcpkg checkout; `update_versions.py` does the same bookkeeping for
this registry and needs nothing but python3.

## Past rounds

Only the current version is published, so this is history, not state: 8.1.2#103
(upstream 3), 8.1.1#102 (2), 7.1.1#101-103 (1), 7.1#102 (2), 6.1.1#100-101,
6.1#100-102. The +100 offset has held throughout.
