# Components

`go-tpm2` is a set of dependency-light Go modules (standard library only,
`CGO_ENABLED=0`, `GOWORK=off`) layered around one narrow transport contract.
Build order is **common-first**: `common` depends on no sibling; everything else
imports it.

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`common`](common.md) | `github.com/go-tpm2/common` | waist | The `Transport` (`Send(cmd)→rsp`) and `Regs` (MMIO) interfaces, the **big-endian** TPM 2.0 wire codec (`BuildCommand`/`ParseResponse`), and the spec-derived constants. |
| [`tis`](tis.md) | `github.com/go-tpm2/tis` | transport | TPM TIS/FIFO MMIO transport over `common.Regs`: `STS`/`DATA_FIFO`, `burstCount`, the `Expect` bit, per-locality access. |
| [`crb`](crb.md) | `github.com/go-tpm2/crb` | transport | TPM CRB (Command Response Buffer) MMIO transport over `common.Regs`: the PTP doorbell + `goIdle`/`cmdReady` state machine. |
| [`efitcg2`](efitcg2.md) | `github.com/go-tpm2/efitcg2` | transport | `EFI_TCG2_PROTOCOL` transport via an injected `Caller`: `SubmitCommand`, `HashLogExtendEvent` (measured boot), and `GetEventLog`. |
| [`devtpm`](devtpm.md) | `github.com/go-tpm2/devtpm` | transport | Transport over the Linux kernel TPM char device `/dev/tpmrm0` (resource-manager) — one write = one command, one read = the response. |
| [`tpm2`](tpm2.md) | `github.com/go-tpm2/tpm2` | command API | Typed TPM 2.0 commands over any `Transport`: Startup, GetRandom, PCR, GetCapability, NV, AK/EK, Quote→Verify, seal/unseal, MakeCredential/ActivateCredential, Import/WrapToPCR. |
| [`attest`](attest.md) | `github.com/go-tpm2/attest` | protocol | Remote-attestation protocol over `tpm2`: a Verifier + a Node implementing node-admission-on-Quote, with golden-PCR or event-log replay policies. |
| [`validate`](validate.md) | `github.com/go-tpm2/validate` | harness | TamaGo + QEMU + live `swtpm` harnesses that prove the transports, the command API, and the attest protocol against a real TPM. |

The two MMIO transports (`tis`, `crb`) drive `common.Regs`; `efitcg2` reaches a
firmware TPM through an injected closure; `devtpm` rides a Linux char device.
The [`tpm2`](tpm2.md) command layer consumes `common.Transport` and nothing
else, so it runs unchanged over any of the four.
