# Trust-Minimized Public Tender System
### Sealed bids without trusting government employees

> A practical, buildable design for running sealed-bid tenders where  
> **no one can read bids early**, **logs are publicly immutable**, and  
> **database manipulation doesn’t matter**.

---

## TL;DR

Current tender systems require trusting insiders. This design replaces trust with **cryptography + public verification**.

- Bids are encrypted **in the browser**
- Decryption keys **do not exist** until opening time
- Every action is logged in a **public, append-only ledger**
- Even compromised servers cannot cheat silently

No new crypto. No theory. Only **already-deployed infrastructure**.

---

## Motivation

In most government tenders today:

- Bids are uploaded to government servers
- Logs live in mutable databases
- Submission order is hidden
- Internal audits are the only defense

This creates a trust gap:
- Insiders *can* leak bid ranges
- Logs *can* be rewritten
- The public cannot independently verify fairness

Even if the process is honest, **there is no way to prove it**.

---

## Design Goal

Assume:

- Government employees **cannot be trusted**
- Databases **can be manipulated**
- Bidders submit via a **web app**
- Bidder collusion is **out of scope**

We want a system where:

> Even if the entire backend is compromised, fairness still holds.

---

## Core Idea

Split the problem into two parts:

1. **Confidentiality** – prevent early access to bids  
2. **Integrity** – prevent rewriting or deleting history  

Solve each independently with proven tools.

---

## 1. Bid Confidentiality  
### “The key does not exist until opening time”

**Tool:** `drand` timelock encryption (`tlock` / `tlock-js`)

### How it works

- Bidder encrypts the bid **locally in the browser** (AES-GCM)
- The AES key is *timelocked* to a **future drand round**
- drand is a public randomness beacon run by independent institutions
- The randomness needed to decrypt **does not exist yet**

### Result

- Servers never see plaintext bids
- Government employees cannot decrypt early
- There is nothing to leak, even with full access

This is strictly stronger than commit–reveal or server-side controls.

---

## 2. Public Immutability  
### “Logs like git commits, but public”

A database alone is not enough:
admins can rewrite history.

So we use a **public append-only transparency log** where:

- Every event is permanent
- Deletions and reordering are detectable
- Anyone can verify history independently

There are **two real-world options**.

---

## Option A — Sigstore Rekor

**What it is**
- Modern transparency log used for software supply-chain security
- Append-only Merkle tree
- Publicly readable
- Signed inclusion receipts
- Inclusion & consistency proofs

**How it’s used**
Each tender action becomes a Rekor entry:
- Tender created
- Bid committed
- Encrypted bid uploaded
- Bid opened
- Winner declared

**Pros**
- Developer-friendly APIs
- High throughput
- Actively maintained
- Designed for arbitrary JSON events

**Cons**
- Benefits from external anchoring (optional)

---

## Option B — Certificate Transparency (CT)

**What it is**
- The system browsers use to detect rogue SSL certificates
- Internet-scale, battle-tested append-only log

**How it’s used**
- Tender events are logged like certificates
- Signed timestamps (SCTs) act as receipts
- Public Merkle roots prevent equivocation

**Pros**
- Extremely strong guarantees
- Proven at global scale

**Cons**
- Heavier operational complexity
- Less flexible APIs

---

## Storage Model

- **Encrypted bid files** → S3 / GCS / IPFS / Arweave
- **Public logs** → Rekor *or* CT
- **Database** → indexing, search, UI only (not trusted)

If the database is altered:
- Receipts + public log expose it immediately

---

## Event Model (Public & Immutable)

Each tender has a hash-chained timeline (git-style):

- `TENDER_CREATED`
- `BID_COMMITTED`
- `BID_UPLOADED`
- `BID_OPENED`
- `WINNER_DECLARED`

Each event includes:
- Tender ID
- Event type
- Payload hash
- Previous event hash
- Timestamp (from the log)

This creates a **verifiable, append-only history**.

---

## Optional: External Anchoring

For extra protection against log-operator collusion:

- Periodically anchor the log’s **Merkle root** to:
  - a public blockchain, or
  - OpenTimestamps / Bitcoin

Only a 32-byte hash is anchored.

---

## What This Solves

✅ No early bid access  
✅ No insider leaks  
✅ No silent deletions  
✅ No backdating  
✅ No “trust us” explanations  

Anyone — citizen, auditor, journalist — can verify everything.

---

## What This Does NOT Solve

- Bidder collusion  
- Poor evaluation criteria  
- Political influence outside the system  

This is infrastructure, not governance magic.

---

## Why Blockchain Is Optional

Blockchain can act as an **external witness**, but:

- Rekor / CT already provide immutability
- Anchoring is an optional extra layer

> Blockchain is an anchor, not the core system.

---

## Why This Is Buildable

All components already exist:

- `drand` + `tlock` (production-used)
- Sigstore Rekor
- Certificate Transparency
- WebCrypto (built into browsers)

This is **systems composition**, not cryptography research.

---

## Getting Started (Where to Actually Begin)

### 🔐 Timelock Encryption (Core Confidentiality)
- drand (randomness beacon):  
  https://github.com/drand/drand
- Timelock encryption (Go):  
  https://github.com/drand/tlock
- Timelock encryption (JavaScript / browser):  
  https://github.com/drand/tlock-js
- League of Entropy (who runs drand):  
  https://www.cloudflare.com/leagueofentropy/

### 📜 Transparency Logs (Public Immutability)
- Sigstore Rekor:  
  https://github.com/sigstore/rekor
- Rekor API & docs:  
  https://docs.sigstore.dev/logging/rekor/
- Certificate Transparency overview:  
  https://certificate.transparency.dev/
- Trillian (CT-style log implementation):  
  https://github.com/google/trillian

### ⛓️ Optional Anchoring / Timestamping
- OpenTimestamps:  
  https://opentimestamps.org/
- OpenTimestamps GitHub:  
  https://github.com/opentimestamps

### 🌐 Browser-Side Crypto
- WebCrypto API (AES-GCM, SHA-256):  
  https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API

### 📦 Public Storage
- IPFS:  
  https://ipfs.tech/
- Arweave (permanent storage):  
  https://www.arweave.org/

---

## Suggested MVP Path

1. Browser encrypts bid with AES-GCM (WebCrypto)
2. Timelock AES key with `tlock-js` to a future drand round
3. Upload ciphertext to S3/IPFS
4. Log events to **Rekor**
5. Build a simple “Tender Explorer” UI that verifies receipts

You can build a working demo in **weeks**, not months.

---

## Why This Matters

In controversial tenders, the real problem is often not corruption —  
it’s **unverifiable trust**.

This design replaces:

> “Trust the process”

with:

> “Verify the math.”

---

## Call to Action

If you are an:
- engineer
- auditor
- policymaker
- open-source contributor

This is a **real, unsolved governance problem** worth building for.

Fork it. Improve it. Critique it. Implement it.

---

## One-Line Summary

> A tender system where bids are encrypted before submission, keys don’t exist until opening time, and every action is publicly logged forever — even government employees can’t cheat.
