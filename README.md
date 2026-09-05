# quality-playbook-linux

This repository stages Linux kernel subsystems for analysis with the [Quality Playbook](https://github.com/andrewstellman/quality-playbook). The approach applies traditional quality engineering, using AI to speed it up: gather the specifications a subsystem is supposed to implement, derive requirements from them, and check the code against those requirements.

We are looking for bugs where the code reads correctly on inspection but does not match the specification. Code review and static analysis do not catch them, and the existing tests usually pass because they were written from the same reading of the specification as the code. A related case is a contract with several sibling implementations where one has drifted from the others. The first patch accepted upstream from this work, a virtio-pci interrupt-return fix, was a bug of that kind.

## How this works

1. Clone the subsystem from `torvalds/linux` into its own target directory and record the commit.
2. Gather the specifications the code is supposed to conform to, using the [documentation gathering prompt](https://github.com/andrewstellman/quality-playbook/blob/main/references/DOC_GATHERING_PROMPT.md) from the Quality Playbook repository.
3. Run the Quality Playbook, all six phases plus iterations. We currently use Claude Opus.
4. Review the reported bugs and suggested patches to decide which are real.
5. Reproduce each real bug under QEMU, then apply the patch and confirm the bug is gone.
6. Submit the patch upstream.

## Targets

Each target is an unmodified snapshot of its subsystem at the listed commit. To write a `Fixes:` tag, start from that commit and blame the specific line to find the commit that introduced the behavior. Do not use the snapshot commit itself, and do not use the most recent commit that touched the function.

| target | subsystem | lines | snapshot commit | specifications | status |
|---|---|---|---|---|---|
| `tls` | [`net/tls`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/tls) | 6,568 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 8446, 5246, 5288, 7905, 5116 | 3 |
| `virtio` | [`drivers/virtio`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/drivers/virtio) | 16,809 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | OASIS VIRTIO v1.4 | 4 |
| `mptcp` | [`net/mptcp`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/mptcp) | 17,295 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 8684, 6824, 8041 | 3 |
| `smc` | [`net/smc`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/smc) | 17,807 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 7609 | 2 |
| `sctp2` | [`net/sctp`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/net/sctp) | 43,963 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | RFC 9260, 4960, 6458, 8260, 4895, 5061, 3758 | 3 |
| `nvme-host-docs` | [`drivers/nvme/host`](https://github.com/torvalds/linux/tree/4d7d9486c04d917265f64c55bd23b2cc4fe7749c/drivers/nvme/host) | 29,338 | [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c) | NVMe Base 2.4, NVM Command Set 1.3, PCIe/RDMA/TCP transports | 4 |

Line counts cover `.c` files only. Status is the step number from the list above.

Each target holds the subsystem source plus `reference_docs/cite/`, which contains the specifications. Files directly under `reference_docs/` are background material, such as CVE lists and git history, and are not specifications. Run output goes in `quality/`, which is not committed to `main`.

## Provenance and licensing

Source trees are unmodified snapshots of `torvalds/linux` at commit [`4d7d9486`](https://github.com/torvalds/linux/commit/4d7d9486c04d917265f64c55bd23b2cc4fe7749c). Specification text is redistributed verbatim from its publisher: IETF RFCs from the RFC Editor, the VIRTIO specification from OASIS, NVM Express specifications from nvmexpress.org, and kernel documentation from the Linux tree.

Nothing here is originally licensed by this repository. Every included file remains under the license and terms of its upstream publisher, and those terms govern any use or further redistribution. The `LICENSE` file applies only to material original to this repository.
