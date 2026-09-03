# Security Policy

## Reporting a vulnerability

Report privately by email to **lbmachado.contato@gmail.com**. Do not open a
public issue for a security problem.

Please include: what you found and why it is a security problem; the exact
Atlas version (`atlas status` prints it); your operating system and how Atlas
was installed (one-line installer, `titans install`, or manual archive); a
minimal reproduction (tool or endpoint, arguments, observed vs expected); and
whether any listener was bound to a non-loopback address.

You will get an acknowledgement of receipt. Fixes ship as a normal signed
release; the release notes name the problem and the version that fixes it.

## Supported versions

Only the latest released version receives fixes. Current line: **0.2.0**.

## Threat model

Atlas is local-first: a single executable that stores everything on the machine
it runs on and binds to loopback (`127.0.0.1:7741`) by default. The assumed
operator is the owner of that machine. Releases are Ed25519-signed and
described in a signed catalog; the installer and the Tool Manager refuse
artifacts whose signature, hash or catalog revision does not verify.
