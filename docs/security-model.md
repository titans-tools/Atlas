# Security model

The source code is not published, so trust is anchored in **verifiable
distribution and verifiable behavior** rather than in reading the code.

## What you can verify before anything runs

1. **The installer is inert until verification.** It pins the Titans release
   public key (Ed25519) and the SHA-256 of the bootstrap binary in its own
   text — read it before running it.
2. **The bootstrap verifies the signed catalog.** Every artifact's URL,
   SHA-256 and signature come from `signed-distribution-catalog.json`, signed
   with the release key. A byte that does not match does not install.
3. **Manual path.** Download the archive + catalog + public key yourself,
   check the hash and signature, extract, run. No script required.
4. **SBOM.** Every release archive carries its software bill of materials.

## What you can verify while it runs

- **Loopback only.** Every network surface binds to `127.0.0.1`. There is no
  remote endpoint, no account, no license phone-home.
- **No telemetry.** Watch the process with any network monitor: the only
  outbound traffic is the update check / downloads you invoke explicitly
  against GitHub Releases.
- **Offline-capable.** After install (and optional model provisioning), the
  product works with the network cable unplugged.
- **Hash-chained audit.** Every mutation lands in an audit chain you can
  verify (`atlas_audit_verify`).

## Updates

Updates only happen through the same signed-catalog verification, and only
with explicit confirmation (`titans versions check` / `titans install`).
Versions can be pinned.

## Reporting

Vulnerabilities: see [SECURITY.md](../SECURITY.md) — private e-mail, no public
issues.
