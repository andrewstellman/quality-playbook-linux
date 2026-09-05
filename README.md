# quality-playbook-linux

Using the [Quality Playbook](https://github.com/andrewstellman/quality-playbook) to find bugs in Linux kernel subsystems.

This is traditional quality engineering — derive the requirements, then verify the implementation against them — with AI doing the parts that used to make it too slow to bother with. A person can read a 500-page specification and check 30,000 lines of driver code against it. Almost nobody does, because it takes weeks. That's the part being sped up.

The target is a specific and under-served class of defect: **code that looks fine when you read it, but doesn't do what the specification says.** Nothing about it trips a static analyzer or a code review. The tests pass, because the tests were written from the same misunderstanding as the code. You only catch it by having the spec in hand and checking, clause by clause, whether the implementation honors it.

The second class, closely related: **one contract, several sibling implementations** — four transports, five backends — where the test suite exercises the common one and a sibling has quietly drifted. That's what produced the first patch accepted upstream from this work, a virtio-pci interrupt-return fix.

## How this works

1. **Clone the subsystem** out of `torvalds/linux` into its own target directory, recording the exact commit.
2. **Gather the documentation** — the specifications the code is supposed to conform to, using the [documentation gathering prompt](https://github.com/andrewstellman/quality-playbook/blob/main/references/DOC_GATHERING_PROMPT.md) from the Quality Playbook repo. This step decides everything downstream; see below.
3. **Run the Quality Playbook** — currently Claude Opus, all six phases plus iterations. It derives requirements from the specs, renders them, and audits the code against them.
4. **Review the findings and suggested patches** and decide which are real. Most candidates die here, to a guard or a validation that exists three functions up the call chain. That's the process working.
5. **Reproduce under QEMU**, then apply the patch for a red/green test — the bug demonstrably present before, demonstrably gone after.
6. **Submit the patch** upstream if it holds up.

Steps 4 through 6 are where the value is protected. A finding that can't be reproduced isn't a bug report, it's a guess — and a guess sent to a kernel maintainer costs credibility that takes a real patch to earn.

## Targets

Source is an unmodified snapshot; the blame commit is the exact `torvalds/linux` commit each target was taken from, verified byte-identical. **Use it when writing a `Fixes:` tag** — blame the line as of this commit, not the commit that most recently touched the function. Those give different answers.

| target | subsystem | lines | blame commit | specifications | status |
|---|---|---|---|---|---|
| `tls` | [`net/tls`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/tls) | 6,568 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 8446, 5246, 5288, 7905, 5116 | 3 — QPB run |
| `virtio` | [`drivers/virtio`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/drivers/virtio) | 16,809 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | OASIS VIRTIO v1.4 | 4 — findings under review |
| `mptcp` | [`net/mptcp`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/mptcp) | 17,295 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 8684, 6824, 8041 | 3 — QPB run |
| `smc` | [`net/smc`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/smc) | 17,807 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 7609 | 2 — requirements gathered |
| `sctp2` | [`net/sctp`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/sctp) | 43,963 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 9260, 4960, 6458, 8260, 4895, 5061, 3758 | 3 — QPB run |
| `nvme-host-docs` | [`drivers/nvme/host`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/drivers/nvme/host) | 29,338 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | NVMe Base 2.4, NVM Command Set 1.3, PCIe/RDMA/TCP transports | 4 — findings under review |

Status numbers are the steps above.

## Why the documentation step decides everything

The same NVMe subsystem, run twice, differing only in what documentation was available:

| | near-empty corpus | full spec corpus |
|---|---|---|
| requirements derived | 4 | 15 |
| citing a specification, byte-verified | 0 | 6 |
| functional areas covered | 1 | 5 |
| what a finding could say | "these two code paths disagree" | "the code violates §X of the specification" |

Without specifications every requirement is derived from the code itself, so the analysis can only ever find internal inconsistency — it has no independent standard to measure against. It's checking the code against itself.

Notably, the better-documented run also *retracted* a finding the first run had confirmed. More documentation produced fewer findings the process was willing to stand behind, which is the improvement.

## Layout

Each target holds the subsystem source plus:

- `reference_docs/cite/` — authoritative contracts, citable with byte verification
- `reference_docs/` (top level) — background context that must **not** be citable as authoritative: CVE and advisory material, git-history notes, provenance records

That split is load-bearing. Treating an advisory as an authoritative contract is a known false-positive trap; an earlier campaign scored 0/3 precision doing exactly that.

There is deliberately no `quality/` directory — runs start fresh and write their own, and results go to a `qpb-results-<date>` branch.

## Corpus inventory

Line counts are the verification. A truncated spec fetch produces citations that verify against a corrupted document, silently — so completeness is checked by counting, not by trusting the fetch.

<details>
<summary><code>tls</code> — 8 files</summary>

```
   8963  rfc8446-tls13.txt          TLS 1.3
   5827  rfc5246-tls12.txt          TLS 1.2
   1235  rfc5116-aead.txt           AEAD interface
    596  tls-offload.rst            kernel documentation
    451  rfc5288-aes-gcm.txt        AES-GCM cipher suites
    451  rfc7905-chacha20.txt       ChaCha20-Poly1305
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
    196  linux-driver-api-writing_virtio_drivers.rst
    145  linux-driver-api-virtio.rst
     42  virtio-spec-v1.4-shared-mem.txt
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

Source trees are unmodified snapshots of `torvalds/linux` at commit [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c), verified byte-identical. Specification text is redistributed verbatim from its publisher: IETF RFCs from the RFC Editor, the VIRTIO specification from OASIS, NVM Express specifications from nvmexpress.org, kernel documentation from the Linux tree.

**Nothing here is originally licensed by this repository.** Every included file remains under the license and terms of its upstream publisher, and those terms govern any use or further redistribution. The `LICENSE` file applies only to material original to this repository.
