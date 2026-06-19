# validate — real-swtpm validation harness

`github.com/go-tpm2/validate` is a real-hardware validation **harness** (not a
library) for the go-tpm2 stack. Each harness boots a
[TamaGo](https://github.com/usbarmory/tamago) `amd64` guest under QEMU and
drives a **real TPM 2.0** — QEMU's `-device tpm-crb` (or `-device tpm-tis`)
backed by a live [`swtpm`](https://github.com/stefanberger/swtpm) **0.10.1** —
through [`crb`](crb.md) / [`tis`](tis.md) (the MMIO transports) and
[`tpm2`](tpm2.md) (the command layer). The guest performs the proof itself,
prints `…: PASS` / `FAIL` on COM1, and terminates QEMU with a matching exit code
via `isa-debug-exit`; the `run-*.sh` wrapper starts swtpm, parses the verdict,
and exits non-zero on FAIL.

This is where the `// INFERRED:` register bits in [`crb`](crb.md)/[`tis`](tis.md)
and the encodings in [`tpm2`](tpm2.md) are confirmed or corrected against actual
TPM behaviour — the same live round-trip discipline used in the go-virtio work.

## Harnesses

Guest harnesses under `cmd/`, each with its `run-*.sh` wrapper:

| Harness | Transport | What it proves against real swtpm |
|---|---|---|
| `cmd/tpmvalidate` | **CRB** | `Startup → GetRandom → PCR_Extend → PCR_Read`: CRB MMIO handshake + framing + bare-index PCR handle + empty-auth session |
| `cmd/tpmvalidate-tis` | **TIS** | Same cycle over the TIS/FIFO transport (`burstCount` + `Expect` bit) |
| `cmd/tpmattest` | CRB | `Quote` over a `CreatePrimary` AK, then off-TPM `VerifyQuote` of the ECDSA-P256 signature + PCR digest |
| `cmd/tpmseal` | CRB | PolicyPCR **seal/unseal**: a secret sealed to a PCR value unseals ONLY while that PCR holds it |
| `cmd/tpmnv` | CRB | NV storage: `NVDefineSpace → NVWrite → NVRead → NVReadPublic → NVUndefineSpace` round-trips |
| `cmd/tpmcap` | CRB | Typed `GetCapability` decoders (PCR banks, properties, manufacturer, algorithms, handles) |
| `cmd/tpmek` | CRB | The EK Credential Profile ECC-P256 (L-2) template produces the expected EK |
| `cmd/tpmcred` | CRB | Credential activation: an off-TPM `MakeCredential` is recovered EXACTLY by `TPM2_ActivateCredential` — proving AK and EK share one TPM |
| `cmd/tpmimport` | CRB | The `Import` / `WrapToPCR` flow: an off-TPM secret wrapped to a storage key + PCR policy, imported and unsealed only when PCRs match |
| `cmd/attestvalidate` | CRB | The [`attest`](attest.md) two-phase protocol end to end (enrolment + admission) |
| `cmd/attesteventlog` | CRB | The `EventLogPolicy` path: replay the TCG event log against a per-measurement allowlist |

In addition, the `EFI_TCG2` measured-boot loop is exercised on real x86 **OVMF**
firmware (see [`efitcg2`](efitcg2.md)).

## Run

```sh
./run.sh            # CRB validate
./run-tis.sh        # TIS validate
./run-seal.sh       # … etc, one per harness above
```

Each script requires `swtpm`, `qemu-system-x86_64`, and the TamaGo toolchain
(path via `$TAMAGO`). The guest is built with:

```sh
GOWORK=off GOOS=tamago GOARCH=amd64 GOOSPKG=github.com/usbarmory/tamago \
  "$TAMAGO" build -ldflags "-T 0x10010000 -R 0x1000" \
  -o tpmvalidate.elf ./cmd/tpmvalidate
```

!!! note "CI is build-only"
    Hosted CI cannot run the swtpm + full-system QEMU round-trips, so the
    workflow builds the TamaGo toolchain from source (the version pinned in
    `go.mod`) and cross-builds all harnesses for tamago/amd64. The actual
    PASS/FAIL proofs run locally via `run-*.sh`.

## What the base cycle proves

- **GetRandom**: two successive 32-byte reads come back non-zero and differ.
- **PCR_Extend / PCR_Read** on the SHA-256 bank of the debug PCR (16): after the
  extend, `PCR_new == SHA256(PCR_old || digest)`, recomputed in-guest with
  `crypto/sha256`. That identity holding against the live swtpm proves the CRB
  MMIO handshake, the command framing, the bare-index PCR handle, and the
  empty-auth password session are all correct.

## Findings against the live swtpm (QEMU 10.2.2 + swtpm 0.10.1)

- `crb.regData = 0x80` (data buffer offset) — **CONFIRMED**: QEMU reports
  `CTRL_CMD_ADDR = CTRL_RSP_ADDR = 0xFED40080`, i.e. offset `0x80`.
- `crb.bufSize` — **CORRECTED 4096 → 3968**: the locality is a `0x80`-byte
  control area at `0xFED40000` followed by the data buffer at
  `0xFED40080..0xFED40FFF`, so `CTRL_CMD_SIZE = CTRL_RSP_SIZE = 3968`. The old
  `4096` was the full locality size, not the buffer, and would have admitted a
  declared response 128 bytes past the real buffer end.

**v0.9.0.** BSD-3-Clause.
