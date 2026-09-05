# quality-playbook-linux

Linux kernel subsystems staged as [Quality Playbook](https://github.com/andrewstellman/quality-playbook) targets, each paired with the authoritative specifications its code is supposed to conform to.

The Quality Playbook derives a project's *intended* behavior from gathered documentation, renders it as testable requirements, then checks the code against them. Its output is only as good as the specifications it is given — so the point of this repo is the pairing: a subsystem's source next to the normative text that governs it, verified complete rather than truncated.

Snapshot of `torvalds/linux` master. There is deliberately no `quality/` directory here; runs start fresh and write their own.

## Why the specs matter

An A/B run on the same NVMe subsystem, differing only in documentation:

| | near-empty corpus | full spec corpus |
|---|---|---|
| requirements derived | 4 | 15 |
| Tier 1 (byte-verified citation into a spec) | 0 | 6 |
| functional areas covered | 1 | 5 |
| finding class | "these two code paths disagree" | "the code violates §X of the specification" |

Without a spec, every requirement is Tier 3 — code-is-the-spec — and the analysis can only ever surface internal inconsistency. With one, it can find conformance defects, which is the class a maintainer acts on.

## Targets

| target | source | files | lines | governing specifications |
|---|---|---|---|---|
| `tls` | `net/tls` | 7 | 6,568 | RFC 8446 (TLS 1.3), 5246, 5288, 7905, 5116 |
| `virtio` | `drivers/virtio` | 20 | 16,809 | OASIS VIRTIO v1.4 (10 spec sections) |
| `mptcp` | `net/mptcp` | 21 | 17,295 | RFC 8684 (MPTCP v1), 6824, 8041 |
| `smc` | `net/smc` | 19 | 17,807 | RFC 7609 (SMC-R) |
| `sctp2` | `net/sctp` | 33 | 43,963 | RFC 9260, 4960, 6458, 8260, 4895, 5061, 3758 |
| `nvme-host-docs` | `drivers/nvme/host` | 17 | 29,338 | NVMe Base 2.4, NVM Command Set 1.3, PCIe/RDMA/TCP transports |

Each target holds:

- the subsystem source tree, unmodified
- `reference_docs/cite/` — authoritative contracts, citable with byte verification
- `reference_docs/` (top level) — background context that must **not** be citable as authoritative: CVE/advisory material, git-history-derived notes, provenance records

That split is load-bearing. Treating an advisory as an authoritative contract is a known false-positive trap — an earlier campaign scored 0/3 precision doing exactly that.

## Corpus inventory

Line counts are the verification: a truncated spec fetch produces citations that verify against a corrupted document, silently.

<details>
<summary><code>tls</code> — 8 files</summary>

```
   8963  rfc8446-tls13.txt          TLS 1.3
   5827  rfc5246-tls12.txt          TLS 1.2
   1235  rfc5116-aead.txt           AEAD interface
    451  rfc5288-aes-gcm.txt        AES-GCM cipher suites
    451  rfc7905-chacha20.txt       ChaCha20-Poly1305
    596  tls-offload.rst            kernel docs
    346  tls.rst
    222  tls-handshake.rst
```
</details>

<details>
<summary><code>virtio</code> — 13 files</summary>

```
   1214  virtio-spec-v1.4-transport-pci.txt
   1044  virtio-spec-v1.4-content.txt
    736  virtio-spec-v1.4-split-ring.txt
    731  virtio-spec-v1.4-packed-ring.txt
    643  virtio-spec-v1.4-admin.txt
    614  virtio-spec-v1.4-transport-mmio.txt
    353  virtio-spec-v1.4-introduction.txt
    323  virtio-spec-v1.4-conformance.txt
    237  virtio-spec-v1.4-device-parts.txt
     42  virtio-spec-v1.4-shared-mem.txt
    196  linux-driver-api-writing_virtio_drivers.rst
    145  linux-driver-api-virtio.rst
     11  linux-driver-api-virtio-index.rst
```
</details>

<details>
<summary><code>mptcp</code> — 5 files</summary>

```
   3795  rfc8684-mptcp-v1.txt       MPTCP v1
   3587  rfc6824-mptcp-v0.txt       v0, still referenced in code
   1683  rfc8041-mptcp-usecases.txt
    166  mptcp-sysctl.rst
    156  mptcp.rst
```
</details>

<details>
<summary><code>smc</code> — 2 files</summary>

```
   8011  rfc7609-smc-r.txt          Shared Memory Communications over RDMA
    140  smc-sysctl.rst
```
</details>

<details>
<summary><code>sctp2</code> — 8 files</summary>

```
   8515  rfc4960.txt                prior core spec, still cited throughout the code
   7396  rfc9260.txt                current core spec — the primary contract
   6443  rfc6458.txt                Sockets API
   2299  rfc5061.txt                dynamic address reconfiguration
   1291  rfc8260.txt                stream schedulers
   1235  rfc3758.txt                partial reliability
   1067  rfc4895.txt                authenticated chunks
     42  sctp.rst
```
</details>

<details>
<summary><code>nvme-host-docs</code> — 13 files</summary>

```
  50743  nvme-base-2.4.txt          NVM Express Base Specification 2.4
   9750  nvme-nvm-command-set-1.3.txt
   2741  nvme-pcie-transport-1.4.txt
   2146  nvme-tcp-transport-1.3.txt
    723  nvme-rdma-transport-1.2.txt
    453  sysfs-nvme.txt
    368  nvme-pci-endpoint-target.rst
    248  data-integrity.rst          NVMe rides on blk-mq
    153  blk-mq.rst
    150  biovecs.rst
    119  pr.rst
     95  writeback_cache_control.rst
     77  feature-and-quirk-policy.rst
```

Plus, as non-citable context: `git-history-nvme-host.txt` (2,908 commits touching this path — every `Fixes:` commit states an invariant that was once violated) and `nvme-host-cves-and-advisories.md`.
</details>

## Running a target

```bash
git clone --depth=1 https://github.com/andrewstellman/quality-playbook.git qpb
python3 qpb/bin/install_skill.py --into <target> --ai-tool claude
cp qpb/schemas.md <target>/.claude/skills/quality-playbook/schemas.md
mkdir -p <target>/quality
python3 <target>/.claude/skills/quality-playbook/bin/qpb_validate.py <target>
```

The Phase 0 validator must print `status=ok findings=0` before any phase runs.

Copying `schemas.md` in is not optional: it is currently missing from installs while `quality_gate.py` references it 69 times, so without it a run is judged against a contract it cannot read.

### One trap worth knowing

Spec text converted from PDF, and RFCs generally, contain page-break form-feed bytes. Python's `str.splitlines()` — which the citation verifier uses — counts them as line breaks; `grep -n` and `wc -l` do not. On the NVMe base spec the divergence reaches 870 lines. Never hand-author a citation line number from `grep -n` output; resolve it through the verifier and round-trip it.

## Provenance and licensing

Source trees are unmodified snapshots of `torvalds/linux` (GPL-2.0 WITH Linux-syscall-note). Specification text is redistributed verbatim from its publisher: IETF RFCs from the RFC Editor, the VIRTIO specification from OASIS, NVM Express specifications from nvmexpress.org, kernel documentation from the Linux tree.

**Nothing here is originally licensed by this repository.** Every included file remains under the license and terms of its upstream publisher, and those terms govern any use or further redistribution. The `LICENSE` file applies only to material original to this repository.
