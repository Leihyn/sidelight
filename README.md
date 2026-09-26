# Sidelight

**A payment that the route carrying it cannot recognise as one payment.**

Lightning payments travel through intermediaries, and under the hash-locked contracts in use
today every one of those intermediaries sees the same payment hash. Any two of them can compare
notes and prove they carried the same payment. Do that across enough of the network and the route
reassembles, along with who paid whom.

Live: https://sidelight-iota.vercel.app

## The mechanism

A point time-locked contract replaces the shared hash with a curve point. The sender picks the
recipient's secret `z` and a blinding offset `o_i` for every hop, then offers hop *i* the point

```
P_i = ( z + Σ_{j≥i} o_j ) · G
```

Every hop is therefore shown a different value, and none of them is `z·G`. Settlement walks
backwards: the recipient releases `z`, and each hop derives its own scalar from the one after it
by adding the offset the sender gave it:

```
s_i = s_{i+1} + o_i        and        s_i · G = P_i
```

So the money still moves, each hop is paid, and no hop ever holds the recipient's secret.

The page builds the same payment under both schemes on the same route, lets you pick any two
intermediaries and have them collude, and shows what each pair can and cannot prove. Under
hash-locking they match immediately. Under point-locking they hold two unrelated points.

## Running it

Static. No build, no server, no node, no wallet.

```
python3 -m http.server 8000
```

Then open `http://localhost:8000`. Append `?demo` to run the whole flow automatically.

## What is implemented

Per-hop blinded payment points, the backwards settlement walk with each hop's scalar checked
against its own point, and the hash-locked comparison it is measured against. Real secp256k1
arithmetic throughout.

Not implemented: adaptor signatures over real channel commitments, onion routing and the delivery
of offsets to each hop, timeouts and failure paths, and anything touching a network. Schnorr
signatures are a precondition for PTLCs on Bitcoin and are not modelled here. This proves the
unlinkability construction; it moves no money.

## Verification

The page's own script is loaded into Node behind a DOM shim, so the tests drive the shipped code
rather than a reimplementation. 19 assertions, ending with an exhaustive sweep:

```
PASS  under hash-locking the two hops prove it is one payment
PASS  under point-locking they find nothing in common
PASS  no two hops ever share a point (140 pairs)     0 collisions
PASS  every route length still settles in full        0 bad
PASS  the recipient secret never reaches a hop        0 leaks
```

The sweep runs every ordered pair of hops at route lengths 3, 5, 7 and 9. A single shared point
anywhere would break the claim, so it is checked everywhere rather than sampled.

## Dependencies

`@noble/secp256k1` v2, vendored, for curve arithmetic. WebCrypto for SHA-256. No framework, no
build step.

## Licence

MIT.
