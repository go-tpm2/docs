# crb — CRB MMIO transport

`github.com/go-tpm2/crb` is a pure-Go TPM 2.0 **Command Response Buffer (CRB)**
MMIO transport. It implements [`common.Transport`](common.md) over the
[`common.Regs`](common.md) MMIO accessor: the platform owns the register mapping
(a physical MMIO window, an `mmap` of `/dev/mem`, or a test stub) and exposes it
through `common.Regs`; this package drives the CRB register handshake defined by
the TCG **PC Client Platform TPM Profile (PTP) Specification**, clause *Command
Response Buffer Interface*. **v0.1.0.**

```sh
go get github.com/go-tpm2/crb
```

## Usage

```go
import (
    "github.com/go-tpm2/common"
    "github.com/go-tpm2/crb"
)

// r is a platform-provided common.Regs over the CRB control area base.
dev, err := crb.Open(r)
if err != nil {
    // not a CRB interface, or the interface never reported valid
}

// *crb.CRB satisfies common.Transport, so it plugs straight into go-tpm2/tpm2:
//   tpm := tpm2.New(dev)
//   tpm.Startup(uint16(common.SUClear))
//   rnd, _ := tpm.GetRandom(20)

// …or drive raw command buffers directly:
cmd := common.BuildCommand(uint16(common.TagNoSessions),
    uint32(common.CCGetRandom), []byte{0x00, 0x02})
rsp, err := dev.Send(cmd) // common.Transport.Send
```

## Send state machine (TCG PTP)

1. **Request locality** — `LOC_CTRL.requestAccess`, wait `LOC_STS.Granted` +
   `LOC_STATE.locAssigned`.
2. **Request command-ready** — `CTRL_REQ.cmdReady`, poll until the TPM leaves
   Idle (`CTRL_STS.tpmIdle` clear) with no error and `cmdReady` self-cleared.
3. **Write command** into the data buffer (`common.WriteBytes`).
4. **Start** — `CTRL_START.start = 1`.
5. **Wait** for the TPM to clear `CTRL_START.start` (bounded spin); on expiry
   drive `CTRL_CANCEL` and return `ErrTimeout`.
6. **Check** `CTRL_STS.Error`.
7. **Read** the 10-byte response header for `responseSize`, bounds-check, then
   read the whole response.
8. **Release** — `CTRL_REQ.goIdle`.

## Endianness

The TPM 2.0 **wire** protocol (the command/response byte stream) is
**big-endian** and handled by [`common`](common.md)'s codec. The CRB **control
registers** are **little-endian** MMIO; `common.Regs.Read32/Write32` access them
in the platform's native width, so this package never byte-swaps register values
— it only moves the opaque command/response byte streams through the data
buffer.

## Validated findings

[`validate`](validate.md) confirms the register layout against QEMU's
`-device tpm-crb` + a live swtpm: `regData = 0x80` (data-buffer offset) is
**confirmed**, and `bufSize` was **corrected 4096 → 3968** — the locality is a
`0x80`-byte control area at `0xFED40000` followed by the data buffer at
`0xFED40080..0xFED40FFF`, so `CTRL_CMD_SIZE = CTRL_RSP_SIZE = 3968`. The old
`4096` was the full locality size, and would have admitted a declared response
128 bytes past the real buffer end.

## Conventions

Pure Go, `CGO_ENABLED=0`, no architecture-specific assembly, big-endian TPM
wire (via `common`), `GOWORK=off`, BSD-3-Clause on every file, 100% statement
coverage.
