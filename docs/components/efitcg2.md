# efitcg2 — EFI_TCG2_PROTOCOL transport

`github.com/go-tpm2/efitcg2` is a pure-Go TPM 2.0 transport backed by the UEFI
**`EFI_TCG2_PROTOCOL`**. It implements [`common.Transport`](common.md) so the
[`tpm2`](tpm2.md) command layer (Quote, GetCapability, seal/unseal, …) can drive
a **firmware** TPM while the machine is still in **Boot Services** — for example
a TamaGo + UEFI loader running under OVMF, where the TPM is owned by firmware and
raw TIS/CRB MMIO is the wrong interface. The protocol is the TCG **EFI Protocol
Specification, Family 2.0**, clause *`EFI_TCG2_PROTOCOL`*. **v0.2.0.**

!!! note "Firmware-validated"
    The cloud-boot loader extends PCR4 through this transport's `MeasureToPCR`
    (`HashLogExtendEvent`) on real x86 OVMF firmware (Fedora
    `OVMF.stateless.fd` + `tpm-crb` + swtpm), confirmed in the firmware DEBUG
    log. v0.1.1 fixed the `SubmitCommand` output block to respect the
    firmware's `MaxResponseSize` (the CRB ceiling is `3968 = 0x1000−0x80`, not
    4096) — found by that real-firmware run.

## Why not TIS/CRB here?

Under a pre-`ExitBootServices` loader the firmware already owns the TPM:
localities, active PCR banks, and the TCG event log are established, and firmware
is still servicing the device. `EFI_TCG2_PROTOCOL.SubmitCommand` is the
supported path for a marshaled TPM 2.0 command, and `HashLogExtendEvent` is the
supported measured-boot primitive (it extends a PCR **and** appends an event-log
record in one firmware call). Poking TIS/CRB registers directly at that point
races firmware and is incorrect.

## Dependency discipline

**go-tpm2 stays UEFI/TamaGo-dependency-free.** This package never imports an EFI
runtime, never dereferences a protocol pointer, and never uses package `unsafe`.
It takes an **injected** firmware-call mechanism — a `Caller` the loader
implements — exactly the way [`tis`](tis.md) and [`crb`](crb.md) take an injected
`common.Regs`. The loader owns the protocol pointer and the EFI calling
convention; `efitcg2` only marshals buffers and interprets results.

```go
// Caller is the entire UEFI-facing surface of efitcg2. The loader holds the
// EFI_TCG2_PROTOCOL pointer + the platform efiCall and performs the actual
// firmware invocation; efitcg2 hands it marshaled buffers and reads the
// returned EFI_STATUS.
type Caller interface {
    SubmitCommand(inputBlock []byte, output []byte) (status uintptr, err error)
    HashLogExtendEvent(flags uint64, dataToHash []byte, event []byte) (status uintptr, err error)
}
```

`status` is the raw `EFI_STATUS` as a `uintptr`, so the platform's native word
width carries the high "error" bit unchanged. `efitcg2` maps it:
`EFI_SUCCESS → nil`, and `EFI_INVALID_PARAMETER` / `EFI_BUFFER_TOO_SMALL` /
`EFI_DEVICE_ERROR` / `EFI_NOT_FOUND` / any-other-error to typed sentinels.

## Event log (v0.2.0)

`GetEventLog` fetches the **firmware-maintained** TCG event log (via the optional
`EventLogCaller`) so [`attest`](attest.md) can `ParseEventLog` + `ReplayPCRs` it
and confirm the replay matches the firmware's PCRs — the attestation payoff of a
measured boot. Note the event log is **little-endian**, unlike the big-endian TPM
command wire.

## Conventions

Pure Go, `CGO_ENABLED=0`, no `unsafe`, big-endian TPM wire (via `common`),
`GOWORK=off`, BSD-3-Clause on every file, 100% statement coverage.
