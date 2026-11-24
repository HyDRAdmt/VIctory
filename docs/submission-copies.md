# Submission copies on Windows

The two reported world frames imply 283 and 315 MB/s. The frame meter times
each shadow update, including dirty checks that copy zero bytes. It also
charges explicit title copies and fills to the same counter. These samples
give the total cost but cannot distinguish the operations. This investigation
used synthetic Windows mappings on the reporting PC. World FPS and the
relative contribution of each operation in the world remain unmeasured.

## Findings

* The old batch took the minimum start and maximum end of every refresh of
  one shadow. Two views at opposite ends of a heap could therefore request
  the whole heap. On unwatched memory the entire span was copied each flush.
  On watched memory even dirtied pages in the gap could be copied. The batch
  now updates the union of the requested intervals. Adjacent and overlapping
  intervals merge within the same shadow and flush. A title write or GPU
  write declaration still flushes pending work, as does the end of submission
  (`crates/victory-gpu/src/host.rs:510,1165,1265,1280`).
* A previously guarded, unchanged physical view used to take a failed
  `GetWriteWatch`, a `VirtualQuery` on the beginning of the interval, and a
  guard lookup even for a single 64 KiB check. Querying the start of a large
  uniform region is expensive. A range whose every block remains in the
  clean guard set can now use the journal directly. A CPU fault, a runtime
  write opening the block, or a remap invalidates this shortcut. Each shadow
  still compares against its own last copy, including a new shadow that has
  never copied the range (`watch.rs:51,132,206,233`, `shadow.rs:325`).
* Shadow updates already coalesced consecutive stale pages and excluded
  pending GPU writes. A partly refreshed page can remain stale and be copied
  in part again. Shadow storage is persistently mapped write-back memory.
  Explicit title copies and upload fills still use checked kernel reads, with
  a smaller-page fallback for holes (`shadow.rs:260,301,325`,
  `host.rs:1171,1312,1778`, all under `crates/victory-gpu/src`).

## Synthetic measurements

Release build, three runs per variant, median elapsed time. All inputs were
generated locally. The sparse test touches the destination before timing
and averages 16 flushes. It exercises the production checked read on
resident system memory. The guard test uses one pagefile section and
queries 128 different blocks near its beginning in rotation.

| Work | Previous | Updated |
| --- | ---: | ---: |
| Two 64 KiB views in one unwatched 64 MiB buffer, per flush | 64 MiB, 9.262 ms | 128 KiB, 0.0081 ms |
| 1,024 clean 64 KiB checks in one 174 MiB mapped view | 665.8 ms | 0.0923 ms |

The contiguous 64 MiB read itself sustained about 7.2 GB/s in this test.
The clean-view test moved zero bytes in both versions. Both results depend
on the distribution of views, writes, and holes.

```
cargo test --release -p victory-gpu sparse_refresh_copy_time -- --ignored --nocapture
cargo test --release -p victory-gpu clean_view_checks -- --ignored --nocapture
```

The second command measures the current implementation. Its baseline was
measured with the same test before changing the guard path. The diagnostics
build (`cargo build --profile diag -p victory --features diagnostics`)
now reports bytes of excluded gaps, bytes actually copied from unwatched
memory, clean-view checks, guard VM query time, and actual copied bytes per
source range. The gap counter measures bytes excluded from consideration.
Only stale bytes in those gaps would have contributed to watched copies.
The detailed counters are now normalized by every frame since their last
log (`copystat.rs:91`, `host.rs:1557`).

## Windows commit during loading

The 35,651,600-byte allocation failure identifies a roughly 34 MiB request.
Its size alone cannot identify an allocation site or explain the system's
42 GB commit total. The following estimates are allocation sizes from source
inspection. Lifetimes overlap and some title resources share backing.

| Category | Allocation and lifetime |
| --- | --- |
| Title physical pages | The observed 6.1 GiB request creates a reserved pagefile section on Windows. Mapping or writing commits the underlying pages once across aliases. Title virtual allocations can also request commitment. See `crates/victory-hle/src/modules/xmem.rs:113,401,735`. |
| Title heaps and resources | A heap allocates its full declared byte size. A committed resource allocates and zeroes its requested size through the title allocator or the Rust allocator. Placed resources can share existing heap memory. See `resource.rs:612,639,673`. |
| Host buffer shadows | Each used registered buffer gets a full-sized committed L0 write-back buffer rounded up to 64 KiB, even for a small view. Aliased buffer objects may each get a copy. Each shadow also tracks eight bytes per 64 KiB page. Loose shadows are retained globally by window start **and** length, rounded to 4 MiB. See `shadow.rs:105,400,445` and `host.rs:1778`. |
| GPU-write snapshots | One live snapshot per writing shadow spans the lowest to highest declared byte. Growing it temporarily keeps both buffers. The reuse pool retains up to eight capacities without a byte limit. Eight 174 MB snapshots would retain about 1.4 GB, conditional on that allocation history. See `shadow.rs:49,151,190`. |
| Uploads and textures | Per-list uploads live until reset. Scratch retains 4 MiB chunks at the list's high-water mark. Each staged texture gets a mapped upload resource sized by D3D12 footprints. A watched texture also retains a CPU vector sized to the full tiled layout. The host texture itself uses a default heap, whose size should be accounted separately from CPU private commit. See `host.rs:1070,1083`, `texture.rs:80,188,322` and `resource.rs:848`. |
| Shader and watch caches | Each distinct shader retains its input container and cached translation. The watch journal retains one timestamp entry per previously observed 64 KiB block and space for the largest kernel answer (one pointer per 4 KiB). Totals depend on unique shaders and address coverage. See `pipeline.rs:59,75` and `watch.rs:31,51`. |

GPU source references in the table are under `crates/victory-gpu/src`.

## Verification

* `cargo build --release -p victory` exited successfully. Its last line
  was ``Finished `release` profile [optimized + debuginfo] target(s) in 11.66s``.
* `cargo test --workspace` exited successfully with 265 passed, zero failed
  and six ignored. The tail was `Doc-tests victory_shader`, then
  `test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s`.
* The release diagnostics GPU suite passed with 46 passed and three ignored
  using `--test-threads=1`. An earlier concurrent run during workspace
  compilation missed a tick in the existing frame-clock timing test.
