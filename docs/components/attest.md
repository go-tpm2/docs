# attest — remote-attestation protocol

`github.com/go-tpm2/attest` is the pure-Go **TPM 2.0 remote-attestation
protocol**: a **Verifier** (control plane) and a **Node** (agent) that together
implement **node-admission-on-Quote**. A node joins a fleet only if it proves —
via a TPM **Quote** over a fresh verifier nonce, signed by an **EK-bound
Attestation Key** — that it booted an approved stack. **v0.3.0.**

It sits one layer above [`tpm2`](tpm2.md): the Verifier is pure Go and **never
touches a TPM** (`MakeCredential` + `VerifyQuote` are off-TPM crypto); the Node
drives a real `*tpm2.TPM` (`CreateEK` / `CreatePrimary` / `ActivateCredential` /
`Quote`).

```sh
go get github.com/go-tpm2/attest
```

## Protocol

Two phases. **Enrolment** (once per node) binds an AK to a trusted EK;
**Admission** (every join) proves an approved boot.

```
Node                                   Verifier
 ── EnrollRequest{EKPub,EKCert,          ──▶  Trusted(EKPub)? MakeCredential
                  AKPub,AKName}                (off-TPM) → blob+secret
 ◀─ EnrollChallenge{CredentialBlob,Secret} ──
 ActivateCredential(AK,EK,…) → secret
 ── EnrollProof{ActivationSecret}        ──▶  const-time compare → BindAK
 ─────────────────────────────────────────────────────────────────────────
 ── AdmissionRequest{AKName}             ──▶  AK bound? fresh Nonce
 ◀─ AdmissionChallenge{Nonce}            ──
 Quote(AK,pcrSel,Nonce) + PCRRead
 ── AdmissionResponse{Quoted,Signature,  ──▶  VerifyQuote (sig+pcrDigest),
                      PCRs,EventLog}            extraData==Nonce (anti-replay),
                                               Policy.Matches(PCRs) → granted
```

## Usage — the two-phase handshake

```go
// --- Verifier (control plane, pure Go, no TPM). ---
reg := attest.NewMemRegistry()
reg.TrustEK(ekPub)                        // or chain the EK cert to a root
v := attest.NewVerifier(reg, attest.GoldenPolicy{ /* set after first boot */ },
    attest.RandNonce)

// --- Node (agent, on the attesting platform). ---
node, _ := attest.NewNode(tpm2.New(transport),
    []tpm2.PCRSelection{{Hash: uint16(common.AlgSHA256), PCRs: []int{0, 7}}})

// Enrolment: bind the AK to a trusted EK.
chal, _ := v.Enroll(node.EnrollRequest(ekCert))
proof, _ := node.RespondEnroll(chal)      // TPM2_ActivateCredential
if err := v.CompleteEnroll(node.AKName(), proof); err != nil {
    log.Fatal(err)                        // attest.ErrActivationFailed
}

// Admission: prove an approved boot on each join.
adChal, _ := v.Challenge(node.AdmissionRequest())
resp, _ := node.RespondAdmission(adChal)  // TPM2_Quote + PCR_Read
granted, err := v.Admit(node.AKName(), resp)
```

`Admit`'s `err` is one of `ErrUnboundAK`, `ErrStaleNonce`, `ErrQuoteSignature`,
`ErrPCRDigestMismatch`, `ErrUntrustedBoot` (`errors.As → *UntrustedBootError`),
`ErrEventLogMismatch`, or `ErrUnapprovedMeasurement` (`errors.As →
*UnapprovedMeasurementError`) when an `EventLogPolicy` is installed.

## Policies

Pluggable `EKRegistry`, `Policy`, and `PendingStore`:

- **`GoldenPolicy`** — match against a golden whole-PCR digest.
- **`EventLogPolicy`** (v0.2.0) — replay the TCG event log against a
  per-measurement allowlist, approving individual measurements instead of a
  whole-PCR digest. The event log is fetched via
  [`efitcg2`](efitcg2.md)'s `GetEventLog`.
- **`PendingStore`** — in-memory, or a shared backend for **HA** multi-replica
  control-planes.

## Conventions

Pure Go, `CGO_ENABLED=0`, big-endian TPM wire (via `common`), `GOWORK=off`,
BSD-3-Clause on every file, 100% statement coverage. End-to-end proven against
real swtpm by [`validate`](validate.md) (`cmd/attestvalidate`,
`cmd/attesteventlog`).
