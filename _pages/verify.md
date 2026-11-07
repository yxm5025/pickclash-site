---
layout: page
title: Verify a record
include_in_header: false
permalink: /verify/
---

# Verify a PickClash record yourself

Every PickClash record page (`pickclash.co/@handle/league`) points at entries on **Base**, a public
blockchain, written through the Ethereum Attestation Service (EAS). Nothing on that ledger can be
edited or deleted after it is written, by anyone, including us. This page is the recipe for checking
one without trusting PickClash.

Network while we test: **Base Sepolia** (explorer `base-sepolia.easscan.org`). At launch: **Base
mainnet** (explorer `base.easscan.org`). The record page always links to the right one.

<br>

## 1. The record entry

Tap **View on the public ledger** on a record page, or scan the QR on a shared receipt. The explorer
shows an attestation with this schema:

```
string handle, string league, uint64 weekStart, uint64 weekEnd,
uint16 wins, uint16 losses, uint16 pushes, int32 returnBps,
uint8 percentile, bytes32 picksRoot, bytes32 prevUID
```

Schema UID (identical on Sepolia and mainnet, because it is derived from the text above):
`0x` followed by the keccak256 of the schema text, a zero resolver address, and the byte `01`
(revocable). The explorer prints it on every entry.

What to check:

- **Attester** is PickClash's signer address (published on this page once we are on mainnet).
- **handle / league / window** match the page. `weekStart` and `weekEnd` are Unix seconds (UTC).
- **wins, losses, pushes, returnBps, percentile** match the receipt. `returnBps` is the return at
  one unit per pick in basis points: `940` means `+9.40%`.
- **prevUID** is the previous week's entry for the same record (zero for the first). Follow it back to
  see the whole chain; every week while the picker stayed Verified has one.
- **Revoked** entries are shown as revoked by the explorer. We revoke only when an account is deleted.

<br>

## 2. The picks behind it

`picksRoot` is the root of a Merkle tree over that week's settled picks. Each pick was committed to
the ledger **before its game ended**, in hourly batches, with a second schema:

```
bytes32 root, uint32 count, uint64 firstLockedAt, uint64 lastLockedAt
```

Each pick on a record page links to its batch. The batch time is the earliest moment anyone can
prove the pick existed; the pick's own locked time is PickClash's word, the batch time is not.

A pick's **leaf** is the SHA-256 of this exact JSON, keys sorted, no spaces, UTF-8:

```
{"betId":"…","gameId":"…","league":"NCAAF","lockedAt":"2026-11-01T15:02:11.123456+00:00",
 "market":"spreads","point":"-6.5","price":"-110","salt":"<32 hex chars>","target":"home",
 "userRef":"<handle>","v":1,"wager":"100"}
```

The `salt` is private to the picker, so a published root reveals nothing about anyone's picks.
A picker can reveal a leaf's JSON and its **proof path** (a list of sibling hashes) to prove that pick
is in a batch. To check it:

```
h = sha256(leaf_json_bytes)
for sibling in proof:            # order does not matter: pairs are sorted
    a, b = sorted([h, sibling])
    h = sha256(a + b)
h == root                        # the batch's root on the ledger
```

An odd node at any level is carried up unchanged. The same rule builds `picksRoot` from the week's
leaf hashes, sorted by locked time.

<br>

## 3. What Verified means

Per league, over the last four weekly windows, scored at **one unit per pick** (coins and boosts do
not count):

1. at least 15 settled picks,
2. active (3+ settled picks) in 3 of the 4 weeks,
3. a winning week in 3 of the 4,
4. a return in the top 10% of all pickers in that league who meet the first check.

It is re-run at every weekly close and can lapse. A lapsed record keeps its entries; it simply stops
getting new ones until the checks are green again.

<br>

## 4. What is never on the ledger

Names, emails, payment details, coin balances, pod membership, or any pick of a picker who has not
earned the check and tapped Share. Pick batches contain hashes only.

Questions or a mismatch: pickclash@gmail.com.
