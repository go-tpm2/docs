# tpm2 — the command API

`github.com/go-tpm2/tpm2` is the transport-agnostic, pure-Go **TPM 2.0 command
API**. It sits one layer above [`common`](common.md): `common` owns the
big-endian wire codec and the `Transport` contract, and this package turns that
codec into typed TPM 2.0 commands — from `Startup` / `GetRandom` / PCR ops up
through full measured-boot attestation. **v0.6.0.**

```sh
go get github.com/go-tpm2/tpm2
```

A `TPM` wraps any [`common.Transport`](common.md) — a `*crb.CRB`, a `*tis.TIS`,
an `*efitcg2` caller, or a `*devtpm.Transport`:

```go
tpm := tpm2.New(transport) // transport implements common.Transport

if err := tpm.Startup(uint16(common.SUClear)); err != nil { /* ... */ }

rnd, err := tpm.GetRandom(20)

sel := []tpm2.PCRSelection{{Hash: uint16(common.AlgSHA256), PCRs: []int{0, 7}}}
_, digests, err := tpm.PCRRead(sel)
err = tpm.PCRExtend(0, uint16(common.AlgSHA256), eventDigest)
```

## What it ships — the full attestation chain

### Attestation (Quote + off-TPM verify)

```go
ak, akPub, _ := tpm.CreatePrimary()                  // ECDSA-P256 AK
quoted, sig, _ := tpm.Quote(ak, nonce, sel)          // TPM2_Quote
info, err := tpm2.VerifyQuote(akPub, quoted, sig, expectedPCRs) // off-TPM
```

`VerifyQuote` checks the ECDSA-P256 signature **and** re-derives and compares the
`pcrDigest`.

### Seal a secret to PCR state (measured-boot payoff)

```go
parent, _ := tpm.CreateStoragePrimary()
priv, pub, _, _ := tpm.SealToPCR(parent, secret, sel, pcrValues)
// …later, unseals ONLY if the PCRs still hold pcrValues:
got, err := tpm.UnsealWithPCR(parent, priv, pub, sel, nonceCaller)
```

### Credential activation (attestation identity)

```go
ek, ekPub, _ := tpm.CreateEK()                        // EK Credential Profile
res, _ := tpm2.MakeCredential(ekPub, akName, secret, rand.Reader) // off-TPM
recovered, err := tpm.ActivateCredential(ak, ek, session,
    res.CredentialBlob, res.Secret)                   // TPM2_ActivateCredential
```

`MakeCredential` + `TPM2_ActivateCredential` prove the AK and the EK live on the
**same TPM** — the binding a verifier needs before it will trust a quote.

### Import / WrapToPCR

Wrap an external secret (off-TPM) to a node's storage key + a PCR policy so the
node imports + unseals it **only when its boot PCRs match** — control-plane
secret sealing for attested nodes.

### Other groups

- **NV storage** — `NVDefineSpace` / `NVWrite` / `NVRead` / `NVReadPublic` /
  `NVUndefineSpace`.
- **GetCapability** — typed decoders: `GetPCRBanks`, `GetTPMProperties`,
  `GetManufacturer`, `GetAlgorithms`, `GetHandles`.
- **Foundation** — Startup, GetRandom, SelfTest, PCR_Read / PCR_Extend.

Each method marshals its parameters with `common.BuildCommand` (the right
`TPM_ST` tag + `TPM_CC`), calls `Transport.Send`, and parses with `common`'s
codec. Every flow above is proven against a real swtpm by
[`validate`](validate.md).

## Conventions

Pure Go, `CGO_ENABLED=0`, big-endian TPM wire (via `common`), `GOWORK=off`,
BSD-3-Clause on every file, 100% statement coverage.
