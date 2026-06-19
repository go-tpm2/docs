# devtpm — /dev/tpmrm0 transport

`github.com/go-tpm2/devtpm` is a pure-Go TPM 2.0 transport over the **Linux
kernel TPM character device** (`/dev/tpmrm0`). It implements
[`common.Transport`](common.md) over a Linux TPM char device and is the
**node-side host-TPM transport** for weft remote attestation: on a real Linux
node it gives the [`attest`](attest.md) Node side a channel to the host's
hardware (or firmware / vTPM) TPM, where the [`tis`](tis.md)/[`crb`](crb.md)
MMIO interfaces are not available. **v0.1.0.**

```sh
go get github.com/go-tpm2/devtpm
```

## Device model

The Linux `tpm` subsystem (`drivers/char/tpm`) exposes a TPM through two
character devices, both with the same framing:

| Device | Role |
|--------|------|
| `/dev/tpmrm0` | **Resource-manager** channel (`DefaultDevice`). Each open fd is an independent, multiplexed command channel; the kernel RM virtualizes the TPM's scarce transient-object/session slots and flushes a client's context on close. **Prefer this.** |
| `/dev/tpm0` | **Raw** channel. Single-open, unmediated, no handle virtualization — a leaked transient handle can wedge the whole TPM. |

The framing this package relies on: one `write()` delivers exactly one complete
command, and the matching `read()` returns exactly one complete response. `Send`
therefore does one `Write` of the full command and one `Read` of the full
response — no length prefix, no chunking.

Because that contract is just "raw TPM2 bytes, one command per write, one
response per read", `New` also accepts any `io.ReadWriteCloser` that honors it —
most usefully a unix socket to **swtpm's `--server` data channel**, which speaks
the identical raw TPM2 protocol. The [`validate`](validate.md) harness uses
exactly that to drive this transport against a real swtpm on a host with no
`/dev/tpmrm0`.

## Usage

```go
import (
    "github.com/go-tpm2/attest"
    "github.com/go-tpm2/devtpm"
    "github.com/go-tpm2/tpm2"
)

// Open the host's resource-manager TPM channel.
dev, err := devtpm.Open(devtpm.DefaultDevice) // "/dev/tpmrm0"
if err != nil {
    // device missing, or insufficient privilege (root / tss group)
}
defer dev.Close()

// *devtpm.Transport satisfies common.Transport, so it plugs straight into the
// go-tpm2/tpm2 command layer and the attestation Node:
tpm := tpm2.New(dev)
node, err := attest.NewNode(tpm, pcrSel)
// node.Quote(nonce) … etc.
```

## Conventions

Pure Go, `CGO_ENABLED=0`, big-endian TPM wire (via `common`), `GOWORK=off`,
BSD-3-Clause on every file, 100% statement coverage.
