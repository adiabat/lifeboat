


## Lifeboat and Lifejacket
Post-ECDLP output recovery


## Abstract

The document describes a commit / reveal scheme to allow safe recovery of outputs after a break in the security of elliptic curves.

## Motivation

Bitcoin currently uses secp256k1 signatures to secure transactions.  If the security of this elliptic curve were to break, either from the construction of a sufficiently powerful quantum computer or a novel algorithm that ran on today's computers, bitcoin would need to switch to a different signature scheme.  While there are promising canditates for new signature schemes, and most users could transition to these schemes before they are needed, not everyone would switch in time.  

Many people won't move their coins unless they have to, and if they have to, it's too late.  People would very much dislike having their coins stolen by an attacker, and having their coins destroyed by a soft fork disallowing EC signatures isn't any better (some would find it worse!).  Lifeboat aims to allow secure recovery of coins that would otherwise be lost or destroyed, while trying to minimize code complexity and keep things easy for users.

## Definitions

- h(): A preimage resistant hash-function, most likely SHA-256.  Use a tagged hash as described in BIP340, with a uniqe tag, ideally one which is only known after the EC break (ie the block hash of the first block proving an EC break)

- AID: Address ID.  

- CTXID: Commitment TXID.  The TXID of the transaction spending the EC UTXOs.

- Sequence Dependent Proof:  A hash of the concatenation of the unknown pubkey and spending txid.  This is like an HMAC, hah based message authenticatoin code, where the key for the hash is later revealed, allowing verification. 

## Description

In the event of an EC break, many UTXOs will still be safe because the output script uses hashes which hide the EC public key.  P2PKH, P2SH, P2WPKH and P2WPKH types which have never been seen as the input to a transaction all have this property.  
These outputs would still be at risk once spent, however, as an attacker would learn the pubkey as the spending transaction is broadcast to the bitcoin network and awaits confirmation. 

Lifeboat would add a new consensus requirement when spending these types of outputs, that the transaction spending them would already exist in the commitment set.  The commitment set is a key:value map which nodes store, much like the existing UTXO set. 

In lifeboat, wallets spending vulnerable outputs first make a commitment to the spending transaction, wait for the commitment to be confirmed, then broadcast the spending transaction.  Because the commitment is hash-based, it does not depend on the ECDLP and lets wallets securely move coins even in the case of an EC break.

The commitment format is:

C = (AID, SDP, CTXID)

Where
AID is h(pubkey) 
SDP is h(pubkey, CTXID)
CTXID is the TXID of the spending transaction.

The commitment is made as an OP_RETURN value, possibly with a marker prepended to indicate that it is a lifeboat commitment.



While the UTXO set is a mapping of outpoint (txid and output index) to UTXO data (Pubkey script, amount, etc), the commitment set would be a mapping of AIDs to (SDP, CTXID, confirmation height).  The confirmation height of the commitments is used to determine which is the first valid seen commitment.

Unlike the UTXO set however, the commitment set does not have to be a 1:1 mapping.  A single AID can store multiple (SDP, CTXID) entries, with differing commitment heights.


### Deterministic commitments

There a variable delay between when a wallet creates and broadcasts a commitment and when it can broadcast the spending transaction, since the wallet must wait for the commitment to be confirmed.  This could result in a problem where the wallet loses data about the spending transaction before broadcasting it.  While the wallet could create another spending transaction and annother commitment, if the transactions differ the later commitment would not be the first-valid commitment and so the funds would be stuck.

To avoid this problem, wallets should use (and this document should later specify!) a deterministic function for constructing the spending transaction based on (publicly recoverable) wallet state and HD keys, using the same HD root key for the new, non-EC destination public keys.



## Discussion

One of the downsides of this method is the need for the commitment set in order to determine which was the first valid commitment.  While this has disadvantages, bitcoin nodes already keep a UTXO set with most of the same issues.

One further complication is that while the normal bitcoin UTXO set can be put into a cryptographic accumulator, allowing nodes to verify without holding the entire set, (see utreexo), the same technique does not work as-is for the commitment set, due to the commitment set supporting multiple values for one key.  Further work is needed to investigate how accumulators can be used for the commitment set.

Another thing to note is that indexing based on AID isn't strictly needed.
You don't need AID in the commitment; you could make the commitment just (SDP, CTXID).  However this would mean verifiers would have to try to match every commitment they have every time a spend comes in, which scales very poorly.  This could be something to keep in mind though when looking at other ways to do it.

## Lifejacket


See lifeboat for discussion of the commitment structure, commitment set, and other aspects.  Lifejacket is based on lifeboat, but lifejacket is designed to move coins where the EC public key is known.  The most common examples of this is are P2PKH / P2WPKH outputs which remain unspent but where other outputs with the same pubkey hash have been spent in the past (generally known as address re-use)

The lifeboat commitment, `C = h(pubkey), h(pubkey, txid), txid`
doesn't work, because `pubkey` is known.  Instead we define a "pre-key", which is some type of precursor to the public key (several types are defined & enumerated).  The simplest pre-key is when a wallet can provide a SHA256 preimage of the private key which signs for the pubkey.  If this is known by the wallet, the wallet can use this to securely commit to a spending transaction, and reveal the preimage during the spending transaction.  As this preimage is hash based it does not depend on ECDLP.

Thus the Lifejacket commitment is

C = h(prekey), h(prekey, txid), txid

With h(prekey) being the address ID instead of h(pubkey)

The commitment is stored, indexed to h(pre-key), in a commitment set.  Multiple conflicting commitments can be stored at the same h(pre-key) entry, with differing timestamps.  As in lifeboat, only the first valid commitment is used.

Unlike in lifeboat, the pre-key would not be revealed by a normal bitcoin transaction the way a pubkey is.  The pre-key must be explicitly revealed in the spending transaction.  To do this, the pre-key is included in an `OP_RETURN` of the spending transaction.  The pre-key reveal `OP_RETURN` is prefixed with the pre-key type, such as SHA256, BIP32, Blake2b, among others.

On the verification side, when seeing a transaction which spends a lifejacket output, the verifier looks at the pre-key, performs the function indicated by the pre-key type identifier, multiplies the result by G, and checks that the resulting EC point is equal to the public key seen in the transaction.  If not, the transaction is invalid.  If the derivation succeeds, nodes check in the commitment set to ensure that it is the first valid commitment for this address ID.  Once the pre-key is known, the validity of all entries sharing that prekey can be determined by checking that the SDP is the hash of the other two elements for each entry.


### BIP32 pre-key type

BIP32 derivation would be the most commonly used pre-key type, as BIP32 is widely used in nearly all current bitcoin wallets.  Unhardened BIP32 derivation (I < 2^31) can be proven without revealing any private key data.  Hardened derivation needs to reveal private key information.  Since the assumption is that Lifejacket will only be used once the ECDLP problem is quickly solvable, one could argue there is no benefit to allowing unhardened derivation proofs, requiring all BIP32 proofs to reveal the private key information.  Still, users and wallet developers may feel safer and prefer to have a recovery method that does not reveal private key information. 

### Known key distinguisher

Likely the hardest implementation problem with lifejacket is that all nodes must be able to tell when an output being spent requires a lifejacket commitment as opposed to a lifeboat commitment.  On a transaction level, the two look the same.  The difference is that for the lifejacket transaction, the public key being used has already been seen.  Thus nodes need some way to determine, given a public key, if the public key has been seen before.  This determination would be part of consensus code, so all nodes must return the same answer.

TODO - write about public keys known but not seen in the blockchain.  Maybe commit some kind of blob of them somewhere.

If we stick to public keys revealed in the blockchain history, this is doable determinstically, since every node has already seen the same pubkeys.  The naive way to do this would be for nodes to build an maintain an address index, storing every pubkey they've seen and checking against it when they observe an EC spending transaction.  While possible this is inefficient, and would greatly increase the hardware cost of running a consensus node.  But it's possible!

Other options to flesh out include:

Filters.  Nodes can build a bloom or other filter of known public keys, and match spends against this filter.  Such filters do have false positives, but no false negatives.  A low but non-zero false positive rate may be tolerable, since it would not destroy coins, but only require what would otherwise be a lifeboat spend to instead use the lifejacket mechanism.

Sparse Merkle trees, or other accumulators that can provide proofs of non-inclusion.
If nodes build a lexographically sorted merkle tree of known public keys, then wallets can provide a non-inclusion proof of they pubkey, opting out of lifejacket and in to lifeboat.  Transaction without a proof would default to using lifejacket.

### UTXO loss

In the case of an EC break, hopefully:
1 Nearly all coins have been moved to EC-break-safe output types.
2 Nearly all of the coins which have not moved to safe output types haven't been subject to address re-use, and can use lifeboat.
3 Of the remaining coins which haven't moved, and have re-used addresses, a majority can be moved using lifejacket.

However, there's no way to look at an output and know if lifejacket will work.  If a wallet created it's private key by using un-hashed randomness, by e.g flipping a coin 256 times, (not a good idea, but possible!) they would have no private information about the key not obtainable through the EC-break.  A similar problem arises if the private key was created through a derivation process, but that derivation process was forgotten and only the resulting private key was retained.  This is likely to be an uncommon scenario, but given the millions of different users, it almost certainly will apply in some cases, and would result in the permanent loss of those coins.


### BIP32 specifics

The BIP32 derivation which is relevant for lifejacket is

`I = HMAC-SHA512(Key = c_par, Data = ser_P(K_par) || ser32(i))`
Truncate `I` to the first 32 bytes and call it `I_L` (discard `I_R`, the latter 32 bytes)
The public key is `K_par + (I_L * G).`

The threat in this case is that there's an attacker with a DL oracle, who gets a pubkey and wants to compute any (c_par, K_par, i) such that it satisfies the public derivation verification.

This isn't possible because of the hash step.  The attacker is given P, and needs to find C, K, i such that  `P = hash(C, K, i)*G + K` 
Ignoring C and i it's `P = hash(K)*G + K` which is basically the taproot equation, which still works when ECDLP is broken.



## Copyright

This document is licensed under the Creative Commons CC0 1.0 Universal license.
