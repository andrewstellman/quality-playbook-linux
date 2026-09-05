# NVMe Host Driver CVEs and Security Advisories

> ## ⚠ TIER-4 BACKGROUND CONTEXT — NOT AN AUTHORITATIVE CONTRACT
>
> **Read this before using anything below.**
>
> 1. **This file is background context, not a specification.** It lives at the top level of
>    `reference_docs/` and must **never** be moved into `reference_docs/cite/`. Nothing in this
>    document is a spec-conformance requirement, a normative "shall" statement, or a citable
>    contract. The authoritative contracts for this driver are the NVMe base and transport
>    specifications in `reference_docs/cite/`.
> 2. **Every issue listed here is HISTORICAL.** Each record states the kernel version in which the
>    bug was introduced and the version in which it was fixed. All of them were fixed upstream.
>    None of them is described here as a live defect.
> 3. **The presence of a CVE in this file is NOT evidence that the current code is defective.**
>    A CVE record describes a bug that existed in a *past* version of a file. Finding
>    `drivers/nvme/host/tcp.c` in this list tells you a bug once lived there; it tells you nothing
>    about the tree under audit. Do not restate a fixed issue as if it were a current finding.
> 4. **Do not cite these records as spec-conformance requirements.** An audit finding of the form
>    "this violates CVE-YYYY-NNNNN" is malformed. Treating an advisory as an authoritative contract
>    has previously caused an audit to score 0/3 on precision.
> 5. **Legitimate use.** Use this file for *orientation only*: to learn which code paths have
>    historically been fragile, what classes of bug recur in this driver, and — most usefully — the
>    behavioural invariants in the closing `## Invariants` section. Those invariants are distilled
>    observations about how the driver is *supposed* to behave, useful as review heuristics. They
>    are heuristics, not contracts.

---

## Sources

All records below were actually retrieved during this pass. URLs are the real endpoints used.

- **Linux kernel CNA published-CVE tree (primary source for every record here)** —
  `git clone https://git.kernel.org/pub/scm/linux/security/vulns.git`, then the
  `cve/published/<year>/CVE-*.json` and `cve/published/<year>/CVE-*.mbox` files.
  Browsable at <https://git.kernel.org/pub/scm/linux/security/vulns.git/>.
  Clone contained 15,192 published CVE records; 55 of them name a file under
  `drivers/nvme/host/` or reach it via an nvme call path.
- **NVD JSON API** — <https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=nvme>
  (223 results) and <https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2025-21927>.
  Used for CVSS v3.1 base scores and NVD publication dates.
- **Ubuntu Security Tracker JSON** — `https://ubuntu.com/security/cves/CVE-YYYY-NNNNN.json`
  (e.g. <https://ubuntu.com/security/cves/CVE-2025-21927.json>). Retrieved for all 55 CVEs;
  used for Ubuntu priority ratings.
- **Red Hat Security Data API** — `https://access.redhat.com/hydra/rest/securitydata/cve/CVE-YYYY-NNNNN.json`
  (e.g. <https://access.redhat.com/hydra/rest/securitydata/cve/CVE-2025-21927.json>).
  Retrieved for all 55 CVEs; used for Red Hat threat severity.
- **Debian Security Tracker** — <https://security-tracker.debian.org/tracker/CVE-2025-21927>
  (spot-check corroboration of description text).
- **CVE Program / MITRE CVE Services API** — <https://cveawg.mitre.org/api/cve/CVE-2026-74384>
  (spot-check corroboration of the CNA record, `programFiles`, and publication date).
- **git.kernel.org stable commit references**, as cited in each CNA record, e.g.
  <https://git.kernel.org/stable/c/ad95bab0cd28ed77c2c0d0b6e76e03e031391064>.

### Sources attempted that did NOT work

- **lore.kernel.org linux-cve-announce search** —
  <https://lore.kernel.org/linux-cve-announce/?q=nvme> returned HTTP 200 but served an Anubis
  anti-bot JavaScript proof-of-work challenge instead of the archive. No data obtained. The same
  content was recovered from the vulns.git `.mbox` files, which are the identical announcement
  emails, so nothing was lost.
- **SUSE CVE pages** — <https://www.suse.com/security/cve/CVE-2026-74384.html> returned HTTP 200
  but the body is a JavaScript-rendered shell with no CVE data in the served HTML. Not used.
- **Ubuntu bulk search endpoint** — `https://ubuntu.com/security/cves.json?q=nvme&limit=50`
  returned HTTP 422 (the API caps `limit` at 20). Also `https://ubuntu.com/security/CVE-XXXX.json`
  is a 404; the correct path is `/security/cves/CVE-XXXX.json`, which is what was used.
- **Red Hat bulk query** — `https://access.redhat.com/hydra/rest/securitydata/cve.json?package=kernel&cve=...`
  returned HTTP 400 `"Found unpermitted parameter : cve"`. Per-CVE endpoint used instead.

### Scope and caveats

- 53 of the 55 records name a file under `drivers/nvme/host/` in the CNA `programFiles` field.
- 2 records matched because `drivers/nvme/host` appears in the crash trace, but the **fix is in
  another subsystem**. They are listed separately under "Adjacent" and are explicitly flagged as
  *not* nvme-host fixes.
- Year range of records: **2021 – 2026**.
- "Fixed in" below is the **mainline** version. Every CVE also has stable-tree backports at
  earlier point releases; those are listed in the per-CVE JSON in vulns.git and are omitted here
  for brevity except where noted.
- No CVE in the retrieved set names `drivers/nvme/host/zns.c` or `drivers/nvme/host/auth.c`
  directly. The DH-CHAP secret leaks (CVE-2023-53792, CVE-2023-53852) are in `core.c`, in the
  sysfs `_store` handlers, not in `auth.c`.

### Tree-vintage note (unverified inference, not a finding)

Spot-checks of the tree under audit show it already contains fixes that landed in mainline 7.1–7.2
(`dev->hmb_sgt = NULL` after free in `pci.c`; `nr_node_ids` sizing in `core.c`;
`int numa_node` in `nvme_setup_descriptor_pools`; `kvzalloc` in `pr.c::nvme_pr_read_keys`).
That suggests the tree is roughly 7.2-era or later, i.e. **the overwhelming majority of the CVEs
below are already fixed in it**. This is an inference from four greps, not a version assertion —
do not rely on it, and do not treat any individual CVE as "still present" without reading the code.

---

## Bug density by area

| Area | File | CVEs in this set |
|---|---|---|
| PCI transport | `pci.c` | 13 |
| Core | `core.c` | 13 |
| TCP transport | `tcp.c` | 9 |
| Fibre Channel | `fc.c` | 5 |
| RDMA transport | `rdma.c` | 3 |
| Multipath | `multipath.c` | 3 |
| Apple | `apple.c` | 2 |
| Fabrics | `fabrics.c` / `fabrics.h` | 2 (one shared with `core.c`) |
| Header | `nvme.h` | 2 (one shared with `multipath.c`) |
| Persistent reservations | `pr.c` | 1 |
| ioctl | `ioctl.c` | 1 |
| sysfs | `sysfs.c` | 1 |
| ZNS | `zns.c` | 0 |
| Auth | `auth.c` | 0 |

---

## core — `drivers/nvme/host/core.c`

### CVE-2022-48790 — nvme: fix a possible use-after-free in controller reset during load
- **Description (as published):** Unlike `.queue_rq`, in `.submit_async_event` drivers may not check the ctrl readiness for AER submission. This may lead to a use-after-free condition that was observed with nvme-tcp.
- **Files/functions:** `core.c`, `nvme_async_event_work()` / `.submit_async_event` path.
- **Bug class:** Use-after-free (race condition).
- **Introduced:** 4.15 (`ad22c355b707a8d8d48e282aadc01c0b0604b2e9`). **Fixed:** mainline **5.17** (`0fa0f99fc84e41057cbdd2efbfe91c6b2f47dd9d`); stable 4.19.231, 5.4.181, 5.10.102, 5.15.25, 5.16.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD CVSS 3.1 **9.8**.
- **Invariant violated:** *An AER (or any admin command) must not be submitted once the controller state has left LIVE/CONNECTING — the state check must happen in the submit path itself, not only in the caller that schedules it.*

### CVE-2022-49003 — nvme: fix SRCU protection of nvme_ns_head list
- **Description:** Walking the `nvme_ns_head` siblings list is protected by the head's SRCU in `nvme_ns_head_submit_bio()` but not `nvme_mpath_revalidate_paths()`. Removing namespaces from the list also fails to synchronize the SRCU. Concurrent scan work can therefore cause use-after-frees.
- **Files/functions:** `core.c` (`nvme_ns_remove()`), `multipath.c` (`nvme_mpath_revalidate_paths()`).
- **Bug class:** Use-after-free (missing SRCU protection).
- **Introduced:** 5.15 (`e7d65803e2bb5bc739548b67a5fc72c626cf7e3b`). **Fixed:** mainline **6.1** (`899d2a05dc14733cfba6224083c6b0dd5a738590`); stable 5.15.82, 6.0.12.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *Every traversal of an `nvme_ns_head` sibling list must be inside that head's SRCU read-side critical section, and removal from the list must synchronize with that same SRCU — not with global RCU.*

### CVE-2023-53670 — nvme-core: fix dev_pm_qos memleak
- **Description:** Call `dev_pm_qos_hide_latency_tolerance()` in the error unwind path to avoid a kmemleak from `nvme_init_ctrl()`.
- **Files/functions:** `core.c`, `nvme_init_ctrl()` error unwind.
- **Bug class:** Memory leak.
- **Introduced:** 6.0 (`f50fff73d620cd6e8f48bc58d4f1c944615a3fea`). **Fixed:** mainline **6.5** (`7ed5cf8e6d9bfb6a78d0471317edff14f0f2b4dd`); stable 6.1.39, 6.3.13, 6.4.4.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *Every error-unwind path in controller init must release everything the successful path would have owned, in reverse order of acquisition.*

### CVE-2023-53792 — nvme-core: fix memory leak in dhchap_ctrl_secret
- **Description:** Free `dhchap_secret` in `nvme_ctrl_dhchap_ctrl_secret_store()` before returning when `nvme_auth_generate_key()` returns an error.
- **Files/functions:** `core.c`, `nvme_ctrl_dhchap_ctrl_secret_store()`.
- **Bug class:** Memory leak.
- **Introduced:** 6.0 (`f50fff73d620cd6e8f48bc58d4f1c944615a3fea`). **Fixed:** mainline **6.5** (`99c2dcc8ffc24e210a3aa05c204d92f3ef460b05`); stable 6.1.39, 6.3.13, 6.4.4.
- **Severity:** Red Hat Low; Ubuntu medium; Ubuntu CVSS 5.5 (no NVD score retrieved).
- **Invariant violated:** *A sysfs `_store` handler that allocates a buffer must free it on every path that does not hand ownership to the controller.*

### CVE-2023-53852 — nvme-core: fix memory leak in dhchap_secret_store
- **Description:** Free `dhchap_secret` in `nvme_ctrl_dhchap_secret_store()` before returning (kmemleak trace included in the advisory).
- **Files/functions:** `core.c`, `nvme_ctrl_dhchap_secret_store()`.
- **Bug class:** Memory leak.
- **Introduced:** 6.0 (`f50fff73d620cd6e8f48bc58d4f1c944615a3fea`). **Fixed:** mainline **6.5** (`a836ca33c5b07d34dd5347af9f64d25651d12674`); stable 6.1.39, 6.3.13, 6.4.4.
- **Severity:** Red Hat Low; Ubuntu medium; Ubuntu CVSS 5.5.
- **Invariant violated:** Same as CVE-2023-53792 — *allocation ownership in a `_store` handler must be resolved on every exit path.*

### CVE-2024-27435 — nvme: fix reconnection fail due to reserved tag allocation
- **Description:** ABBA deadlock via tag allocation: a keep-alive request holds the single reserved admin tag while `admin_q` is quiesced during reset, so the connect command for reconnect can never get a tag and `admin_q` reconnect fails forever.
- **Files/functions:** `core.c`, `fabrics.h` (`NVMF_RESERVED_TAGS`).
- **Bug class:** Deadlock (resource starvation / ABBA on tag allocation).
- **Introduced:** 5.12 (`ed01fee283a067c72b2d6500046080dbc1bb9dae`). **Fixed:** mainline **6.9** (`de105068fead55ed5c07ade75e9c8e7f86a00d1d`); stable 6.1.83, 6.6.23, 6.7.11, 6.8.2.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.5.
- **Invariant violated:** *A reserved tag needed to make reconnect progress must never be consumable by a command that can itself block until reconnect completes. The admin queue must reserve enough tags that connect can always be issued.*

### CVE-2024-41073 — nvme: avoid double free special payload
- **Description:** If a discard request needs to be retried, and that retry may fail before a new special payload is added, a double free results. Clear `RQF_SPECIAL_PAYLOAD` when the request is cleaned.
- **Files/functions:** `core.c`, request cleanup / discard retry path.
- **Bug class:** Double-free.
- **Introduced:** 4.10 (`f9d03f96b988002027d4b28ea1b7a24729a4c9b5`). **Fixed:** mainline **6.10** (`e5d574ab37f5f2e7937405613d9b1a724811e5ad`); stable 5.10.237, 5.15.164, 6.1.101, 6.6.42, 6.9.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *A request's special payload must be freed exactly once. The flag that says "this request owns a special payload" must be cleared at cleanup so a retry cannot free it a second time.*

### CVE-2024-45013 — nvme: move stopping keep-alive into nvme_uninit_ctrl()
- **Description:** Keep-alive is started in `nvme_init_ctrl_finish()` but was not stopped in `nvme_uninit_ctrl()`, so keep-alive work could stay pending after a failed controller start; unloading the host driver then triggered a use-after-free.
- **Files/functions:** `core.c`, `nvme_uninit_ctrl()` / `nvme_keep_alive_work()`.
- **Bug class:** Use-after-free.
- **Introduced:** 6.7 (`3af755a46881c32fecaecfdeaf3a8f0a869deca5`). **Fixed:** mainline **6.11** (`a54a93d0e3599b05856971734e15418ac551a14c`); stable 6.10.7.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *Every periodic work item started during controller init must be stopped on the matching uninit path, including when init failed partway through.*
- **Note:** this fix itself regressed — see CVE-2024-53169.

### CVE-2024-53169 — nvme-fabrics: fix kernel crash while shutting down controller
- **Description:** A keep-alive request can sneak in while a fabric controller is shutting down, racing the admin queue destroy path. `nvme_keep_alive_end_io()` drops `admin->q_usage_counter` to zero, letting `blk_mq_destroy_queue()` proceed on another CPU while the keep-alive dispatcher is still touching admin queue resources. Explicitly a regression from CVE-2024-45013's fix (`a54a93d0e359`).
- **Files/functions:** `core.c`, `nvme_stop_keep_alive()` moved into `nvme_remove_admin_tag_set()`.
- **Bug class:** Use-after-free (race condition).
- **Introduced:** 6.11 (`a54a93d0e3599b05856971734e15418ac551a14c` / `4101af98ab573554c4225e328d506fec2a74bc54`). **Fixed:** mainline **6.13** (`e9869c85c81168a1275f909d5972a3fc435304be`); stable 6.11.11, 6.12.2.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *Keep-alive must be stopped (in-flight completed or cancelled) **before** the admin queue is destroyed and its tagset removed. No work item may dispatch to a queue whose teardown has begun.*

### CVE-2025-68265 — nvme: fix admin request_queue lifetime
- **Description:** Namespaces can access the controller's admin `request_queue`, and stale namespace references may exist after controller teardown. The controller `put` was moved to after all controller references are released. Fixes a KASAN slab-use-after-free in `blk_queue_enter()` reached via `nvme_submit_user_cmd()` from an ioctl.
- **Files/functions:** `core.c`, controller/namespace refcount teardown ordering.
- **Bug class:** Use-after-free (lifetime/refcount ordering).
- **Introduced:** 6.1 (`fe60e8c534118a288cd251a59d747cbf5c03e160`). **Fixed:** mainline **6.18** (`03b3bcd319b3ab5182bc9aaa0421351572c78ac0`); stable 6.1.168, 6.6.120, 6.12.62, 6.17.12.
- **Severity:** Red Hat Moderate; Ubuntu low; NVD 7.8.
- **Invariant violated:** *The controller's admin `request_queue` must outlive every namespace that can reach it. The controller's final `put` must happen only after all controller references are released.*
- **Note:** this fix itself regressed — see CVE-2026-23360.

### CVE-2026-23360 — nvme: fix admin queue leak on controller reset
- **Description:** When `nvme_alloc_admin_tag_set()` is called during a controller reset, a previous admin queue may still exist; release it before allocating a new one. Explicitly a regression from CVE-2025-68265's fix (`03b3bcd319b3`).
- **Files/functions:** `core.c`, `nvme_alloc_admin_tag_set()`.
- **Bug class:** Memory leak (resource orphaning).
- **Introduced:** 6.18 (`03b3bcd319b3ab5182bc9aaa0421351572c78ac0` and siblings). **Fixed:** mainline **7.0** (`b84bb7bd913d8ca2f976ee6faf4a174f91c02b8d`); stable 6.1.168, 6.6.131, 6.12.77, 6.18.17, 6.19.7.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *A resource must never be reallocated over a live pointer. If reset can re-enter an allocator, the allocator must release the previous instance first.*

### CVE-2026-74361 — nvme: fix FDP fdpcidx bounds check
- **Description:** The `fdpcidx` bounds check sets `n = NUMFDPC + 1` but used `>` instead of `>=`, incorrectly accepting `fdp_idx` when it equals `n`.
- **Files/functions:** `core.c`, FDP configuration parsing (`nvme_query_fdp_info()` area).
- **Bug class:** Off-by-one / missing bounds check (out-of-bounds read).
- **Introduced:** 6.16 (`30b5f20bb2ddab013035399e5c7e6577da49320a`). **Fixed:** mainline **7.2** (`0967074f6830718fd2597404ef119bddd0dbfd00`); stable 6.18.40, 7.1.5.
- **Severity:** Red Hat Moderate; Ubuntu low; NVD 9.8.
- **Invariant violated:** *Every index derived from controller-reported identify/log data must be range-checked with the correct inclusive/exclusive comparison before it is used to index anything.*

### CVE-2026-74384 — nvme-multipath: fix flex array size in struct nvme_ns_head
- **Description:** `struct nvme_ns_head`'s flexible array `current_path[]` is indexed by `numa_node_id()` but was allocated with `num_possible_nodes()` entries. On architectures with sparse NUMA node IDs (observed on powerpc: nodes 0, 8, 252–255 with `num_possible_nodes() == 6`), indexing goes out of bounds. KASAN slab-out-of-bounds write in `nvme_mpath_revalidate_paths()`.
- **Files/functions:** `core.c`, `nvme_alloc_ns_head()` sizing of `struct nvme_ns_head`.
- **Bug class:** Out-of-bounds write.
- **Introduced:** 4.20 (`f333444708f82c4a4d3ccac004da0bfd9cfdfa42`). **Fixed:** mainline **7.2** (`001e57554de81aa79c25c18fd53911d8a415c304`); stable 5.10.261, 5.15.212, 6.1.178, 6.6.145, 6.12.97, 6.18.40, 7.1.5.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *An array indexed by an identifier must be sized by that identifier's maximum value + 1 (`nr_node_ids`), never by the count of valid identifiers (`num_possible_nodes()`). Identifier spaces are not required to be dense.*

---

## pci — `drivers/nvme/host/pci.c`

### CVE-2022-49492 — nvme-pci: fix a NULL pointer dereference in nvme_alloc_admin_tags
- **Description:** `admin_q` can be set to an ERR_PTR if `blk_mq_init_queue()` fails. The teardown path (`nvme_dev_disable()` → `nvme_suspend_queue()`) only checked for non-NULL, then quiesced a queue that never existed.
- **Bug class:** NULL/ERR_PTR pointer dereference.
- **Introduced:** 3.19 (`35b489d32fcc37e8735f41aa794b24cf9d1e74f5`). **Fixed:** mainline **5.19** (`da42761181627e9bdc37d18368b827948a583929`); stable 4.9.318, 4.14.283, 4.19.247, 5.4.198, 5.10.121, 5.15.46, 5.17.14, 5.18.3.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *A pointer that an allocator may set to an `ERR_PTR` must be validated with `IS_ERR_OR_NULL()` before any teardown path dereferences it — a non-NULL check is not sufficient.*

### CVE-2022-50756 — nvme-pci: fix mempool alloc size
- **Description:** The max transfer size was not converted to bytes to match the divisor computing worst-case PRP entries. The code rounded to 1 PRP list where 2 could be required, corrupting memory beyond the mempool size (observed by KFENCE).
- **Bug class:** Out-of-bounds write (undersized allocation).
- **Introduced:** 4.18 (`943e942e6266f22babee5efeb00f8f672fbff5bd`). **Fixed:** mainline **6.2** (`c89a529e823d51dd23c7ec0c047c7a454a428541`); stable 5.10.163, 5.15.87, 6.0.17, 6.1.3.
- **Severity:** Red Hat Moderate; Ubuntu high; NVD 7.8.
- **Invariant violated:** *A descriptor pool must be sized for the worst case number of entries, with numerator and divisor in consistent units.*

### CVE-2024-42276 — nvme-pci: add missing condition check for existence of mapped data
- **Description:** `nvme_map_data()` is called when a request has physical segments; `nvme_unmap_data()` lacked the same condition.
- **Bug class:** NULL pointer dereference (map/unmap asymmetry).
- **Introduced:** 5.2 (`4aedb705437f6f98b45f45c394e6803ca67abd33`). **Fixed:** mainline **6.11** (`c31fad1470389666ac7169fe43aa65bf5b7e2cfd`); stable 5.4.282, 5.10.224, 5.15.165, 6.1.103, 6.6.44, 6.10.3.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.8.
- **Invariant violated:** *`unmap` must be guarded by exactly the same condition as `map`. Paired operations must have identical guards.*

### CVE-2024-50135 — nvme-pci: fix race condition between reset and nvme_dev_disable()
- **Description:** `nvme_dev_disable()` modifies `dev->online_queues`, so `nvme_pci_update_nr_queues()` racing it can pass invalid values to `blk_mq_update_nr_hw_queues()` (WARN in `pci_irq_get_affinity()`). Fixed by taking `shutdown_lock` and giving up if disable is running or has run.
- **Bug class:** Race condition (unsynchronized shared state).
- **Introduced:** 4.6 (`949928c1c731417cc0f070912c63878b62b544f4`). **Fixed:** mainline **6.12** (`26bc0a81f64ce00fc4342c38eeb2eddaad084dd2`); stable 6.6.59, 6.11.6.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 4.7.
- **Invariant violated:** *`dev->online_queues` must only be read or written under `shutdown_lock`. Controller reset and controller disable must be mutually exclusive.*

### CVE-2024-56756 — nvme-pci: fix freeing of the HMB descriptor table
- **Description:** The HMB descriptor table is sized to the maximum possible descriptor count, but `__nvme_alloc_host_mem()` can break out of the loop early on allocation failure, so `dma_free_coherent()` was passed the wrong (larger) size.
- **Bug class:** Incorrect free size (DMA API misuse).
- **Introduced:** 4.13 (`87ad72a59a38d1df217cfd95bc222a2edfe5d399`). **Fixed:** mainline **6.13** (`3c2fb1ca8086eb139b2a551358137525ae8e0d7a`); stable 5.4.287, 5.10.231, 5.15.174, 6.1.120, 6.6.64, 6.11.11, 6.12.2.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *A DMA free must use exactly the size passed to the corresponding alloc. Track the actual number of descriptors allocated, not the planned maximum.*

### CVE-2026-23174 — nvme-pci: handle changing device dma map requirements
- **Description:** `dma_needs_unmap` may be false initially but become true while mapping the data iterator (enabling swiotlb is one such case). The driver assumed `dma_vecs` was always allocated up front, producing a NULL dereference when the requirement changed mid-iteration.
- **Bug class:** NULL pointer dereference (stale sampled state).
- **Introduced:** 6.17 (`b8b7570a7ec872f2a27b775c4f8710ca8a357adf`). **Fixed:** mainline **6.19** (`071be3b0b6575d45be9df9c5b612f5882bfc5e88`); stable 6.18.10.
- **Severity:** Ubuntu medium (no Red Hat entry, no NVD score retrieved).
- **Invariant violated:** *State that can change during an iteration must be re-evaluated inside the iteration, not sampled once before it starts.*

### CVE-2026-31523 — nvme-pci: ensure we're polling a polled queue
- **Description:** A user can change the polled queue count at run time. During a reset there is a window where a hipri task may poll a queue before the block layer updates the queue maps, racing the now interrupt-driven queue and potentially causing double completions.
- **Bug class:** Race condition (double completion).
- **Introduced:** 5.0 (`4b04cc6a8f86c4842314def22332de1f15de8523`). **Fixed:** mainline **7.0** (`166e31d7dbf6aa44829b98aa446bda5c9580f12a`); stable 5.10.253, 5.15.203, 6.1.168, 6.6.131, 6.12.80, 6.18.21, 6.19.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 4.7.
- **Invariant violated:** *A request must never be completed twice. A queue may only be polled if it is actually in the poll queue map — a hipri poll must never run against an interrupt-driven queue.*

### CVE-2026-43448 — nvme-pci: Fix race bug in nvme_poll_irqdisable()
- **Description:** `pdev` can be disabled by a concurrent `nvme_reset_work()` between the `disable_irq(pci_irq_vector(...))` and the matching `enable_irq(...)`, so the two calls resolve to different IRQ numbers (MSI-X vs INTx), producing "Unbalanced enable for IRQ 10". Fixed by latching the IRQ number in a local.
- **Bug class:** Race condition (unbalanced IRQ refcount).
- **Introduced:** 5.7 (`fa059b856a593a7bddd4d3779ae8ab1380e05d91`). **Fixed:** mainline **7.0** (`fc71f409b22ca831a9f87a2712eaa09ef2bb4a5e`); stable 6.1.167, 6.6.130, 6.12.78, 6.18.19, 6.19.9.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 4.7.
- **Invariant violated:** *`disable_irq()` and `enable_irq()` must operate on the same IRQ number. Any value derived from mutable device state and used in a paired acquire/release must be latched once, not recomputed on the release side.*

### CVE-2026-43449 — nvme-pci: Fix slab-out-of-bounds in nvme_dbbuf_set
- **Description:** `dev->online_queues` is a count; valid indices are `0 .. online_queues - 1`. The loop condition in `nvme_dbbuf_free()`/`nvme_dbbuf_set()` walked past the end. KASAN slab-out-of-bounds read.
- **Bug class:** Out-of-bounds read (off-by-one loop bound).
- **Introduced:** 5.10 (`0f0d2c876c96d4908a9ef40959a44bec21bdd6cf` and siblings). **Fixed:** mainline **7.0** (`b4e78f1427c7d6859229ae9616df54e1fc05a516`); stable 5.10.253, 5.15.203, 6.1.167, 6.6.130, 6.12.78, 6.18.19, 6.19.9.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.1.
- **Invariant violated:** *A loop over queues must be bounded by `dev->online_queues` (indices 1..online_queues-1 for I/O queues; 0 is admin), not by the allocated maximum.*

### CVE-2026-64019 — nvme-pci: fix dma mapping leak on data setup error
- **Description:** The initial DMA mapping leaks during iteration if the tracking descriptor allocation fails for both PRP and SGL. Mappings also leaked when the driver detected an invalid `bio_vec` while mapping PRPs.
- **Bug class:** Resource leak (DMA mapping leak on error path).
- **Introduced:** 6.17 (`7ce3c1dd78fca86ea8b9aee370db10c7a8cfc3c2`). **Fixed:** mainline **7.1** (`1bf86336e4b6cf40873fda47a7fe191446864937`); stable 7.0.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; Ubuntu CVSS 5.5.
- **Invariant violated:** *Every DMA mapping created during descriptor setup must be unmapped on every error path out of that setup, including paths that abort before the tracking structure exists.*

### CVE-2026-64020 — nvme-pci: fix dma_vecs leak on p2p memory
- **Description:** P2P memory is never unmapped, so it does not need tracking; the `dma_vec` allocation was leaking on completion.
- **Bug class:** Memory leak.
- **Introduced:** 6.17 (`b8b7570a7ec872f2a27b775c4f8710ca8a357adf`). **Fixed:** mainline **7.1** (`85686c72966c5ee637893f124ddb31a1cace7bee`); stable 7.0.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.5.
- **Invariant violated:** *Resources that are never unmapped must not have unmap-tracking state allocated for them; tracking allocation and unmap must be gated on the same condition.*

### CVE-2026-64071 — nvme-pci: fix use-after-free in nvme_free_host_mem()
- **Description:** `nvme_free_host_mem()` frees `dev->hmb_sgt` via `dma_free_noncontiguous()` but never cleared the pointer, so a second call on the `nvme_probe()` error path dereferenced freed memory. Reproducible on Thunderbolt-attached NVMe devices that return I/O errors during HMB setup.
- **Bug class:** Use-after-free (double free of a stale pointer).
- **Introduced:** 6.13 (`63a5c7a4b4c49ad86c362e9f555e6f343804ee1d`). **Fixed:** mainline **7.1** (`b35a13036755c5803168a7cb93bc66035c3e65b8`); stable 6.18.34, 7.0.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.8.
- **Invariant violated:** *A pointer must be set to NULL immediately after the resource it names is freed, so that a second free on an error path is a no-op. Cleanup functions must be idempotent.*

### CVE-2026-74383 — nvme-pci: fix out-of-bounds access in nvme_setup_descriptor_pools
- **Description:** `nvme_setup_descriptor_pools()` indexes `dev->descriptor_pools[]` with the `numa_node` forwarded from `hctx->numa_node`. On a `CONFIG_NUMA=n` kernel that is `NUMA_NO_NODE` (-1); because the parameter was declared `unsigned`, it became `UINT_MAX` and the index walked off the array.
- **Bug class:** Out-of-bounds read (signedness bug).
- **Introduced:** 6.16 (`d977506f8863807129d7a11f4057dfb1b38085ea`). **Fixed:** mainline **7.2** (`a192b8cfa447e1b3701a13434a31c392b2e7ed29`); stable 6.18.40, 7.1.5.
- **Severity:** Red Hat Moderate; Ubuntu low; NVD 8.4.
- **Invariant violated:** *A NUMA node id used as an array index must be typed signed, and `NUMA_NO_NODE` must be normalized (to node 0) before indexing. Sentinel values must never reach an index expression.*

---

## tcp — `drivers/nvme/host/tcp.c`

### CVE-2022-48686 — nvme-tcp: fix UAF when detecting digest errors
- **Description:** The `io_work` loop must bail when `rd_enabled` is set to true, so the driver does not keep reading from a socket whose TCP stream is already out-of-sync or corrupted.
- **Bug class:** Use-after-free.
- **Introduced:** 5.0 (`3f2304f8c6d6ed97849057bd16fee99e434ca796`). **Fixed:** mainline **6.0** (`160f3549a907a50e51a8518678ba2dcf2541abea`); stable 5.4.213, 5.10.143, 5.15.68, 5.19.9.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *Once the TCP stream is known to be out of sync (a digest error), the receive loop must stop consuming from the socket immediately — it must not attempt to interpret further bytes.*

### CVE-2022-48789 — nvme-tcp: fix possible use-after-free in transport error_recovery work
- **Description:** `nvme_tcp_submit_async_event_work` checks ctrl and queue state, but the check is not reliable; error recovery must flush `async_event_work` after setting the ctrl state to RESETTING and before destroying the admin queue.
- **Bug class:** Use-after-free (race condition).
- **Introduced:** 5.0 (`3f2304f8c6d6ed97849057bd16fee99e434ca796`). **Fixed:** mainline **5.17** (`ff9fc7ebf5c06de1ef72a69f9b1ab40af8b07f9e`); stable 5.4.181, 5.10.102, 5.15.25, 5.16.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *Setting the controller state to RESETTING is not by itself a fence. Error recovery must flush `async_event_work` after the state change and before destroying the admin queue.*

### CVE-2023-53643 — nvme-tcp: don't access released socket during error recovery
- **Description:** While error recovery is failing reconnect attempts, `nvme list` causes a NULL dereference by calling `getsockname()` on a released socket. The socket is released and recreated during recovery, so it is not safe to access without a check.
- **Bug class:** NULL pointer dereference (use of released object).
- **Introduced:** 6.1 (`02c57a82c0081141abc19150beab48ef47f97f18`). **Fixed:** mainline **6.3** (`76d54bf20cdcc1ed7569a89885e09636e9a8d71d`); stable 6.1.18, 6.2.5.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.8.
- **Invariant violated:** *`queue->sock` must not be dereferenced from a sysfs/ioctl path without holding the queue lock and confirming the queue is live. A released socket must not remain reachable.*
- **Note:** this fix itself regressed — see CVE-2024-53100.

### CVE-2024-53100 — nvme: tcp: avoid race between queue_lock lock and destroy
- **Description:** The `mutex_lock()` added by CVE-2023-53643's fix races `mutex_destroy()` in `nvme_tcp_free_queue()`, producing `DEBUG_LOCKS_WARN_ON(lock->magic != lock)` when `nvme_tcp_get_address()` runs from sysfs.
- **Bug class:** Race condition (use of a destroyed lock).
- **Introduced:** 5.0 (`3f2304f8c6d6ed97849057bd16fee99e434ca796`); regression surfaced by `76d54bf20cdc`. **Fixed:** mainline **6.12** (`782373ba27660ba7d330208cf5509ece6feb4545`); stable 6.1.118, 6.6.62, 6.11.9.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.8.
- **Invariant violated:** *A mutex must not be acquired after it has been destroyed. The lock's lifetime must cover every path that can take it, including sysfs reads racing queue teardown.*

### CVE-2024-56632 — nvme-tcp: fix the memleak while create new ctrl failed
- **Description:** When creating a new controller fails, the tagset occupied by `admin_q` was not freed.
- **Bug class:** Memory leak.
- **Introduced:** 6.7 (`fd1418de10b9ca03d78404cf00a95138689ea369`). **Fixed:** mainline **6.13** (`fec55c29e54d3ca6fe9d7d7d9266098b4514fd34`); stable 6.12.5.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 7.5.
- **Invariant violated:** *If controller setup fails after the admin tagset is allocated, the tagset must be removed before the controller structure is freed.*

### CVE-2025-21927 — nvme-tcp: fix potential memory corruption in nvme_tcp_recv_pdu()
- **Description:** `nvme_tcp_recv_pdu()` did not check the validity of the header length. With header digests enabled, a target could send a PDU with an invalid header length (e.g. 255), causing `nvme_tcp_verify_hdgst()` to access memory outside the allocated area and overwrite it with the calculated digest.
- **Bug class:** Out-of-bounds write (missing bounds check on wire input).
- **Introduced:** 5.0 (`3f2304f8c6d6ed97849057bd16fee99e434ca796`). **Fixed:** mainline **6.14** (`ad95bab0cd28ed77c2c0d0b6e76e03e031391064`); stable 6.12.19, 6.13.7.
- **Severity:** Red Hat **Important**; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *The PDU header length reported by the target must be validated against the expected length for that PDU type before it is used to index the receive buffer or to compute/place a digest. A remote NVMe/TCP target is untrusted input.*

### CVE-2025-38209 — nvme-tcp: remove tag set when second admin queue config fails
- **Description:** Secure-concatenation support made `nvme_tcp_setup_ctrl()` call `nvme_tcp_configure_admin_queue()` twice. The second call (`new=false`) does not call `nvme_remove_admin_tag_set()` on failure, but `nvme_tcp_create_ctrl()` assumes it did, and frees `nvme_tcp_ctrl` (which embeds `admin_tag_set`). The timeout handler later touches the freed tagset — KASAN slab-use-after-free in `blk_mq_queue_tag_busy_iter()`.
- **Bug class:** Use-after-free.
- **Introduced:** 6.15 (`104d0e2f622233477ef7e57e59e8a4c3bb062c82`). **Fixed:** mainline **6.16** (`e7143706702a209c814ed2c3fc6486c2a7decf6c`); stable 6.15.4.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *A resource allocated exactly once must have exactly one owner responsible for freeing it, regardless of which of several setup attempts fails. Caller and callee must not each assume the other frees it.*

### CVE-2025-38264 — nvme-tcp: sanitize request list handling
- **Description:** Validate the request in `nvme_tcp_handle_r2t()` to ensure it is not already part of any list — otherwise a malicious R2T PDU can inject a loop into request list processing.
- **Bug class:** Missing input validation → infinite loop / list corruption (denial of service).
- **Introduced:** 5.0 (`3f2304f8c6d6ed97849057bd16fee99e434ca796`). **Fixed:** mainline **6.16** (`0bf04c874fcb1ae46a863034296e4b33d8fbd66c`); stable 6.12.36, 6.15.5.
- **Severity:** Red Hat **Important**; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *A request referenced by an incoming R2T PDU must not already be on any host-side list. Target-supplied identifiers must never be able to cause a request to be linked twice or inject a cycle into a list.*

### CVE-2026-80862 — nvme-tcp: fix usage of page_frag_cache
- **Description:** nvme uses `page_frag_cache` to preallocate a PDU per preallocated request. Block devices are created in parallel threads, so `page_frag_cache` was used in a non-thread-safe manner, causing incorrect refcounting of backstore pages and premature free (caught by `!sendpage_ok` in the network stack). Fixed by serializing use of the cache.
- **Bug class:** Race condition → refcount corruption / premature free.
- **Introduced:** 6.12 (`4e893ca8117022de68ce1b61c0309e3d17bb8a25`). **Fixed:** mainline **7.3-rc1** (`36ac05f7cfd59d90c597071304b14e98090d5dd1`); stable 6.12.108, 6.18.49, 7.1.13, 7.2.3.
- **Severity:** No Red Hat / Ubuntu / NVD rating retrieved (published 2026-09-04, the day of this pass).
- **Invariant violated:** *`page_frag_cache` is not thread-safe. Per-queue PDU preallocation must be serialized, because block-device creation (and therefore request init) runs in parallel threads.*
- **Caveat:** this is the newest record in the set and the least corroborated — only the kernel CNA record was available.

---

## rdma — `drivers/nvme/host/rdma.c`

### CVE-2021-47378 — nvme-rdma: destroy cm id before destroy qp to avoid use after free
- **Description:** The CM id must always be destroyed before the QP, to avoid receiving a CM event after the QP was destroyed. In the RDMA connection-establishment error flow, do not destroy the QP in the CM event handler; report `cm_error` upward and destroy the QP in `nvme_rdma_alloc_queue()` after destroying the CM id.
- **Bug class:** Use-after-free (teardown ordering).
- **Introduced:** 4.8 (`7110230719602852481c2793d054f866b2bf4a2b`). **Fixed:** mainline **5.15** (`9817d763dbe15327b9b3ff4404fa6f27f927e744`); stable 5.10.70, 5.14.9.
- **Severity:** Red Hat Moderate; Ubuntu low; NVD 9.8.
- **Invariant violated:** *The CM id must be destroyed before the QP, so that no CM event can arrive referencing a destroyed QP. Teardown order must be the reverse of the order in which references were created.*

### CVE-2022-48788 — nvme-rdma: fix possible use-after-free in transport error_recovery work
- **Description:** Same shape as the TCP case: error recovery must flush `async_event_work` after setting the ctrl state to RESETTING and before destroying the admin queue.
- **Bug class:** Use-after-free (race condition).
- **Introduced:** 4.8 (`7110230719602852481c2793d054f866b2bf4a2b`). **Fixed:** mainline **5.17** (`b6bb1722f34bbdbabed27acdceaf585d300c5fd2`); stable 4.19.231, 5.4.181, 5.10.102, 5.15.25, 5.16.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** Same as CVE-2022-48789 — *the RESETTING state change must be followed by an explicit flush of `async_event_work` before admin queue destruction.*

### CVE-2024-49569 — nvme-rdma: unquiesce admin_q before destroy it
- **Description:** When controller creation fails, the kernel hangs forever in `blk_mq_freeze_queue_wait()` because `admin_q` was quiesced to cancel requests but never unquiesced before destroy, so pending requests can never drain.
- **Bug class:** Deadlock / hang.
- **Introduced:** 5.12 (`7da81eaf8710130a9e63d7429627183be5a93787` and siblings). **Fixed:** mainline **6.13** (`5858b687559809f05393af745cbadf06dee61295`); stable 6.6.88, 6.12.5.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.5.
- **Invariant violated:** *A quiesced queue must be unquiesced before it is destroyed; otherwise `blk_mq_freeze_queue_wait()` can never make progress.*

---

## fc — `drivers/nvme/host/fc.c`

### CVE-2023-52508 — nvme-fc: Prevent null pointer dereference in nvme_fc_io_getuuid()
- **Description:** The `nvme_fc_fcp_op` structure describing an AEN operation is initialized with a NULL request pointer. An FC LLDD may call `nvme_fc_io_getuuid()` with an `nvmefc_fcp_req` for an AEN operation. Validate the request pointer before dereference.
- **Bug class:** NULL pointer dereference.
- **Introduced:** 5.19 (`7a41fdf27a4b1ee565ce5bf3e409b2df0b8514c4`, `827fc630e4c8087df5a8e8ee013b686bd6f13736`). **Fixed:** mainline **6.6** (`8ae5b3a685dc59a8cf7ccfe0e850999ba9727a3c`); stable 6.1.56, 6.5.6.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *An operation structure that describes an AEN carries a NULL `request` pointer by construction. Any callback reachable from an LLDD must validate it before dereference.*

### CVE-2024-26846 — nvme-fc: do not wait in vain when unloading module
- **Description:** The module exit path races controller deletion against freeing leftover IDs; `wait_for_completion` did not handle all cases and blktests could hang module unload forever. Fixed by flushing `nvme_delete_wq` and dropping the unnecessary `ida_destroy()`.
- **Bug class:** Deadlock (hang) / potential double-free.
- **Introduced:** 5.3 (`4c73cbdff1119d088ed16d63def59ad32b11b18f`). **Fixed:** mainline **6.8** (`70fbfc47a392b98e5f8dba70c6efc6839205c982`); stable 5.10.211, 5.15.150, 6.1.80, 6.6.19, 6.7.7.
- **Severity:** Red Hat Low; Ubuntu low; NVD 4.4.
- **Invariant violated:** *Module exit must not race controller deletion against ID-allocator destruction. Flushing the delete workqueue is the synchronization point; an ad-hoc completion wait is not sufficient.*

### CVE-2025-40261 — nvme-fc: Ensure ->ioerr_work is cancelled in nvme_fc_delete_ctrl()
- **Description:** `nvme_fc_delete_association()` waits for pending I/O, and an error can queue `->ioerr_work` *after* `cancel_work_sync()` was called. Moving the cancel after `nvme_fc_delete_association()` ensures `->ioerr_work` is not running when the `nvme_fc_ctrl` object is freed. Manifested as `list_del corruption` / `kernel BUG at lib/list_debug.c:52`.
- **Bug class:** Use-after-free (work item outliving its object).
- **Introduced:** 5.11 (`19fce0470f05031e6af36e49ce222d0f0050d432`, `f1cd8c40936ff2b560e1f35159dd6a4602b558e5`). **Fixed:** mainline **6.18** (`0a2c5495b6d1ecb0fa18ef6631450f391a888256`); stable 5.10.253, 5.15.209, 6.1.167, 6.6.118, 6.12.60, 6.17.10.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8.
- **Invariant violated:** *A work item that can be re-queued by in-flight I/O must be cancelled **after** the association that generates that I/O is deleted, not before. No work item may still be queued or running when the object it references is freed.*

### CVE-2025-40342 — nvme-fc: use lock accessing port_state and rport state
- **Description:** `nvme_fc_unregister_remote()` removes the remote port whenever there is no active association. This races reconnect, because `nvme_fc_create_association()` does not take a lock to check `port_state` and atomically increment the rport active count.
- **Bug class:** Race condition (unsynchronized check-then-act on refcount).
- **Introduced:** 4.10 (`e399441de9115cd472b8ace6c517708273ca7997`). **Fixed:** mainline **6.18** (`891cdbb162ccdb079cd5228ae43bdeebce8597ad`); stable 5.10.247, 5.15.197, 6.1.159, 6.6.117, 6.12.58, 6.17.8.
- **Severity:** Red Hat Moderate; Ubuntu low; NVD 8.8.
- **Invariant violated:** *`port_state` must be checked and the rport active count incremented atomically under the same lock. A check-then-act on port liveness is a race against unregister.*

### CVE-2026-23261 — nvme-fc: release admin tagset if init fails
- **Description:** `nvme_fc_init_ctrl()` allocates admin blk-mq resources after `nvme_add_ctrl()` succeeds. Later failures jump to `fail_ctrl`, which tears down controller references but never frees the admin queue/tag set (kmemleak report from blktests nvme/fc).
- **Bug class:** Memory leak.
- **Introduced:** 6.18 (`0d1840b2dd8fe073c020c39bf8e8e89488070801` and siblings). **Fixed:** mainline **6.19** (`d1877cc7270302081a315a81a0ee8331f19f95c8`); stable 6.6.124, 6.12.70, 6.18.10.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *Every failure path after admin tagset allocation must call `nvme_remove_admin_tag_set()`. A `fail_*` label must unwind everything acquired before the jump, not just the most recent acquisition.*

---

## fabrics — `drivers/nvme/host/fabrics.c` / `fabrics.h`

### CVE-2024-41082 — nvme-fabrics: use reserved tag for reg read/write command
- **Description:** If userspace issues enough nvme commands to exhaust all `admin_q` tags, and a reset or I/O timeout occurs before they finish, the reconnect path cannot get a tag to update the NVMe registers, hanging the kernel forever. Fixed by making `reg_read32()`/`reg_read64()`/`reg_write32()` use reserved tags.
- **Bug class:** Deadlock (resource starvation).
- **Introduced:** 5.4 (`e7832cb48a654cd12b2bc9181b2f0ad49d526ac6`). **Fixed:** mainline **6.10** (`7dc3bfcb4c9cc58970fff6aaa48172cb224d85aa`); stable 6.9.11.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *Controller enable/disable register access must always be able to make progress. It must use reserved tags so that userspace cannot starve the reconnect path by exhausting the admin queue's normal tags.*

*(CVE-2024-27435 also touches `fabrics.h` — see the core section.)*

---

## multipath — `drivers/nvme/host/multipath.c`

### CVE-2024-53093 — nvme-multipath: defer partition scanning
- **Description:** Partition scanning must be suppressed inside the controller's `scan_work` context. If a path error occurs there, the I/O waits until a path becomes available or all paths are torn down — but that action also happens in `scan_work`, so it deadlocks. The scan is deferred to a different context.
- **Files:** `multipath.c`, `nvme.h`.
- **Bug class:** Deadlock.
- **Introduced:** 4.15 (`32acab3181c7053c775ca128c3a5c6ce50197d7f`). **Fixed:** mainline **6.12** (`1f021341eef41e77a633186e9be5223de2ce5d48`); stable 6.1.118, 6.6.62, 6.11.9.
- **Severity:** Red Hat Moderate; Ubuntu low; NVD 7.5.
- **Invariant violated:** *No work item may block waiting on a condition that only the same work item can satisfy. Partition scanning must not run in `scan_work` context, because path recovery also runs there.*
- **Note:** this fix itself regressed — see CVE-2025-68218.

### CVE-2025-38397 — nvme-multipath: fix suspicious RCU usage warning
- **Description:** `nvme_mpath_add_sysfs_link()` triggers "RCU-list traversed in non-reader section!!" at `multipath.c:1203` — the SRCU lock is held but the list traversal was not annotated with it.
- **Bug class:** Incorrect RCU annotation (potential use-after-free if the annotation were wrong in the other direction).
- **Introduced:** 6.15 (`4dbd2b2ebe4cc5f101881e2c091a70ccd38db7ee`). **Fixed:** mainline **6.16** (`d6811074203b13f715ce2480ac64c5b1c773f2a5`); stable 6.15.6.
- **Severity:** No Red Hat entry; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *An SRCU-protected list traversal must declare the specific lock that protects it (`srcu_read_lock_held(&head->srcu)`), so lockdep can verify the protection actually exists.*

### CVE-2025-68218 — nvme-multipath: fix lockdep WARN due to partition scan work
- **Description:** blktests nvme/014, 057, 058 fail with a lockdep WARN indicating a possible deadlock from the dependency chain `disk->open_mutex` → kblockd workqueue completion → `partition_scan_work` completion. Fixed by running `partition_scan_work` on `nvme_wq` rather than kblockd.
- **Bug class:** Deadlock (workqueue dependency cycle).
- **Introduced:** 6.12 (`1f021341eef41e77a633186e9be5223de2ce5d48` and siblings — the CVE-2024-53093 fix). **Fixed:** mainline **6.18** (`6d87cd5335784351280f82c47cc8a657271929c3`); stable 6.1.159, 6.6.118, 6.12.60, 6.17.10.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 7.5.
- **Invariant violated:** *Work whose completion is waited on while `disk->open_mutex` is held must not run on `kblockd`, which is itself in that dependency chain. Deferring work is only safe if the destination workqueue is outside the lock's dependency graph.*

---

## ioctl — `drivers/nvme/host/ioctl.c`

### CVE-2026-64072 — nvme: fix bio leak on mapping failure
- **Description:** The local `bio` was always NULL, so the bio leaked when the integrity mapping failed. Fixed by getting it directly from the request.
- **Bug class:** Memory leak.
- **Introduced:** 6.18 (`d0d1d522316e91f2b935a78bbf962b8e529d8c4f`). **Fixed:** mainline **7.1** (`2279cd9c61a330e5de4d6eb0bc422820dd6fdf36`); stable 6.18.34, 7.0.11.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *An error path must release the object the request actually owns, obtained from the request — not a local variable that may never have been assigned.*

---

## sysfs — `drivers/nvme/host/sysfs.c`

### CVE-2024-27392 — nvme: host: fix double-free of struct nvme_id_ns in ns_update_nuse()
- **Description:** When `nvme_identify_ns()` fails it frees the `struct nvme_id_ns` pointer before returning, but `ns_update_nuse()` called `kfree()` on it anyway — KASAN double-free, observed by blktests nvme/045.
- **Bug class:** Double-free.
- **Introduced:** 6.8 (`a1a825ab6a60380240ca136596732fdb80bad87a`). **Fixed:** mainline **6.9** (`8d0d2447394b13fb22a069f0330f9c49b7fff9d3`); stable 6.8.2.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 7.8.
- **Invariant violated:** *Ownership on failure must be unambiguous: if a callee frees its output buffer on error, the caller must not free it again. Every allocated object must have exactly one free.*

---

## pr (persistent reservations) — `drivers/nvme/host/pr.c`

### CVE-2026-23244 — nvme: fix memory allocation in nvme_pr_read_keys()
- **Description:** `nvme_pr_read_keys()` takes `num_keys` from userspace and uses it in `struct_size()` to size an allocation, bounded by `PR_KEYS_MAX` (64K). A large `num_keys` produces up to a 4 MB allocation, warning in the page allocator when the order exceeds `MAX_PAGE_ORDER`. Fixed by using `kvzalloc()`.
- **Bug class:** Unbounded (userspace-controlled) allocation → allocator warning / DoS.
- **Introduced:** 6.5 (`5fd96a4e15de8442915a912233d800c56f49001d`). **Fixed:** mainline **7.0** (`c3320153769f05fd7fe9d840cb555dd3080ae424`); stable 6.6.130, 6.12.77, 6.18.17, 6.19.7.
- **Severity:** Red Hat Low; Ubuntu medium; NVD 7.1. Found by syzkaller.
- **Invariant violated:** *An allocation whose size is derived from a userspace-supplied count must use `kvzalloc()`/`kvmalloc()` (or be bounded below `MAX_PAGE_ORDER`), never a plain `kzalloc()`.*

---

## apple — `drivers/nvme/host/apple.c`

### CVE-2024-43913 — nvme: apple: fix device reference counting
- **Description:** Drivers must call `nvme_uninit_ctrl()` after a successful `nvme_init_ctrl()`. The Apple driver did this wrong and leaked the controller device memory on a tagset failure. The allocation side was split out to clarify the error-handling boundary.
- **Bug class:** Refcount/lifetime error → memory leak.
- **Introduced:** 5.19 (`5bd2927aceba181b84286e00aa2f56e117e699c3`). **Fixed:** mainline **6.11** (`b9ecbfa45516182cd062fecd286db7907ba84210`); stable 6.6.64, 6.10.5.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.0.
- **Invariant violated:** *`nvme_uninit_ctrl()` must be called after any successful `nvme_init_ctrl()`. Allocation and initialization must be split so the error-handling boundary is unambiguous.*

### CVE-2026-72131 — nvme-apple: Prevent shared tags across queues on Apple A11
- **Description:** On Apple A11, tags of pending commands must be unique across the admin and I/O queues, else the firmware crashes with "duplicate tag error for tag N". The existing M1 workaround (reserving two tags for the admin queue) was extended to A11.
- **Bug class:** Hardware/firmware contract violation (tag-space collision).
- **Introduced:** 6.18 (`04d8ecf37b5e06d16228a4d37d8548c17cf70461`). **Fixed:** mainline **7.2** (`6fe0687245e8406bf26143bd45eb16441bbe5280`); stable 6.18.40, 7.1.5.
- **Severity:** Red Hat Moderate; Ubuntu medium; Ubuntu CVSS 5.5.
- **Invariant violated:** *On Apple A11/M1 controllers, tags of in-flight commands must be unique across the admin and I/O queues. Queues must not share a tag space where the firmware requires uniqueness.*

---

## header — `drivers/nvme/host/nvme.h`

### CVE-2022-50388 — nvme: fix multipath crash caused by flush request when blktrace is enabled
- **Description:** A flush request initialized by `blk_kick_flush()` has a NULL bio and may be handled by `nvme_end_req()` on completion. With blktrace enabled, `nvme_trace_bio_complete()` with multipath active dereferences the NULL bio.
- **Files:** `nvme.h` (`nvme_trace_bio_complete()`).
- **Bug class:** NULL pointer dereference.
- **Introduced:** 5.4 (`35fe0d12c8a3d5e45f297562732ddc9ba9dc58dd`). **Fixed:** mainline **6.2** (`3659fb5ac29a5e6102bebe494ac789fd47fb78f4`); stable 5.10.163, 5.15.87, 6.0.19, 6.1.5.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 5.5.
- **Invariant violated:** *`req->bio` may legitimately be NULL (flush requests carry no bio). Every completion/tracepoint helper must NULL-check it before dereference.*

---

## Adjacent — fixed OUTSIDE `drivers/nvme/host`, reached via nvme call paths

These two matched a text search for `drivers/nvme/host` only because nvme frames appear in the crash trace. **The CNA `programFiles` field for both names a file in another subsystem.** They are included because they document nvme-triggered failure modes, but they are **not** nvme host driver defects and must not be treated as such.

### CVE-2024-47696 — RDMA/iwcm: Fix WARNING at check_flush_dependency
- **Fix location:** `drivers/infiniband/core/iwcm.c` (**not** nvme). Reached via `nvme_rdma_free_queue()` → `_destroy_id()` → `flush_workqueue(iwcm_wq)`.
- **Bug class:** Potential deadlock (missing `WQ_MEM_RECLAIM` on a flushed workqueue).
- **Introduced:** 6.11 lineage. **Fixed:** mainline **6.12** (`86dfdd8288907f03c18b7fb462e0e232c4f98d89`).
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 7.5.
- **Relevance to nvme:** `nvme_rdma_reset_ctrl_work()` is the caller that surfaced it. The nvme code was correct.

### CVE-2026-80589 — block: stop the timeout timer when releasing a never added disk
- **Fix location:** `block/genhd.c` (**not** nvme). `disk_release()` did not stop `q->timeout` for a disk whose probe failed before `add_disk()`.
- **Bug class:** Use-after-free (timer outliving the freed request_queue).
- **Introduced:** 6.0 (`6f8191fdf41d3a53cc1d63fe2234e812c55a0092`). **Fixed:** mainline **7.2** (`26cb8ebbfaf713c82e142d08828d4d765057633b`); stable 6.1.184, 6.6.153, 6.12.105, 6.18.46, 7.1.10.
- **Severity:** Red Hat Moderate; Ubuntu medium; NVD 9.8. Found by FuzzNvme.
- **Relevance to nvme:** nvme reaches it because `nvme_update_ns_info()` submits Report Zones or FDP io-mgmt-recv on `ns->queue` **before the disk is added**. That behaviour is legal; the block layer's release path was incomplete. Worth knowing as a shape: nvme issues I/O on a namespace queue before `add_disk()`.

---

## Invariants

The consolidated must / must-never rules distilled from the whole CVE set above. **These are review
heuristics, not spec-conformance requirements** — they are useful because each one is a rule the
driver already tried to follow and got wrong at least once. Each is annotated with the CVE(s) that
motivated it.

### 1. Teardown ordering and lifetime

- **A controller must not be freed while any queue, namespace, or work item still references it.**
  The final `put` must come after all references are released. *(CVE-2025-68265)*
- **Teardown order must be the exact reverse of setup order.** The CM id must be destroyed before
  the QP, so no CM event can reference a destroyed QP. *(CVE-2021-47378)*
- **Keep-alive must be stopped — in-flight completed or cancelled — before the admin queue is
  destroyed and its tagset removed.** *(CVE-2024-45013, CVE-2024-53169)*
- **No work item may still be queued or running when the object it references is freed, and a work
  item that in-flight I/O can re-queue must be cancelled *after* that I/O source is torn down, not
  before.** *(CVE-2025-40261, CVE-2024-26846)*
- **A quiesced queue must be unquiesced before it is destroyed**, or the freeze wait can never
  complete. *(CVE-2024-49569)*
- **A mutex must never be acquired after it has been destroyed.** The lock's lifetime must cover
  every path that can take it, including sysfs reads racing queue teardown. *(CVE-2024-53100)*
- **Every periodic work item started during init must be stopped on the matching uninit path,
  including when init failed partway through.** *(CVE-2024-45013, CVE-2024-43913)*
- **`nvme_uninit_ctrl()` must be called after any successful `nvme_init_ctrl()`.** Split allocation
  from initialization so the error boundary is unambiguous. *(CVE-2024-43913)*

### 2. Free exactly once; pointer hygiene

- **Every allocated object must be freed exactly once.** Ownership on failure must be unambiguous:
  if a callee frees its output on error, the caller must not free it again.
  *(CVE-2024-27392, CVE-2024-41073)*
- **A pointer must be set to NULL immediately after the resource it names is freed**, so a second
  cleanup call on an error path is a no-op. Cleanup functions should be idempotent.
  *(CVE-2026-64071)*
- **A resource must never be reallocated over a live pointer.** If reset can re-enter an allocator,
  the allocator must release the previous instance first. *(CVE-2026-23360)*
- **A flag that marks "this request owns an extra payload" must be cleared at cleanup**, so a retry
  cannot free the payload a second time. *(CVE-2024-41073)*
- **A free must use exactly the size passed to the corresponding alloc** — track the actual count
  allocated, not the planned maximum. *(CVE-2024-56756)*
- **A pointer an allocator may set to an `ERR_PTR` must be checked with `IS_ERR_OR_NULL()`, not
  just for NULL, before teardown dereferences it.** *(CVE-2022-49492)*

### 3. Locking, RCU/SRCU, and check-then-act

- **Every traversal of an `nvme_ns_head` sibling list must be inside that head's SRCU read-side
  critical section, and removal must synchronize with that same SRCU — not global RCU.**
  *(CVE-2022-49003)*
- **An SRCU-protected list traversal must name the specific lock that protects it**, so lockdep can
  verify the protection actually exists. *(CVE-2025-38397)*
- **`dev->online_queues` must only be read or written under `shutdown_lock`; reset and disable must
  be mutually exclusive.** *(CVE-2024-50135)*
- **`port_state` must be checked and the rport active count incremented atomically under the same
  lock** — a check-then-act on port liveness races unregister. *(CVE-2025-40342)*
- **A controller state change is not by itself a fence.** After setting RESETTING, error recovery
  must explicitly flush `async_event_work` before destroying the admin queue.
  *(CVE-2022-48789, CVE-2022-48788, CVE-2022-48790)*
- **A value derived from mutable device state and used in a paired acquire/release must be latched
  once, not recomputed on the release side** — `disable_irq()` and `enable_irq()` must use the same
  IRQ number. *(CVE-2026-43448)*
- **A per-queue `page_frag_cache` is not thread-safe** and must be serialized, because request
  preallocation runs in parallel block-device-creation threads. *(CVE-2026-80862)*

### 4. Completion and request-state discipline

- **A request must never be completed twice.** A queue may only be polled if it is actually in the
  poll queue map — a hipri poll must never run against an interrupt-driven queue.
  *(CVE-2026-31523)*
- **A request must not be linked onto a list it is already on**; nothing the peer sends may inject a
  cycle into a host-side list. *(CVE-2025-38264)*
- **`req->bio` may legitimately be NULL** (flush requests carry no bio); every completion and
  tracepoint helper must NULL-check before dereference. *(CVE-2022-50388)*
- **An AER must not be submitted once the controller state has left LIVE/CONNECTING**, and the
  state check must live in the submit path itself, not only in the scheduler that queues it.
  *(CVE-2022-48790)*

### 5. Untrusted wire input (a remote fabrics target is hostile input)

- **The PDU header length reported by a target must be validated against the expected length for
  that PDU type before it is used to index the receive buffer or place a digest.**
  *(CVE-2025-21927)*
- **A request identified by an incoming R2T PDU must be validated as not already listed** before it
  is used. *(CVE-2025-38264)*
- **Once the stream is known to be out of sync (a digest error), the receive loop must stop
  consuming from the socket immediately** and must not attempt to interpret further bytes.
  *(CVE-2022-48686)*
- **A released socket must never remain reachable**; `queue->sock` must not be dereferenced from a
  sysfs/ioctl path without holding the queue lock and confirming the queue is live.
  *(CVE-2023-53643)*

### 6. Untrusted local input (userspace and controller-reported data)

- **An allocation sized from a userspace-supplied count must use `kvzalloc()`/`kvmalloc()`**, or be
  bounded below `MAX_PAGE_ORDER`. *(CVE-2026-23244)*
- **Every index derived from controller-reported identify/log data must be range-checked with the
  correct inclusive/exclusive comparison before use.** *(CVE-2026-74361)*
- **Userspace must not be able to starve a recovery path of resources.** Register read/write must
  use reserved tags so controller enable/disable can always make progress. *(CVE-2024-41082)*
- **A reserved tag needed to make reconnect progress must never be consumable by a command that can
  itself block until reconnect completes.** *(CVE-2024-27435)*

### 7. Bounds, indexing, and sizing

- **An array indexed by an identifier must be sized by that identifier's maximum value + 1
  (`nr_node_ids`), never by the count of valid identifiers (`num_possible_nodes()`).** Identifier
  spaces are not required to be dense. *(CVE-2026-74384)*
- **A node id used as an array index must be typed signed, and `NUMA_NO_NODE` must be normalized
  before indexing.** Sentinel values must never reach an index expression. *(CVE-2026-74383)*
- **A loop over queues must be bounded by `dev->online_queues`** (I/O queues are indices
  1..online_queues-1; 0 is admin), not by the allocated maximum. *(CVE-2026-43449)*
- **A descriptor pool must be sized for the worst case**, with numerator and divisor in consistent
  units. *(CVE-2022-50756)*

### 8. Error-path completeness and paired-operation symmetry

- **Every error-unwind path must release everything the successful path would have owned, in
  reverse order of acquisition.** A `fail_*` label must unwind everything acquired before the jump,
  not just the most recent acquisition.
  *(CVE-2023-53670, CVE-2023-53792, CVE-2023-53852, CVE-2026-23261, CVE-2024-56632, CVE-2026-64072)*
- **If controller setup fails after the admin tagset is allocated, the tagset must be removed before
  the controller structure is freed** — and a resource allocated once must have exactly one owner
  responsible for freeing it regardless of which of several setup attempts fails.
  *(CVE-2024-56632, CVE-2025-38209, CVE-2026-23261)*
- **`unmap` must be guarded by exactly the same condition as `map`.** Paired operations must have
  identical guards. *(CVE-2024-42276)*
- **Every DMA mapping created during descriptor setup must be unmapped on every error path out of
  that setup**, including paths that abort before the tracking structure exists. *(CVE-2026-64019)*
- **Resources that are never unmapped must not have unmap-tracking state allocated for them** —
  tracking allocation and unmap must be gated on the same condition. *(CVE-2026-64020)*
- **An error path must release the object the *request* owns, obtained from the request** — not a
  local variable that may never have been assigned. *(CVE-2026-64072)*

### 9. Deadlock avoidance

- **No work item may block waiting on a condition that only the same work item can satisfy.**
  Partition scanning must not run in `scan_work` context, because path recovery also runs there.
  *(CVE-2024-53093)*
- **Deferring work is only safe if the destination workqueue is outside the waited-on lock's
  dependency graph.** Work whose completion is waited on under `disk->open_mutex` must not run on
  `kblockd`. *(CVE-2025-68218)*
- **Module exit must synchronize controller deletion against ID-allocator destruction by flushing
  the delete workqueue** — an ad-hoc completion wait is not sufficient. *(CVE-2024-26846)*

### 10. Callback and hardware contracts

- **An operation structure describing an AEN carries a NULL `request` pointer by construction**; any
  callback reachable from a low-level driver must validate it before dereference.
  *(CVE-2023-52508)*
- **On Apple A11/M1 controllers, tags of in-flight commands must be unique across the admin and I/O
  queues** — queues must not share a tag space where the firmware requires uniqueness.
  *(CVE-2026-72131)*
- **State that can change during an iteration must be re-evaluated inside the iteration**, not
  sampled once before it starts. *(CVE-2026-23174)*

### 11. Meta-observation: fixes in this driver regress

Four of the CVEs above were introduced *by the fix for an earlier CVE in this same list*:

| Fix for | …introduced | Shape |
|---|---|---|
| CVE-2024-45013 (`a54a93d0e359`) | CVE-2024-53169 | Moving keep-alive stop later opened a shutdown race |
| CVE-2023-53643 (`76d54bf20cdc`) | CVE-2024-53100 | Adding a `mutex_lock()` raced `mutex_destroy()` |
| CVE-2024-53093 (`1f021341eef4`) | CVE-2025-68218 | Deferring the scan to kblockd created a lockdep cycle |
| CVE-2025-68265 (`03b3bcd319b3`) | CVE-2026-23360 | Reordering the controller `put` orphaned the old admin queue |

*Heuristic, not a rule:* changes to controller/queue teardown ordering in this driver have a
demonstrated history of trading one lifetime bug for another. Reviewing such a change means
checking both the new ordering **and** every other consumer of the moved operation.
