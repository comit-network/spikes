# Kzen Two Party ECDSA Review

* Authors: Lloyd Fournier
* Date: 2019-05-08

Issue: [#897](https://github.com/comit-network/comit-rs/issues/897)

## Context

In order to do a [*Scriptless Scripts* atomic swap](https://lists.linuxfoundation.org/pipermail/lightning-dev/attachments/20180426/fe978423/attachment-0001.pdf) with ECDSA signatures we need to have an implementation of Yehuda Lindell's [Fast Secure Two-party ECDSA Signing](https://eprint.iacr.org/2017/552.pdf) protocol.
The point of this spike is to review the protocol and KZen's [rust implementation](https://github.com/KZen-networks/multi-party-ecdsa) and make notes about.

## Overview

The paper by Yehuda Lindell's [Fast Secure Two-party ECDSA Signing](https://eprint.iacr.org/2017/552.pdf)is a way to generate a joint public key and to generate joint signatures under this public key.
This is usually called signature *aggregation*.
There's a video by the Lindell summarizing the paper here: https://www.youtube.com/watch?v=pwc_Ork-1aA

Aggregated signatures are a nice thing to have.
They look exactly like normal signatures but are produced by multiple parties instead of one.
Practically, this means you don't need to lock Bitcoin to multiple public keys and check the signature with `OP_CHECKMULTISIG`; you can just lock it to a joint public key using a normal `P2WSH` output (i.e. use `OP_CHECKSIG`).

Since ECDSA signatures doesn't have a nice algebraic structure it is **very** difficult to do signature aggregation.
In schnorr you can quite literally just add each party's share of the signature together.
The method we have to use in ECDSA is to move the operations to produce the *s* component of the signature into a [Paillier group](https://en.wikipedia.org/wiki/Paillier_cryptosystem) rather than doing them simply modulo the Secp256k1 curve order.
Once you're in Paillier land things become much easier but it comes at a cost of boatloads of messages and complexity during the key generation and exchange phase.
It is so complicated that it's amazing that anyone actually bothered to come up with this.

Once the keys are generated, Paillier is used to sign roughly in the this way:

1. P1 sends the Paillier encryption of their private key to P2 (this is actually done in the keygen phase)
2. P2 uses Paillier homorphic operations with the encrypted private key (addition and multiplication) to build a new ciphertext which has P2's parts of the signature in it and sends it to P1.
3. P1 then decrypts it and then pretty much has a valid ECDSA signature.

The code for the signing part is in the main 2pECDSA crate: https://github.com/KZen-networks/multi-party-ecdsa/blob/e5a741bf8dd756b650b35ef8d65f6cecbd4f196a/src/protocols/two_party_ecdsa/lindell_2017/

The hard bit is doing the setup phase which involves quite a few proofs.
Some of them are in the above crate but some of them are imported from elsewhere.
Here's a list of the main proofs that are needed to implement the protocol.

## Key Generation

### Generating a shared public key

To produce signatures under a shared public key you first need a shared public key.
Because of rogue key attacks, you can't just have one party send their public key and the other send one back.

The technique used in the paper (and followed by the implementation) is to do a coin tossing like protocol.
This ensures that the public key is totally random and cannot be biased by the party that chooses their key second.

Note, The ability to simulate a random coin toss gives this protocol very strong security guarantees. Not doing so, doesn't necessarily mean it would be *insecure*.

The protocol is roughly as follows:

- P1 -> Hash(p1_public_key, schnorr_nizk(p1_public_key))
- P2 -> p2_public_key, schnorr_nizk(p2_public_key)
- P1 -> Opens commitment from first message
- Both parties calculate `joint_public_key = p1_public_key + p2_public_key`.

#### Implementation notes
The implementation commits by sending two hashes, one for the key and one for the NIZK.

The NIZK is implemented here: https://github.com/KZen-networks/curv/blob/18f0081a9a3025eafd49ad496e5032647edf4aa3/src/cryptographic_primitives/proofs/sigma_dlog.rs

### Proving Paillier modulus is well formed

In order to use Paillier encryption P1 needs to generate a RSA type modulus `N = pq` where `p` and `q` are large primes.
P1 needs to prove that she generated `N` properly (I actually don't know the details of what the risk is to P2 if she doesn't).

The proof mentioned in the original paper is this one: https://eprint.iacr.org/2011/494.
But in a more recent paper on multi-party ECDSA Lindell mentioned a more efficient protocol to make the proof: [https://eprint.iacr.org/2018/987.pdf](https://eprint.iacr.org/2018/987.pdf).
See the section 6.2.3, which explains how to modify the proof in [https://eprint.iacr.org/2018/057.pdf](https://eprint.iacr.org/2018/057.pdf) to simply prove `N` is generated fairly.

Roughly speaking, the strategy is to prove that `N` is *square free* and to show that `f(x) = x ^ N (mod N)` is a permutation which proves that `N = pq`.

#### Implementation notes

Since it's a public coin protocol, you can make an non-interactive version of the proof using the [Fiat-Shamir transformation](https://en.wikipedia.org/wiki/Fiat%E2%80%93Shamir_heuristic).
They've implemented a non-interactive version of it here: https://github.com/KZen-networks/zk-paillier/blob/a7f3907fb2b3464f12b55e3ad2f50405db54f36a/src/zkproofs/correct_key_ni.rs (which uses the interactive proof module but supplies the challenge as a hash of the initial commitment).

### Proving a Paillier encryption is a private key of a EC public key

In order to produce two party aggregated ECDSA signatures on a joint Secp256k1 public key we need one party (P1) their Paillier encrypted private key over to P2.
But in order for P2 to know that he's signing under their joint public key, he has to know that this encrypted private key is the right one (without learning what the private key is).

The proof for this was developed in paper. It's a little tricky to get your head around.
It works by the P2 doing some homomorphic operations on the encrypted private key and sending the result to P1.
P1 then decrypts it and figures out the corresponding public key.

The weird thing is that P1 doesn't actually use his knowledge of what the private key is in order to produce the proof that the cipher text doesn't contain the private key.
He simply uses his ability to decrypt the Paillier private key!

#### Implementation notes

This proof isn't in its own module but the code for it hangs around with the rest of the protocol:

https://github.com/KZen-networks/multi-party-ecdsa/blob/e5a741bf8dd756b650b35ef8d65f6cecbd4f196a/src/protocols/two_party_ecdsa/lindell_2017/party_one.rs

It looks like this protocol cannot be made non-interactive. I think it's the only one that couldn't be so it accounts for most of the rounds of communication.
It has four rounds of communication.

### Range proof

In order for the previous proof to actually prove the statement you have to couple it with a range proof which proves that the encrypted private key is in the curve order (i.e. is a valid private key).
The poof chosen was originally from [https://www.iacr.org/archive/eurocrypt2000/1807/18070437-new.pdf](https://www.iacr.org/archive/eurocrypt2000/1807/18070437-new.pdf) but I found it was easier to understand in Lindell's paper anyway (see Appendix A).

The proof uses the cut and choose technique so it's quite large.
It's tricky to understand, but doesn't use any special math. You just have to follow what happens closely.

#### Implementation notes

To prove that the private key lies within the curve order P1 first has to choose their private key so that it's in `Z_q/3` rather than `Z_q`.
Without this the proof will not be *complete*.

It's implemented here:

https://github.com/KZen-networks/zk-paillier/blob/a7f3907fb2b3464f12b55e3ad2f50405db54f36a/src/zkproofs/range_proof.rs


## Messages

### Keygen

Here's my early sketch of how many messages you need:

1. P1 -> `Hash(p1_public_key, schnorr_nizk(p1_public_key))`
2. P2 ->
   1. `p2_public_key`
   2. `schnorr_nizk(p2_public_key)`
3. P1 ->
   1. Opens commitment from first message
   2. Paillier modulus `N`
   3. Proof `N` was generated properly
   4. `(c,r) = PaillierEncrypt(p1_private_key)`
   5. Range proof for `c` encrypts a valid private key
4. P2 -> Challenge for `c` being the private key of `p1_public_key`
5. P1 -> Committed response for `c` being valid
6. P2 -> Reveal challenge from (4)
7. P1 -> Open response from (5)
