## Recovery claims and resource exhaustion

When implementing PQ recovery protocols such as lifeboat, one of the tough implementation details is how to ensure that only the first valid commitment is used.  

Dropkick, another recovery protocol, doesn't have a "first valid" requirement, and allows any valid commitment of sufficient age to claim the coins.  This simplifies the protocol and avoids the need to compare multiple commitments, but at a security cost: if an attacker can postpone confirmation of the collection transaction long enough, they can steal the funds.

The problem seems simple enough to solve: keep track of all commitments, then reject the invalid and late commitments, allowing only the first valid one.  The simplest way to do this would be to use a key/value database, like leveldb, which bitcoin core uses to store the UTXO set.  Unlike the UTXO set, though, this would store many values at the same key, as there are multiple competing claims on the same coin.  This is also tractable: when a new claim is made, check the DB to see if there are any existing claims for that coin, and if there are, append the new one rather than starting a new entry in the set.

The problem is that this value can be appended to indefinitely.  But so can the UTXO set, so maybe it's no worse?  The problem is that when a collection transaction is broadcast, that's the first time verifying nodes can go through and figure out which claim is the first valid one.  And in the worst case, that involves reading all the data in the entry for that coin.  So an attacker can post a few million claims for the same coin, only the last one of which is valid.  When they broadcast the collection transaction, verifying nodes would need to process the prior million entries to make sure there is no earlier valid claim.

While this attack is costly for the attacker, the problem is that there's no limit to the resources they can cause a single transaction to take.  The validation time grows linearly with the resources the attacker spends.  The obvious workarounds, such as "limit the number of entries to 100" don't work.  Any such limit would then allow an attacker to destroy coins by confirming 100 invalid claims to a coin, outspending the valid claim's fees.

Another approach would be to make the claim process interactive.  This is slow and not great UX, but might be acceptable since the idea here is that these are coins that were dormant for years.  If the user needs to spend a few days to recover them and migrate to new addresses, that might be OK.  

I think SHRINC-ray is better and removes the need for an interactive claim protocol, but I'll mention it here to show how it would work, in case other people think it's a promising direction.

## Interactive claims

In interactive mode, the rule is that there's only 1 claim allowed at a time.  It's still stored in a K/V DB, but instead of appending, the only operation allowed is to overwrite, replacing the value V at the already claimed K.  The first confirmed claim gets in for free, but any later claim which tries to overwrite the first must *dis*prove the first claim.

The structure of the claims would be:

```
claim99:
h99(PK), txid, h(h98(PK), txid)
^        ^     ^ 
A        B     C  
```

Where PK is the hidden pubkey, txid is the txid of the desired collection transaction, and h99(), h98(), etc are the 99th and 98th sequential hash of SHA256.  

When first making a claim, the user computes the 98th and 99th hash of their hidden pubkey and tries to get it confirmed in an op_return.  If they do, great, nobody else can invalidate their claim.  But if an attacker flips a bit, they can respond.

If the attacker flips a bit in A, no action is needed since A is the coin identifier, and a different A means their claim doesn't conflict with the user's valid claim.

If the attacker flips a bit in B or C, the user can show this by making their new claim:

```
claim98:
h98(PK), txid, h(h97(PK), txid)
```

Verifiers presented with the new A can see if it is indeed the preimage of the old A.  They can also then see if the old B and C are constructed as they should be, since h98(PK) is now known.

If claim99 was invalid, it is replaced with claim98.  If claim99 was valid, it stands -- this is to prevent attacks on the claim after PK is revealed.

This seems to work but the attacker can just flip a bit in claim98 part C.  Then the honest user needs to reveal h97(PK) and try again.  The protocol wouldn't be limited to 99 tries, as it could efficiently be constructed with a tree billions of levels deep.  Even with a billion hash chain, it can be verified in constant time with the right tree construction.  So there's no resource exhaustion attack for verifiers here.  But it's really no fun for the user.  The attacker can delay the user indefinitely by consistently outspending him on fees.  As soon as the user gets a claim confirmed, they win.  But this is not a good setup for users.

(Another variant would be where claims have to send some small amount of coin to the claimed address.  This would no be a problem for users since they'd get it right back, but would be for attackers since each time they delay the user they'd be contributing to the user's funds.)


## SHRINC ray

A better construction is to keep the "only 1 claim per coin" restriction of the interactive protocol, but have claims be verifiable in isolation.

This claim is

```
SR(PK), txid, Sig(SR(PK), txid)
^       ^     ^
A       B     C
```

SR() is the shrinc-ray function, which takes as input any byte string, and outputs a SHRINCS public key.  B is the txid of the desired transaction, and C is the signature from the SHRINCS pubkey in A on the txid.

Only someone with knowledge of PK can compute SR(PK), and sign with that key.  When nodes see a claim of this format, they check that C is a signature on message B with key A.  If it's not, they reject the claim and don't store anything.  If it is, they store it, and ignore any later attempts to claim with key A.

When the collection transaction shows up in the mempool, nodes see that its txid matches B, and check that the pubkey being revealed shrinc-rays into A.  If those work, the transaction is OK, if not it's rejected.

(I think nodes don't have to store the signature C.  And can probably store a hash of A if it's much larger than 32B)

This allows a constant size storage for each claimed coin, with no contention over which claims are valid or invalid.  If an attacker flips a bit in B or C, the signature won't work anymore and the claim is rejected.  If the attacker flips a bit in A, the claim is not conflicting with the honest user.  (It wouldn't work to flip a bit, but the attacker could make up arbitrary keys A and claim & sign with them, but that will just make new entries in the DB, and is no worse than spamming the UTXO set.)


... still details to fill in, but I think this construction is the best direction for PQ recovery protocols.
