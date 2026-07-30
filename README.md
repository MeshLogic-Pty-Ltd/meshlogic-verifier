# MeshLogic offline proof-bundle verifier

`meshlogic-verify` is the **zero-trust, offline** verifier for MeshLogic R7 compliance-evidence proof bundles. It lets an auditor confirm that a bundle is genuine and untampered **without trusting MeshLogic** — no MeshLogic API, no network call, no MeshLogic-supplied trust store. The verifier's trust anchors are **pinned into the binary at build time**.

This repository exists so the verifier and its pinned signing-key fingerprint are obtainable and independently verifiable **from outside MeshLogic's own systems**.

## Downloads

Get the binaries from the [**Releases**](https://github.com/MeshLogic-Pty-Ltd/meshlogic-verifier/releases) page. Each release carries, per platform:

| Platform | Binary |
|---|---|
| Linux x86-64 | `meshlogic-verify-linux-x64` |
| macOS (Apple silicon) | `meshlogic-verify-macos-arm64` |
| Windows x86-64 | `meshlogic-verify-windows-x64.exe` |

…each with a `.sha256` checksum and a `.pinned-spki.b64` (the exact public signing key the binary trusts).

## Verify what you downloaded (do this first)

1. **Checksum the binary** against its published `.sha256`:
   ```
   sha256sum -c meshlogic-verify-linux-x64.sha256        # Linux
   shasum -a 256 -c meshlogic-verify-macos-arm64.sha256   # macOS
   ```
   Windows PowerShell: `Get-FileHash meshlogic-verify-windows-x64.exe -Algorithm SHA256`.
   The `.sha256` is served from the same origin as the binary, so it proves transport integrity — **not** authenticity. Authenticity comes from step 2.

2. **Confirm the pinned key is MeshLogic's real signing key.** Ask the binary which key it trusts:
   ```
   ./meshlogic-verify-linux-x64 --print-pinned-spki
   ```
   It must equal the contents of the `.pinned-spki.b64` asset **and** the fingerprint MeshLogic publishes on an independent channel (see *Independent key anchor* below). Cross-checking the key across a second channel you don't get from this repo is what closes the trust loop.

## The pinned production signing key (P-256 SPKI, base64 DER)

```
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEe0D2WWUi4WF8iJMoDO0SKsy5uygJEonf3jjvUHvsZv5A7JIA59PqQDqzCjS4TPy9LXVmj5bGOqIAibDBxZa2ug==
```

### Independent key anchor (Sigstore Rekor)

To keep the "verify without trusting us" chain honest, the pinned key is anchored on a channel **independent of MeshLogic** — the public [Sigstore](https://www.sigstore.dev/) transparency log (Rekor). At release time MeshLogic's CI **keyless-signs** the exact pinned SPKI with a short-lived [Fulcio](https://docs.sigstore.dev/certificate_authority/overview/) certificate bound to its GitHub Actions OIDC identity, and the signing event is logged to public Rekor. You confirm — against Rekor and Fulcio, which **neither you nor MeshLogic control** — that this key was published by MeshLogic's official release workflow, **without querying any MeshLogic system**.

Each release carries three anchor assets:

| Asset | What it is |
|---|---|
| `meshlogic-verify.spki.b64` | the pinned SPKI (same bytes as each binary's `.pinned-spki.b64`) |
| `meshlogic-verify.spki.sig` | the keyless signature over it |
| `meshlogic-verify.spki.pem` | the Fulcio certificate (carries the CI identity) |

**Anchor the key** ([install cosign](https://docs.sigstore.dev/system_config/installation/) first):

```
cosign verify-blob \
  --certificate meshlogic-verify.spki.pem \
  --signature   meshlogic-verify.spki.sig \
  --certificate-identity-regexp '^https://github\.com/MeshLogic-Pty-Ltd/MeshLogic-Platform-Prod/\.github/workflows/publish-verifier\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  meshlogic-verify.spki.b64
```

`Verified OK` means: this SPKI was signed by MeshLogic's official `publish-verifier` workflow (the `--certificate-identity-regexp`), the certificate was issued by GitHub's OIDC-backed Fulcio (`--certificate-oidc-issuer`), and the event is recorded in Rekor. Then confirm the anchored bytes are the key your binary trusts:

```
diff <(./meshlogic-verify-linux-x64 --print-pinned-spki) meshlogic-verify.spki.b64
```

No difference → the key the verifier is pinned to is the same key MeshLogic's CI publicly anchored. The trust loop is closed **outside** MeshLogic's account.

## Verify a proof bundle

```
./meshlogic-verify-linux-x64 <bundle.zip>
```

- Exit **0** — every record graded **PROVEN** (signature verifies against the pinned key; the Merkle commitment is anchored; an RFC 3161 timestamp and a Sigstore Rekor transparency-log witness are present and check out).
- Exit **2** — any record graded worse (`ALTERED`, `UNTRUSTED_KEY`, `NOT_YET_ANCHORED`, `NOT_YET_WITNESSED`, `MALFORMED`) or the bundle is unreadable. The report names the reason per record.
- Exit **64** — usage error.

The verifier also prints the bundle's signed **scope** (`SCOPE:` line): a `committed-only` bundle deliberately excludes not-yet-anchored evidence so its overall verdict can be a clean PROVEN; a `full-export` bundle includes everything and grades each record honestly.

## Provenance

- Builds are reproducible (`cargo build --locked --release`) and pinned to the live production signing key at release time.
- Built on GitHub-hosted runners only — never on infrastructure that holds any signing material.
- The verifier's own pinned roots — not anything read out of a bundle — are the trust anchor.
