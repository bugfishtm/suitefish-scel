# SCEL Signing System — Operational Guide

This document describes the cryptographic system used to authenticate
**Release Files** issued under the Suitefish Commercial Exception License (SCEL).
It is the canonical implementation reference for Section 7 of SCEL v1.4.

This is an **operational** document, not a legal one. The legal terms live in
`scel_1.4.md`. This document explains *how* the authentication mechanism
described there actually works in practice.

---

## 1. Purpose

When someone claims "I have a Suitefish commercial license," there must be a way
to **prove or disprove** that claim without relying on a central authority.

The system described here uses standard public-key cryptography:

- The copyright holder generates a key pair and publishes the public half.
- Every Release File is signed with the private half.
- Anyone — courts, licensees, future Suitefish staff — can verify that a given
  Release File genuinely came from the copyright holder.
- A second check against a public registry ensures that even a leaked private
  key cannot produce forged licenses with arbitrary License IDs.

This is the same model used by Debian, the Linux kernel, OpenBSD, Homebrew,
and essentially every reputable open-source distribution.

---

## 2. Why this works

### Cryptographically

A modern asymmetric signature (Ed25519, the recommended algorithm) is
mathematically unforgeable. Without the private key, an attacker cannot
produce a signature that verifies against the public key. This is true
regardless of who generated the key or where it came from — what matters is
that the copyright holder controls the private half.

### Legally (EU / Germany)

Under EU **eIDAS** regulation (Regulation 910/2014), digital signatures fall
into three tiers:

1. **Simple electronic signature** — any electronic mark of intent. GPG and
   Ed25519 signatures fall here.
2. **Advanced electronic signature** — uniquely tied to the signatory and
   under their sole control. Self-generated keys with proper handling qualify.
3. **Qualified electronic signature** — issued by a certified Trust Service
   Provider. Legally equivalent to a handwritten signature.

A self-generated signature is at least Tier 1 and, with sound key hygiene,
Tier 2. Under eIDAS Article 25, such signatures **cannot be denied legal
effect or admissibility as evidence solely on the grounds of being
electronic**.

For SCEL enforcement — where the question is "did you actually issue this
Release File?" — Tier 1/2 is sufficient. A court will accept a verifiable
signature against a long-published public key as compelling evidence of
authenticity.

### Where the burden falls

This is the key practical point: **the claimant must produce a verifiable
Release File**, not the copyright holder. If someone says they have a
license but cannot show a Release File that:

- verifies against the published public key, AND
- matches an entry in the public registry,

then by Section 7(d) of SCEL, they have no rights under the exception. The
burden of proof sits with whoever is invoking the exception.

---

## 3. Tool choice

**Recommended: `minisign`**

- Ed25519, ~32-byte keys
- One command to sign, one to verify
- No web-of-trust, no expiring subkeys, no confusing options
- Available on every major platform
- Created by the libsodium author

**Alternatives:**

| Tool | Algorithm | Notes |
|------|-----------|-------|
| `signify` | Ed25519 | OpenBSD's tool, used for their releases |
| `ssh-keygen -Y sign` | Ed25519 | Reuses your existing SSH key infrastructure |
| GPG | RSA-4096 or Ed25519 | Universal but UX-heavy; only choose if you already use it |

All commands and examples in this document use `minisign`. Adapt as needed if
you choose differently.

> **Important:** Whichever tool you choose, do not switch later without
> publishing a key transition document. Verifiers cache and trust your
> published public key — silent changes break their ability to verify.

---

## 4. One-time setup

### 4.1 Generate the signing key

```bash
minisign -G -p suitefish_scel.pub -s suitefish_scel.key
```

You will be prompted for a passphrase. **Choose a strong one** — this
passphrase is the last line of defense if the private key file is ever
stolen.

This produces two files:

- `suitefish_scel.pub` — public key. Publish this.
- `suitefish_scel.key` — private key. Protect this with your life.

### 4.2 Publish the public key

Host the public key at a **stable, permanent URL** under your control. For
example:

```
https://suitefish.example/scel/pubkey.minisign
```

Best practices:

- Publish under HTTPS with a certificate from a reputable CA.
- Mention the URL in your README, your website footer, and in the SCEL
  documents you issue.
- Once published, **never change the URL**. Verifiers depend on it being
  stable across years.
- Consider mirroring the key on a second domain or a public archive (e.g.
  archive.org) as evidence of when it was first published.

### 4.3 Initialize the registry

Create an empty registry file:

```json
{
  "scel_registry_version": 1,
  "issuer_pubkey_fingerprint": "RWQf6LRCGA9i...",
  "issued_at": "2026-05-28T00:00:00Z",
  "licenses": {}
}
```

Host it at a stable URL:

```
https://suitefish.example/scel/registry.json
```

Sign the registry file itself:

```bash
minisign -S -s suitefish_scel.key -m registry.json
```

This produces `registry.json.minisig`. Host both files side by side.

### 4.4 Secure the private key

The private key file `suitefish_scel.key` must be kept offline.

**Recommended setups, in order of preference:**

1. **YubiKey or hardware token** — the key never touches a connected
   computer. Even malware on your daily-driver machine cannot extract it.
2. **Air-gapped machine** — a dedicated device that never connects to the
   internet. Transfer Release Files in and signatures out via USB.
3. **Encrypted external drive** stored physically separately from your
   main machine. Only mount when signing.

**Never acceptable:**

- Storing the key on your daily laptop, even encrypted.
- Storing the key in cloud storage (Dropbox, iCloud, Google Drive).
- Committing the key to any git repository, public or private.
- Storing the key in a CI/CD secret manager.

### 4.5 Create a backup

Generate **at least one** backup of the private key:

- Encrypted with the same passphrase
- Stored in a separate physical location (e.g. a safe deposit box, a
  trusted family member's safe, an encrypted USB stick locked away)

Without a backup, losing the key means you can never issue another valid
Release File or update the registry — every existing license becomes
permanently frozen at its current state.

---

## 5. Issuing a license

For each new licensee:

### 5.1 Assemble the Release File

Create a JSON document, for example `release_SCEL-2026-0001.json`:

```json
{
  "scel_version": "1.4",
  "license_id": "SCEL-2026-0001",
  "licensee": "ACME GmbH",
  "licensee_fingerprint": "sha256:b94d27b9934d3e08...",
  "approval_codename": "BLUEFIN",
  "approved_project": "ACME Customer Portal",
  "issued_at": "2026-05-28T12:00:00Z",
  "designated_folders": [
    "/_data/modules/acme_portal",
    "/_data/themes/acme_theme",
    "/_data/integrations/acme_sso"
  ]
}
```

### 5.2 Compute the folder list hash

The `folder_list_hash` is a SHA-256 over the **canonical** form of the
`designated_folders` array — sorted lexicographically, each path joined
with a single `\n`, no trailing newline.

```bash
jq -r '.designated_folders | sort | join("\n")' release_SCEL-2026-0001.json \
  | sha256sum
```

Add the hash into the file:

```json
{
  ...,
  "folder_list_hash": "sha256:a1b2c3d4..."
}
```

### 5.3 Sign the Release File

```bash
minisign -S -s suitefish_scel.key -m release_SCEL-2026-0001.json
```

This produces `release_SCEL-2026-0001.json.minisig`.

Deliver **both** files to the licensee.

### 5.4 Update the registry

Add an entry to `registry.json`:

```json
"SCEL-2026-0001": {
  "licensee_fingerprint": "sha256:b94d27b9934d3e08...",
  "folder_list_hash": "sha256:a1b2c3d4...",
  "approval_codename": "BLUEFIN",
  "issued_at": "2026-05-28T12:00:00Z",
  "status": "active"
}
```

Re-sign the registry:

```bash
minisign -S -s suitefish_scel.key -m registry.json
```

Upload both `registry.json` and `registry.json.minisig` to the hosting
location. **The registry is append-only** — never remove or rewrite
entries. To deactivate a license, change its status, do not delete it.

---

## 6. Verifying a Release File

Anyone — a licensee checking their own paperwork, a court, future Suitefish
staff — can verify a Release File with three steps:

### Step 1: Fetch the public key and registry

```bash
curl -O https://suitefish.example/scel/pubkey.minisign
curl -O https://suitefish.example/scel/registry.json
curl -O https://suitefish.example/scel/registry.json.minisig
```

### Step 2: Verify the registry's own signature

```bash
minisign -V -p pubkey.minisign -m registry.json
```

If this fails, the registry has been tampered with — stop.

### Step 3: Verify the Release File

```bash
minisign -V -p pubkey.minisign -m release_SCEL-2026-0001.json
```

Then check:

1. The `license_id` from the Release File appears in `registry.json`.
2. The `folder_list_hash` in the Release File matches the registry entry.
3. The registry entry's `status` is `"active"` (not `"revoked"` or
   `"suspended"`).
4. Recompute the SHA-256 of the canonical folder list and confirm it
   matches `folder_list_hash` (in case the Release File was tampered
   *with* a valid signature, which is impossible if signature verification
   passed — but belt-and-suspenders).

If **all four** checks pass, the Release File is genuine and active.
If **any** check fails, Section 7(d) of SCEL applies: the Release File is
invalid and confers no rights.

---

## 7. Revoking a license

When you determine misuse under Section 11(c) of SCEL:

### 7.1 Update the registry entry

Change the license's status:

```json
"SCEL-2026-0001": {
  ...,
  "status": "revoked",
  "revoked_at": "2027-04-15T09:00:00Z",
  "revocation_reason": "Reselling of CMS Core functionality"
}
```

### 7.2 Re-sign and republish the registry

```bash
minisign -S -s suitefish_scel.key -m registry.json
```

Upload the new `registry.json` and `registry.json.minisig`.

### 7.3 Notify the licensee in writing

Per Section 11(c), revocation must be communicated in writing. Email is
sufficient. Include:

- The License ID being revoked
- The effective date and time of revocation
- A summary of the conduct that triggered the revocation
- The URL of the registry where the revocation is publicly recorded

Keep a copy of the notification with a timestamp (e.g. send to yourself,
or use a service that timestamps emails).

---

## 8. Key rotation

You may eventually want to rotate the signing key — out of habit, after a
suspected exposure, or to upgrade algorithms.

### 8.1 Generate a new key

```bash
minisign -G -p suitefish_scel_v2.pub -s suitefish_scel_v2.key
```

### 8.2 Publish a transition statement

Create a `key_transition.json` document:

```json
{
  "old_pubkey_fingerprint": "RWQf6LRCGA9i...",
  "new_pubkey_fingerprint": "RWS8a1c2d3e4...",
  "rotation_date": "2028-01-01T00:00:00Z",
  "reason": "Routine rotation",
  "old_key_status": "retired_but_valid_for_pre_rotation_files"
}
```

Sign this statement with **both** the old and new keys. Publish all four
artifacts (statement + both signatures) at stable URLs.

### 8.3 Re-sign the registry with the new key

The registry going forward is signed by the new key. Old Release Files
remain verifiable using the old public key (which stays published with
`status: retired`).

### 8.4 Do not destroy the old key

Keep the old key in cold storage. You may need to issue an emergency
signature on it (e.g. to revoke a pre-rotation license) for years to come.

---

## 9. Compromise response

If you suspect or confirm that the private key has been stolen:

1. **Immediately publish a revocation statement** signed by the old key
   (if you still have access) and the new key.
2. **Generate a new key** and publish the transition statement.
3. **Re-sign and re-issue every active Release File** with the new key.
   Distribute the new signatures to all active licensees.
4. **Mark the old key as compromised** in the registry, with the date
   from which signatures by the old key should no longer be trusted.
5. Optionally, file a written record with a German notary timestamping
   the date of compromise. This is the strongest possible evidence in a
   future dispute over whether a given Release File predates or postdates
   the compromise.

The faster you publish the compromise statement, the smaller the window
in which an attacker could forge plausible-looking Release Files.

---

## 10. Quick reference

| Action | Command |
|--------|---------|
| Generate key | `minisign -G -p PUB -s KEY` |
| Sign file | `minisign -S -s KEY -m FILE` |
| Verify file | `minisign -V -p PUB -m FILE` |
| SHA-256 hash | `sha256sum FILE` |

Public URLs you must keep stable:

- Public key: `https://suitefish.example/scel/pubkey.minisign`
- Registry: `https://suitefish.example/scel/registry.json`
- Registry signature: `https://suitefish.example/scel/registry.json.minisig`

Files the licensee receives per license:

- `release_<LICENSE_ID>.json`
- `release_<LICENSE_ID>.json.minisig`
- The signed SCEL document (`scel_1.4.pdf` with the license fields filled
  in for that licensee)

---

## 11. Threat model summary

| Threat | Mitigation |
|--------|------------|
| Forged Release File | Signature verification fails against published public key |
| Tampered Release File | Signature verification fails |
| Stolen License ID claimed by unauthorized party | They cannot produce a signed Release File |
| Stolen private key, attacker forges new License ID | Registry check fails — License ID not present |
| Stolen private key, attacker re-signs existing License ID with modified folders | Folder hash check fails against registry |
| Tampered registry on the hosting server | Registry signature verification fails |
| Revoked license still claimed by ex-licensee | Registry shows `status: revoked` |
| Lost private key | Backup; if no backup, registry can never be updated again |

The only scenario this system does **not** defend against is a complete and
simultaneous compromise of the private key **and** the hosting infrastructure
**and** the absence of any independent timestamping of the original public
key publication. That scenario is implausible in practice and would
constitute extraordinary circumstances in any legal proceeding.
