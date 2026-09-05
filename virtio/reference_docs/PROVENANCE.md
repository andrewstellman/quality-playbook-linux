# virtio reference documentation — provenance

Every file here was copied **verbatim from its original upstream source**. Nothing in this
folder is summarized, paraphrased, compiled, or AI-generated. Each file is traceable to an
exact upstream path and commit below.

Retrieved: 2026-08-04

---

## 1. The virtio specification (normative)

The behavioral contract for the driver subsystem under audit.

- Source: https://github.com/oasis-tcs/virtio-spec (OASIS Technical Committee's own repo)
- Revision: **virtio-v1.4-cs01**, dated 8 April 2026
- Commit: `bb1dd2e`

| File here | Upstream file | Covers |
|---|---|---|
| `virtio-spec-v1.4-introduction.tex` | `introduction.tex` | terminology, RFC 2119 keyword definitions, conformance targets |
| `virtio-spec-v1.4-content.tex` | `content.tex` | device initialization, device status, device reset, feature negotiation, config space, virtqueue basics |
| `virtio-spec-v1.4-transport-pci.tex` | `transport-pci.tex` | virtio-over-PCI: common cfg layout, ISR, notification, queue reset, MSI-X vs INTx |
| `virtio-spec-v1.4-transport-mmio.tex` | `transport-mmio.tex` | virtio-over-MMIO: register map, Version semantics, status/reset |
| `virtio-spec-v1.4-split-ring.tex` | `split-ring.tex` | split virtqueue format and memory-ordering rules |
| `virtio-spec-v1.4-packed-ring.tex` | `packed-ring.tex` | packed virtqueue format and memory-ordering rules |
| `virtio-spec-v1.4-conformance.tex` | `conformance.tex` | per-component conformance clause lists |
| `virtio-spec-v1.4-admin.tex` | `admin.tex` | admin virtqueue (VIRTIO_F_ADMIN_VQ) |
| `virtio-spec-v1.4-device-parts.tex` | `device-parts.tex` | device state parts |
| `virtio-spec-v1.4-shared-mem.tex` | `shared-mem.tex` | shared memory regions |

### How to read the spec files

The spec is LaTeX. Normative requirements are explicitly delimited:

- `\drivernormative{...}` — binds the **driver** (the code under audit)
- `\devicenormative{...}` — binds the **device** (NOT the driver's responsibility)
- `\begin{note}` — non-normative commentary

RFC 2119 keywords are defined in the introduction file and carry their standard meanings.
**A driver-side SHOULD is not a MUST.** Do not report a deviation from a SHOULD as a spec
violation without labelling it as such. Attribute a requirement to the driver only when it
appears under `\drivernormative`.

Transport scope: `transport-ccw.tex` (s390 channel I/O) was deliberately excluded — that
transport lives in `drivers/s390/virtio`, outside the audited surface.

## 2. Linux kernel documentation

Copied from the **same commit as the source code under audit**, so the docs and the code
match exactly.

- Source: https://github.com/torvalds/linux
- Commit: `c21bb4193868a8de71fc4693fa741e195fdf5d86` (2026-08-04)

| File here | Upstream path |
|---|---|
| `linux-driver-api-virtio-index.rst` | `Documentation/driver-api/virtio/index.rst` |
| `linux-driver-api-virtio.rst` | `Documentation/driver-api/virtio/virtio.rst` |
| `linux-driver-api-writing_virtio_drivers.rst` | `Documentation/driver-api/virtio/writing_virtio_drivers.rst` |
| `linux-dt-bindings-virtio-mmio.yaml` | `Documentation/devicetree/bindings/virtio/mmio.yaml` |
| `linux-dt-bindings-virtio-device.yaml` | `Documentation/devicetree/bindings/virtio/virtio-device.yaml` |
| `linux-dt-bindings-virtio-pci-iommu.yaml` | `Documentation/devicetree/bindings/virtio/pci-iommu.yaml` |
| `linux-process-coding-style.rst` | `Documentation/process/coding-style.rst` |

---

## Removed on 2026-08-04 (not primary sources)

Two files previously in this folder were removed because they were AI-compiled secondary
material presented in the register of a specification. Both are preserved unchanged in
`../virtio.pre-cleanup-2026-08-04/`.

- **`virtio-spec-behavioral-contracts.md`** — self-described as "Extracted from OASIS ...
  specifications (v1.0, v1.1, v1.2), community discussions, and implementation experience."
  It contained at least one requirement that misstates the spec: under a heading
  "Device Reset — MUST Requirements" it asserted *"Driver MUST wait for device_status read
  to return 0 before reinitializing."* The actual spec (v1.4-cs01, `content.tex`,
  `\drivernormative` Device Reset) says the driver **SHOULD** consider a reset complete when
  it reads status as 0. A prior audit built a confirmed bug on that fabricated MUST.
- **`virtio-community-development-history.md`** — self-described as "Compiled from LWN.net
  articles." Secondary, synthesized, and not a statement of required behavior.

Use the spec files above as the authority for required behavior.
