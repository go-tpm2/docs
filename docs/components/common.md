# common — the narrow waist

`github.com/go-tpm2/common` is the transport-agnostic, platform-agnostic
foundation shared by every layer of the stack. It owns the small plug-in
interfaces, the **big-endian** TPM 2.0 wire codec, and the spec-derived
constants. The command layer ([`tpm2`](tpm2.md)) sits above it; the MMIO
transports ([`tis`](tis.md), [`crb`](crb.md)) sit below it. Build it first —
nothing in the family is a dependency of `common`.

```sh
go get github.com/go-tpm2/common
```

## Two interfaces stitch the stack together

### `Transport`

The contract a concrete transport implements and the command layer consumes —
deliberately minimal:

```go
type Transport interface {
    Send(cmd []byte) (rsp []byte, err error)
}
```

The command layer marshals a complete command buffer (header + parameters, via
`BuildCommand`), hands it to `Send`, and gets the complete response buffer back
(ready for `ParseResponse`). Framing, MMIO handshakes, locality, and retry stay
inside the transport.

### `Regs`

The platform-provided MMIO accessor the register-level drivers (CRB, TIS) use to
touch the TPM register block. `off` is a byte offset within the TPM register
window.

```go
type Regs interface {
    Read8(off uint32) uint8
    Read32(off uint32) uint32
    Write8(off uint32, v uint8)
    Write32(off uint32, v uint32)
}
```

Plus byte-stream helpers for the CRB command buffer / TIS FIFO:

```go
func ReadBytes(r Regs, off uint32, p []byte)
func WriteBytes(r Regs, off uint32, p []byte)
```

## Wire codec — BIG-ENDIAN

The TPM 2.0 command/response byte stream is **big-endian** per the TCG *TPM 2.0
Library, Part 1*. `common`'s codec owns that encoding — `BuildCommand` writes
the `TPM_ST` tag, the total size, and the `TPM_CC` command code ahead of the
parameter bytes; `ParseResponse` reads the response header and returns the
response code and parameter bytes. No other module in the family byte-swaps a
payload.

The MMIO **control registers** of [`tis`](tis.md)/[`crb`](crb.md) are, by
contrast, little-endian and accessed at native width through `Regs`; and the
TCG **event log** parsed by [`efitcg2`](efitcg2.md)/[`attest`](attest.md) is
little-endian. Those are the two deliberate exceptions to the big-endian wire.

## Conventions

Pure Go, `CGO_ENABLED=0`, no architecture-specific assembly, `GOWORK=off`,
BSD-3-Clause on every file, 100% statement coverage. **v0.1.0.**
