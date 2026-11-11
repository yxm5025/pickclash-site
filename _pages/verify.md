---
layout: page
title: Check a tracker
include_in_header: false
permalink: /verify/
---

# Check a PickClash tracker yourself

Every PickClash pick tracker (`pickclash.co/@handle/league`) points at entries on **Base**, a public
blockchain, written through the Ethereum Attestation Service (EAS). Nothing on that ledger can be
edited or deleted after it is written, by anyone, including us. This page is the recipe for checking
a tracker without trusting PickClash.

Network while we test: **Base Sepolia** (explorer `base-sepolia.easscan.org`). At launch: **Base
mainnet** (explorer `base.easscan.org`). The tracker always links to the right one.

<br>

## 1. Every pick

Open **See every pick on the public ledger** on a tracker. Each pick shows one time, when it was
made, and an **on ledger ↗** link. The link opens the hourly batch that pick was saved in:

```
bytes32 root, uint32 count, uint64 firstLockedAt, uint64 lastLockedAt
```

What the batch proves: the pick existed, exactly as shown, by the time printed on the explorer, and
has not changed since. Picks are saved **within the hour they are made**. The "Made" time on the
page is PickClash's clock; the batch time is the ledger's, and it is the one nobody can move.

Picks can be made while a game is being played. If that matters to you, compare the batch time with
the game: a pick whose batch was written before the game ended was certainly made before the result.

A pick's **leaf** is the SHA-256 of this exact JSON, keys sorted, no spaces, UTF-8:

```
{"betId":"…","gameId":"…","league":"NCAAF","lockedAt":"2026-11-01T15:02:11.123456+00:00",
 "market":"spreads","point":"-6.5","price":"-110","salt":"<32 hex chars>","target":"home",
 "userRef":"<handle>","v":1,"wager":"100"}
```

The `salt` is private to the picker, so a batch reveals nothing about anyone's picks. A picker can
reveal a leaf's JSON and its **proof path** (a list of sibling hashes) to prove that pick is in a
batch. To check it:

```
h = sha256(leaf_json_bytes)
for sibling in proof:            # order does not matter: pairs are sorted
    a, b = sorted([h, sibling])
    h = sha256(a + b)
h == root                        # the batch's root on the ledger
```

An odd node at any level is carried up unchanged.

<br>

## 2. The season record

When a picker publishes their tracker, and after each weekly close that changes their record while
the tracker stays public, PickClash writes one readable entry (the **Season record entry ↗** link at
the bottom of the every-pick page):

```
string handle, string league, uint64 seasonStart, uint64 asOf,
uint32 wins, uint32 losses, uint32 pushes, int32 unitsCenti,
uint64 verifiedSince, bytes32 picksRoot, bytes32 prevUID
```

Schema UID (identical on Sepolia and mainnet, because it is derived from the text above):
`0x` followed by the keccak256 of the schema text, a zero resolver address, and the byte `01`
(revocable). The explorer prints it on every entry.

What to check:

- **Attester** is PickClash's signer address (published on this page once we are on mainnet).
- **handle / league** match the page. `seasonStart`, `asOf` and `verifiedSince` are Unix seconds (UTC).
- **wins, losses, pushes** are this season's record as of `asOf`, a weekly close. The page itself is
  live, so it can be ahead of the latest entry by the picks settled since.
- **unitsCenti** is the season's units times 100, at one unit risked on every pick: `945` means
  `+9.45` units. A win pays the pick's price, a loss costs one, a push is zero.
- **picksRoot** is the root of a Merkle tree over the picks that settled in the week ending at `asOf`
  (the same sorted-pair rule as above, leaves sorted by the time the pick was made).
- **prevUID** is the picker's previous entry (zero for the first). Follow it back to see the chain.
- **Revoked** entries are shown as revoked by the explorer. We revoke only when an account is deleted.

Entries written on the test network before November 10, 2026 use an earlier layout (a four-week
window with `returnBps` and `percentile`).

<br>

## 3. What Verified means

Per league, over **all** of a picker's settled picks there, scored at **one unit per pick** (coins
and boosts do not count):

1. at least 50 settled picks,
2. at least 3 weeks with 3 or more settled picks,
3. a win rate of 56% or better (wins divided by wins plus losses; pushes are left out),
4. units above zero.

It is checked at every weekly close. A picker who passes all four is Verified in that league for
good: it does not lapse. The tracker keeps showing every pick after that, wins and losses.

<br>

## 4. What is never on the ledger

Names, emails, payment details, coin balances, pod membership. Pick batches contain hashes only. A
readable season record is written only for a picker who is Verified and has published their tracker.
A picker can take a tracker offline at any time: the page closes and no new entries are written, but
entries already on the ledger stay there.

Questions or a mismatch: pickclash@gmail.com.
