# go-tpm2

**Pure-Go, transport-agnostic TPM 2.0 stack.**

`go-tpm2` is a family of Go modules that implement a
[TPM 2.0](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
stack — guest/firmware-side drivers + the command API — in pure Go, with
no cgo and no kernel. One narrow `Transport` interface carries every
command, so the same command layer drives a TPM over TIS/FIFO MMIO, over
the CRB command buffer, or over `EFI_TCG2_PROTOCOL` under UEFI Boot
Services — swap the backplane, keep the code.

The point: one reusable, well-tested, transport-pluggable TPM 2.0 stack
instead of the cgo-bound `tpm2-tss` shim or a kernel `/dev/tpm0`
dependency. Spec-traceable to the TCG
[*TPM 2.0 Library*](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
(Parts 1–4), the
[*PC Client Platform TPM Profile (PTP)*](https://trustedcomputinggroup.org/resource/pc-client-platform-tpm-profile-ptp-specification/),
and the
[*EK Credential Profile*](https://trustedcomputinggroup.org/resource/tcg-ek-credential-profile-for-tpm-family-2-0/).
Built for the [cloud-boot](https://github.com/cloud-boot) measured-boot
loader, but designed to run anywhere a TPM does — and every command is
validated end-to-end against a **real** software-TPM and real UEFI
firmware (see [Validation](#validation)).

## Repositories

| Repo | Latest | Role |
| --- | --- | --- |
| [`common`](https://github.com/go-tpm2/common) | v0.1.0 | The narrow waist: the `Transport` interface (`Send(cmd)→rsp`), the `Regs` MMIO accessor the MMIO transports use, the **big-endian** TPM2 wire codec, and the shared constants. Everything else plugs in. |
| [`tis`](https://github.com/go-tpm2/tis) | v0.1.0 | TPM **TIS/FIFO** transport — the classic `0xFED4_0000` MMIO interface. `Regs`-driven `STS`/`DATA_FIFO` state machine; `locality`, `Expect`/`dataAvail` handshake. |
| [`crb`](https://github.com/go-tpm2/crb) | v0.1.0 | TPM **CRB** transport — the Command/Response Buffer MMIO interface (PTP §6). Doorbell `Start` + `goIdle`/`cmdReady` state machine over the command/response buffers. |
| [`efitcg2`](https://github.com/go-tpm2/efitcg2) | v0.2.0 | **`EFI_TCG2_PROTOCOL`** transport — a firmware TPM under UEFI Boot Services. Reaches the protocol through an injected firmware-call closure, so `go-tpm2` stays UEFI-free. `SubmitCommand` + `HashLogExtendEvent` for measured boot, and **`GetEventLog`** to fetch the firmware-maintained TCG event log (feeds `attest`'s `EventLogPolicy`). |
| [`tpm2`](https://github.com/go-tpm2/tpm2) | v0.5.0 | The command API over any `Transport`: Startup, GetRandom, PCR_Read / PCR_Extend, GetCapability (+typed decode), SelfTest; CreatePrimary + an ECC-P256 **Attestation Key**; **remote attestation** (Quote → VerifyQuote); **PolicyPCR sealing**; NV storage; the **Endorsement Key** (EK Credential Profile); and **credential activation** (MakeCredential / ActivateCredential). |
| [`attest`](https://github.com/go-tpm2/attest) | v0.2.2 | Control-plane **remote-attestation protocol** over `tpm2`: a pure-Go **Verifier** + a **Node** agent implementing **node-admission-on-Quote**. A node joins a fleet only if it proves, via a `Quote` over a fresh nonce signed by an **EK-bound AK**, that it booted an approved stack. Two-phase handshake — enrollment (`MakeCredential` / `ActivateCredential` binds the AK to the enrolled EK) then admission (nonce → `Quote` → verify signature, nonce, `pcrDigest`, policy). Pluggable `EKRegistry` + `Policy` — golden PCR digests **or `EventLogPolicy`** (TCG event-log replay against a per-measurement allowlist). |
| [`validate`](https://github.com/go-tpm2/validate) | v0.8.0 | Real-TPM validation harness. Drives a real `swtpm` over the TIS and CRB transports from a TamaGo+QEMU guest, runs the `EFI_TCG2` measured-boot loop on real x86 **OVMF** firmware, and exercises the `attest` protocol end-to-end (`cmd/attestvalidate`, `cmd/attesteventlog`) — **10 swtpm harnesses** in all. |

## How the pieces fit

```
     go-tpm2/attest  — control-plane protocol (Verifier + Node)
   fleet admission : node joins only on a verified Quote (EK-bound AK)
                            │  consumes tpm2
   Startup · GetRandom · PCR_Read/Extend · GetCapability · SelfTest
   CreatePrimary · AK (ECC-P256) · Quote→Verify · PolicyPCR seal/unseal
   NV · EK (EK Credential Profile) · MakeCredential/ActivateCredential
                           go-tpm2/tpm2  — command API
                            │
              ┌─────────────▼──────────────┐
              │       go-tpm2/common        │                  the narrow waist
              │  Transport (Send→rsp) · Regs (MMIO) ·         │
              │  BIG-ENDIAN TPM2 codec · constants            │
              └─────────────┬──────────────┘
                            │  Transport
   ┌──────────────┬─────────┴──────────┬─────────────────────┐
   │     tis      │        crb         │       efitcg2        │  three transports
   │  TIS/FIFO    │   CRB command      │  EFI_TCG2_PROTOCOL   │
   │   MMIO       │   buffer MMIO      │  (UEFI Boot Services) │
   └──────────────┴────────────────────┴─────────────────────┘
```

The `tpm2` command layer consumes `common.Transport` and nothing else.
The two MMIO transports (`tis`, `crb`) sit on `common.Regs`; `efitcg2`
reaches a firmware TPM through an injected closure. One command API, three
backplanes.

## Three transports, one API

This is the highlight — the same TPM 2.0 command layer over three
completely different backplanes:

- **`tis` — TIS / FIFO MMIO.** The classic TPM Interface Specification:
  a `STS`/`DATA_FIFO` register file at a fixed MMIO base, driven through
  `common.Regs`. Byte-at-a-time FIFO with the `Expect` / `dataAvail`
  handshake and per-locality access.
- **`crb` — CRB command buffer MMIO.** The PTP Command/Response Buffer
  interface: ring the doorbell (`Start`), DMA the command into the
  command buffer, read the response back out. A `goIdle` / `cmdReady`
  state machine, no byte-FIFO.
- **`efitcg2` — EFI_TCG2_PROTOCOL.** A firmware TPM exposed by UEFI Boot
  Services. `go-tpm2` itself stays UEFI-free: the caller injects a
  firmware-call closure, and `efitcg2` issues `SubmitCommand` and
  `HashLogExtendEvent` through it. This is the measured-boot path —
  extending a PCR *and* appending the matching TCG event-log record in
  one firmware call.

## The attestation story

`go-tpm2/tpm2` ships the **full remote-attestation chain**, end to end,
plus local sealing — not a toy subset:

- **Foundation.** Startup, GetRandom, SelfTest, PCR_Read / PCR_Extend,
  and GetCapability with typed decode of the capability blobs.
- **Keys.** CreatePrimary, an ECC-P256 **Attestation Key (AK)**, and the
  **Endorsement Key (EK)** per the TCG EK Credential Profile.
- **Credential activation.** Off-TPM `MakeCredential` +
  `TPM2_ActivateCredential` proves the AK and the EK live on the **same
  TPM** — the binding a verifier needs before it will trust a quote.
- **Quote → Verify.** `Quote` signs a `TPMS_ATTEST` over a PCR selection;
  `VerifyQuote` checks the ECDSA-P256 signature **and** re-derives and
  compares the `pcrDigest`. Remote attestation, closed.
- **PolicyPCR sealing.** Seal a secret to a PCR state and unseal it
  **only when the PCRs still match** — the measured-boot payoff, on the
  same TPM, locally.
- **NV storage** for persistent indices.

Put together: **EK → AK → ActivateCredential → PCR_Extend → Quote →
verify**, plus PolicyPCR seal/unseal. The whole loop, in pure Go.

That exact chain — EK → AK → ActivateCredential → Quote → verify — is
packaged as a ready-to-use protocol in [`attest`](https://github.com/go-tpm2/attest)
(a **Verifier** + a **Node** agent): a control plane admits a node to a
fleet *only* on a verified `Quote` over a fresh nonce from an EK-bound
AK, against a golden PCR policy. The "fleet admission gate" — first
consumer being [weft](https://github.com/openweft) node admission and
sealed-secret release.

## Validation

A TPM stack that only passes its own unit tests has proven only that it
is self-consistent. `go-tpm2/validate` proves the wire is right against
**real** implementations.

Every `tpm2` command is validated end-to-end against a real
[`swtpm`](https://github.com/stefanberger/swtpm) **0.10.1**, over **both**
the `tis` and `crb` transports — a bare-metal
[TamaGo](https://github.com/usbarmory/tamago) guest under QEMU drives the
real software-TPM and round-trips every command in the API: GetRandom,
PCR_Extend/Read, the full EK→AK→ActivateCredential→Quote→verify chain,
and PolicyPCR seal/unseal.

The `efitcg2` measured-boot loop is validated on **real x86 firmware** —
Fedora `OVMF.stateless.fd` + a `tpm-crb` device + `swtpm`. The loader
extends **PCR4** via `efitcg2.HashLogExtendEvent` and the extension is
**confirmed in the firmware DEBUG log**, not just in guest memory.

The running theme — the same discipline that makes
[go-virtio](https://github.com/go-virtio) catch 9 encoding bugs against
real renderers — is that **real emulators and real firmware see what a
self-consistent fake cannot**. Here it caught **14 real bugs**, the
go-tpm2 five being the sharpest:

- the CRB **data buffer is 3968 bytes, not 4096** (= `0x1000 − 0x80`,
  the buffer offset) — a fake never noticed the missing header;
- TIS `stsValid` qualifies **only** the `Expect` / `dataAvail` bits,
  nothing else;
- `TPM_RC_POLICY_FAIL` is `0x09D` — wrong-PCR unseal must fail with
  *that* code;
- `GetCapability(CAP_PCRS)` needs a **non-zero `propertyCount`** or real
  firmware returns nothing;
- the `efitcg2` output block must be **≤ the CRB `MaxResponseSize` 3968**
  — the same 3968, found a *second* time, on real firmware.

Real hardware sees what a fake cannot.

## Project standards

- **Pure Go.** `CGO_ENABLED=0` across the org. No `tpm2-tss` shim, no
  `/dev/tpm0`, no kernel dependency.
- **100% statement coverage** is the bar — error branches, not just
  happy paths. Measured, not asserted.
- **Spec-traceable.** Every register, command field, and return code
  cites the TCG TPM 2.0 Library, PTP, or EK Credential Profile section
  it implements.
- **Transport-pluggable.** One command API over TIS MMIO, CRB MMIO, and
  EFI_TCG2 — proven on a real `swtpm` and real OVMF.
- **Multi-arch.** Pure-Go code, no architecture-specific assembly. Same
  source on amd64, arm64, riscv64, loongarch64.
- **BSD-3-Clause** on every source file.

## Who uses it

The first consumer is [cloud-boot](https://github.com/cloud-boot)
measured boot — the TamaGo + UEFI loader measures each loaded image into
a PCR via `efitcg2` as it boots. Beyond that:
[weft](https://github.com/openweft) microVM **attestation** — its control
plane uses `attest` for **node admission** and **sealed-secret release**,
admitting a node only on a verified `Quote` — plus **measured boot** of
firmware payloads and **TamaGo secure boot** — any Go project that needs
a TPM without dragging in cgo or a kernel.

## Landing page

Project landing page: <https://go-tpm2.github.io>.
