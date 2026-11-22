---
title: "nvcounter — TPM monotonic counter"
weight: 90
---

# nvcounter — TPM monotonic counter {#nvcounter-tpm-monotonic-counter}

`github.com/go-tpm2/nvcounter` is a **monotonic counter on a TPM 2.0**: a
`TPM_NT_COUNTER` NV index (TCG TPM 2.0 Library Part 3, 31.8) that is created
once, incremented under its own authValue and never goes back. It is a rollback
guard for anything that must refuse an older copy of itself;
[theca](https://github.com/go-theca/theca) binds a container to one. Moved here
from theca's `tpmcounter` package in **v0.1.0**.

```sh
go get github.com/go-tpm2/nvcounter
```

```go
import "github.com/go-tpm2/nvcounter"

c, err := nvcounter.Create(nvcounter.Options{Auth: secret}) // defines a free owner index
err = c.Increment()
v, err := c.Read()
id, err := c.Identity() // names THIS TPM: the digest of its ECC P-256 EK

c, err = nvcounter.Open(nvcounter.Options{Handle: c.Handle(), Auth: secret})
```

## Why a module of its own {#why-a-module-of-its-own}

[`tpm2`]({{< relref "tpm2.md" >}}) depends on `common` alone. The counter opens
devices, so it needs [`devtpm`]({{< relref "devtpm.md" >}}) and through it
`golang.org/x/sys`. Keeping it out of `tpm2` leaves the command layer without a
transport dependency; the primitives (`NVDefineCounter`, `NVIncrementAuth`,
`NVReadCounter`, `EKIdentityAuth`) stay in `tpm2`.

## Behaviour {#behaviour}

- A `Counter` holds no connection: each operation opens the device and closes it.
- Transports: `/dev/tpmrm0` (Linux, the default), Windows TBS (`tbs:`, the
  Windows default) and `unix:<socket>` for a software TPM. macOS has no TPM:
  `ErrUnsupported` unless a device is named.
- New counters carry `TPMA_NV_NO_DA`; a wrong authValue does not lock the TPM.
- The Name of a counter changes once, at its first increment (it covers
  `TPMA_NV_WRITTEN`); an index nobody incremented cannot be read.
- `Identity` names the TPM, it does not prove the TPM answering is that one:
  whoever controls a software TPM can present any EK. It needs the endorsement
  hierarchy password in `Options.EndorsementAuth` if one is set.
- Every answer from the TPM that becomes a value is bounded (authValues at most
  32 bytes, the EK Name exactly 34 bytes starting `000b`).
- The authValue still travels in a password session; a salted HMAC session is
  tracked in [tpm2#20](https://github.com/go-tpm2/tpm2/issues/20).

Tested against a real software TPM in CI (a missing swtpm is a failure) and a
fake TPM for every error path, with 100% statement coverage. A hardware TPM was
not measured.
