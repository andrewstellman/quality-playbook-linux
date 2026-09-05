# Provenance

Source: net/vmw_vsock and the listed headers, byte-identical to torvalds/linux
commit 4d7d9486c04d917265f64c55bd23b2cc4fe7749c.

reference_docs/cite/virtio-spec-v1.4-*.txt: LaTeX source of the OASIS VIRTIO
specification v1.4. The core files (content, split-ring, packed-ring,
transport-*, admin, conformance, and so on) were copied from the `virtio` target
in this same directory. The three `virtio-spec-v1.4-vsock-*.txt` files are
`device-types/vsock/{description,device-conformance,driver-conformance}.tex`
from https://github.com/oasis-tcs/virtio-spec at tag v1.4-cs01 (commit
917e900e0246b7fe21cdde795b0e566dd4f57d8d). The `.txt` extension is so QPB's
citation tooling accepts them; the content is unmodified.

Not covered by any specification here: the VMCI transport (VMware, no public
spec) and the Hyper-V transport.
