# GPU memory and performance audit

Date: 2026-11-25. Scope: static inspection of the repository's open-source platform code and documentation. No proprietary executable, assets or extracted material used. No Windows GPU execution, build, or independent benchmark was performed. The baseline is the main branch at the time this audit was created.

## Baseline from README (reported by the fork author, not independently verified)

- RX 9070 XT 16 GB, Windows 11, 64 GB system RAM, Ryzen 7 9800X3D.
- Menus: 30 FPS; world: 3–5 FPS; approximately 22 GB RAM and video memory demand exceeding card capacity; approximately 33 GB process commit after 15 minutes.
- A display driver timeout was reported. Some graphics pipelines are unsupported. The README reports substantial CPU-bound work even when asynchronous submission is enabled.
- The documentation describes separate descriptor-heap compatibility issues on NVIDIA; VRAM capacity alone cannot explain these.

## Architecture and source map

| Path under crates/victory-gpu/src | Role |
| --- | --- |
| host.rs | D3D12 host device, DXGI memory-budget query, resource retirement |
| resource.rs | Host resource creation, texture accounting, mip selection |
| texture.rs | Texture upload and staging cache |
| texpool.rs | Shared allocations for small textures and upload buffers |
| shadow.rs | Host copies of buffers, sparse mappings, snapshot caches |
| memory.rs | Address-to-view mapping and resource ownership |
| memstat.rs | Allocation counters and memory reporting |
| descriptor/host.rs | Descriptor heaps and host view bookkeeping (not yet reviewed in full) |

Other relevant documentation: docs/architecture.md, docs/view-heap-limit.md, docs/asset-policy.md. Keep asset policy intact.

## Findings and investigation queue

1. **Budget is not the same as dedicated capacity.** host.rs:248–270 queries current usage and DXGI budget in both local and non-local segments. No fixed 16 GiB launch restriction was established by the inspected paths. A full repository search and actual error evidence are still required before ruling out other checks.

2. **Memory adaptation is already present.** resource.rs:1127–1185 uses cached local budget pressure to select a texture size cap. It is restricted to eligible streamed compressed 2D textures with mip chains and no target/write flags. The host copy is resized at creation time; no proof was found that existing allocations are resized in response to later budget changes. Do not equate this policy with total graphics memory control.

3. **Staging accounting is not physical allocation accounting.** texture.rs:83–155,193–205 and 616–690 distinguish logical cached bytes from actual upload allocation lifetimes. Cache removal reduces STAGED immediately, whereas GPU references and retirement may keep backing allocations alive. The default logical cache budget is 384 MiB. In-flight references can prevent eviction, so it is a soft target, not a hard maximum. Measure cache occupancy separately from actual live upload allocations; do not force early release of resources held by a GPU submission.

4. **Potentially incomplete snapshot telemetry.** shadow.rs has separate general and tile snapshot pools. Both have substantial retention limits. The inspected pool_bytes function reports only the general pool; memstat.rs uses this function in its snapshot line. This is a concrete accounting gap, not evidence that the pools leak. A synthetic test should verify the accounting of both pools, and a subsequent diagnostic change should expose their capacities independently.

5. **Resource pools need separate logical and physical metrics.** texpool.rs groups small allocations into slabs; unused slots in a partially occupied slab do not necessarily release the heap. Report reserved heap bytes, occupied slot bytes and retirement separately. Some counters describe overlapping allocations, so adding all categories together is not a valid estimate of total VRAM.

6. **Potential retained views.** memory.rs:27–155 holds resources in a vector of views. Remapping removes window indices and updates the address map; the observed path does not remove their entries from that vector. Investigate whether retained resources can grow during remapping. This is an ownership/lifetime hypothesis, not a confirmed leak. Descriptor and GPU fence lifetime constraints must be understood before considering changes.

7. **Driver-specific limits.** docs/view-heap-limit.md describes view and sampler descriptor constraints and distinguishes root-table remapping from direct heap indexing. It reports NVIDIA failures even on a higher-capacity GPU. This is a separate compatibility axis from memory pressure.

8. **Performance is not solely a VRAM problem.** The README reports CPU-side copying and submission work as well as GPU indirect-command costs. Lower memory demand does not guarantee playable performance. Frame timing and paging measurements must be separated from resource counts.

## Validation plan (synthetic / authorized test workloads)

- Add unit tests for memory accounting and pressure classification; verify empty pools, populated pools, release and repeated cycles.
- Exercise resource creation, retirement and remapping with the existing synthetic fixtures and Windows WARP tests where applicable. Verify no invalid resource access.
- Capture timestamped local/non-local DXGI usage and budget, process commit, live buffers, logical cache bytes, pool capacity, and frame timings. Compare steady-state and peak values; avoid summing overlapping counters.
- Use a controlled synthetic D3D12 workload on 8, 12 and 16 GiB adapters if available. Hold workload, driver, settings and scene constant. Record failure HRESULT and adapter feature support.
- Run cargo test --workspace and cargo xtask policy before merging a code change. No test result is claimed by this audit.

## Next decision

Prioritize instrumentation correctness and independent measurements before changing allocation lifetime, descriptor mapping or shader translation. No guarantee of lower VRAM support or frame-rate improvement is made. The main branch is not changed by this report.
