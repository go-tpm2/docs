# go-tpm2

**Pure-Go, transport-agnostic TPM 2.0 stack** — no cgo.

`go-tpm2` is a family of Go modules that implement a
[TPM 2.0](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
stack — guest / firmware / node-side drivers plus the command API — in pure
Go. One narrow [`Transport`](components/common.md) interface
(`Send(cmd) → rsp`) carries every command, so the **same command layer** drives
a TPM over TIS/FIFO MMIO, over the CRB command buffer, over
`EFI_TCG2_PROTOCOL` under UEFI Boot Services, or over the Linux kernel
resource-manager char device `/dev/tpmrm0`. Swap the backplane, keep the code.

The point: one reusable, transport-pluggable TPM 2.0 stack instead of the
cgo-bound `tpm2-tss` shim. It is built for the
[cloud-boot](https://github.com/cloud-boot) measured-boot loader and weft
remote attestation, but designed to run anywhere a TPM does — and every command
is validated end-to-end against a **real** software-TPM and real UEFI firmware
(see [`validate`](components/validate.md)).

## The narrow waist, and a big-endian codec

Everything pivots on [`common`](components/common.md): the `Transport`
interface, the `Regs` MMIO accessor the register-level drivers use, the
spec-derived constants, and the **big-endian** TPM 2.0 wire codec. The TPM 2.0
command/response byte stream is big-endian per the TCG *TPM 2.0 Library, Part 1*
— `common` owns that encoding so no other module byte-swaps payloads. (The MMIO
**control registers** of TIS/CRB are little-endian, accessed at native width
through `Regs`; and the TCG **event log** parsed by
[`efitcg2`](components/efitcg2.md) is little-endian — two deliberate exceptions
to the big-endian wire.)

Build order is **common-first**: `common` has no siblings as dependencies; the
transports (`tis`, `crb`, `efitcg2`, `devtpm`) and the command layer (`tpm2`)
import it; `attest` sits above `tpm2`; `validate` exercises the whole stack.

## How the pieces fit

```
   go-tpm2/attest   — control-plane protocol (Verifier + Node)
                      node joins only on a verified Quote (EK-bound AK)
                            │  consumes tpm2
   go-tpm2/tpm2     — TPM 2.0 command API
                      Startup · GetRandom · PCR · GetCapability · NV ·
                      AK/EK · Quote→Verify · seal/unseal · Import/WrapToPCR
                            │  consumes common.Transport
              ┌─────────────▼──────────────┐
              │       go-tpm2/common        │   the narrow waist
              │  Transport (Send→rsp) · Regs (MMIO) ·
              │  BIG-ENDIAN TPM2 codec · constants
              └─────────────┬──────────────┘
                            │  Transport
   ┌──────────┬──────────┬──────────────────┬────────────────────┐
   │   tis    │   crb    │     efitcg2      │      devtpm        │
   │ TIS/FIFO │ CRB cmd  │ EFI_TCG2_PROTOCOL│  /dev/tpmrm0       │
   │  MMIO    │ buf MMIO │ (UEFI Boot Svcs) │ (Linux RM chardev) │
   └──────────┴──────────┴──────────────────┴────────────────────┘
```

The `tpm2` command layer consumes `common.Transport` and nothing else. The two
MMIO transports (`tis`, `crb`) sit on `common.Regs`; `efitcg2` reaches a
firmware TPM through an injected closure; `devtpm` writes commands to and reads
responses from the Linux kernel resource-manager char device. One command API,
four backplanes.

## Components

| Module | Import path | Role |
|--------|-------------|------|
| [`common`](components/common.md) | `github.com/go-tpm2/common` | The narrow waist — `Transport` + `Regs` interfaces, the **big-endian** TPM 2.0 wire codec, and the spec-derived constants. |
| [`tis`](components/tis.md) | `github.com/go-tpm2/tis` | TPM **TIS/FIFO** MMIO transport — `STS`/`DATA_FIFO` handshake with `burstCount` and the `Expect` bit. |
| [`crb`](components/crb.md) | `github.com/go-tpm2/crb` | TPM **CRB** MMIO transport — the PTP Command/Response Buffer doorbell + `goIdle`/`cmdReady` state machine. |
| [`efitcg2`](components/efitcg2.md) | `github.com/go-tpm2/efitcg2` | **`EFI_TCG2_PROTOCOL`** transport — a firmware TPM under UEFI Boot Services via an injected `Caller`; `GetEventLog` for measured boot. |
| [`devtpm`](components/devtpm.md) | `github.com/go-tpm2/devtpm` | Transport over the Linux kernel TPM char device **`/dev/tpmrm0`** — the node-side host-TPM path. |
| [`tpm2`](components/tpm2.md) | `github.com/go-tpm2/tpm2` | The TPM 2.0 **command API** over any `Transport`: PCR, NV, AK/EK, Quote→Verify, seal/unseal, MakeCredential/ActivateCredential, Import/WrapToPCR. |
| [`attest`](components/attest.md) | `github.com/go-tpm2/attest` | Control-plane **remote-attestation protocol** — a Verifier + a Node implementing node-admission-on-Quote, with golden-PCR or event-log replay policies. |
| [`validate`](components/validate.md) | `github.com/go-tpm2/validate` | Real-hardware **validation harness** — TamaGo + QEMU + a live `swtpm` exercise the transports, the command API, and the attest protocol end to end. |

## Conventions

Pure Go, `CGO_ENABLED=0`, no architecture-specific assembly, big-endian TPM
wire (via `common`), `GOWORK=off`, BSD-3-Clause on every file, and 100%
statement coverage on the libraries. Spec-traceable to the TCG *TPM 2.0
Library* (Parts 1–4), the *PC Client Platform TPM Profile (PTP)*, the TCG *EFI
Protocol Specification, Family 2.0*, and the *EK Credential Profile*.
