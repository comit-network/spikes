# Secret seed storage and key derivation

* Authors: Lloyd Fournier
* Date: 2019-04-16

Issue: [#672](https://github.com/comit-network/comit-rs/issues/672)

## Context

Currently, our node's 32 byte secret seed is stored in the config file as hex and we derive child seeds from it for each protocol execution.
This system was invented ad-hoc and the motivation for this spike is to design a well thought out solution that is informed by other projects.

I present the conclusions first and a summary of what I learned from LND.

## Secret Seed Storage

This section recommends how we should generate and store the secret seed by answering key questions.

### Can you recover anything if you have your secret seed but no database backup?

**In practice, No.**

In Theory and in the case of Ethereum only you could scan the blockchain for any HTLC looking contract paying money to an address you generated deterministically from your seed and reclaim it.

In Bitcoin you can't because you'll never be able to compute the P2WSH scriptPubKey without knowing the other person's input.
If it has already been redeemed, you could scan for redeem transactions with script that matches our Bitcoin HTLC, recover the secret and then look for the HTLC on the other chain (as above, assuming it's Ethereum).

I don't think this partial recovery is worth putting the significant effort that it would require to implement.

### Should it be encrypted?

I think this should be optional.
A COMIT node doesn't hold funds long term.
It can only control the funds during the execution of a protocol.
This is different to LND where if someone gets access to your seed they can take all your funds regardless of whether you have channels open or not.

The best attack vector against an unencrypted seed is for the attacker to get short term access to the system COMIT is running on, covertly steal their secret seed and then wait for them to execute a swap and then redeem to their own address.
Encrypting the seed does give you protection against this.

Having said that, giving an attacker access to your COMIT node will probably give them access to the HTTP API which they can use to redeem funds to their own address.
This is probably a better attack vector than the above.

Note: We could prevent this attack by giving the node a way of generating the final redeem addresses without getting them from the api.

For the sake of expediency I propose we incrementally increase the security, i.e. (i) leave the secret unencrypted for now, (ii) make encryption an option later on and (iii) make encryption the default eventually.

### Where to store it?

I don't think we should store secret seed in the database file (this is what lnd does).
This lets users backup the secret seed separately from the database (although they need to backup both).
Given that the two are linked, we should store a hash of the secret seed along with the swap id (in the same row or whatever).
This allows the user to change the secret seed while not invalidating the entire database.

By default I'd put it somewhere like:

```
~/.comit/secret_seed
```

But its location should be configurable with `secret_seed_file` in the `.toml` config.

The format of the file should be toml with a single entry `secret_seed` (so we can `encrypted_secret_seed` later).

We should make the permissions restrictive so only the user has read permission on it.

#### What about encrypting the database?

I don't think it's worth encrypting the database yet until we have a motivation to do it.
An attacker stealing the database would learn what protocol executions you're involved in so atm it's just a privacy issue.

We *could* encrypt the database with the `secret_seed` seeing as they're separated.

### How to generate it?

If `secret_seed_file` isn't set, I think we should just generate it from `OsRng` to `~/.comit/secret_seed` if it doesn't exist.

We can make a tool to generate/restore it from a mnemonic and encrypt/decrypt it later on.

### Should it have a checksum?

This probably won't catch much but it seems like a decent idea to attach a checksum to it if it's stored or transferred in a way that could be corrupted (or perhaps corrupted accidentally via a text editor).

## Secret/NodeID Derivation

I think the way we are doing it now is roughly the right way but I recommend modifying it in the following ways:

### length extension attacks

Informally, there are three things I think you need to ensure when doing deterministic derivation:

1. The derived keys should appear to be random.
2. Seeing a derived key shouldn't help you learn anything else about the keys that will be derived in the future (or the keys that have been derived in the past).
3. You shouldn't use deterministic derivation where a malicious party can use this against you.

[Length extension](https://en.wikipedia.org/wiki/Length_extension_attack) attacks break (2) and as we are using SHA-256 we could be susceptible to them.
A contrived example is that let's say you derive `secret` and `secret_key` by doing `hash(seed || "secret")` and `hash(seed || "secret_key")`.
With SHA256 it is possible to figure out what `secret_key` is once you've seen `secret` IF the length of `seed || secret` happens to be equal to the block length of the hash function (I think).

Obviously, in our case you can't do this attack but I'd suggest we use a hash function that isn't susceptible to it.

### Use u8 rather than strings to identify key types

There's nothing inherently wrong with using utf-8 encoded strings to a hash function to produce secret keys etc. e.g. `hash(seed || "secret")` (putting length extension attacks to the side).
This is what we do now in rfc003's [SecretSource](https://github.com/comit-network/comit-rs/blob/48d807d3b173f62b3e6adb2c5aebcb2ebb6967a5/application/comit_node/src/swap_protocols/rfc003/secret_source.rs).
There is another way to do it which is to simply encode integers for each type of key:

``` rust
enum SecretType {
    Secret = 0,
    RedeemKey = 1,
    RefundKey = 2,
}

..
fn secp256k1_redeem(&self) -> KeyPair {
    KeyPair::from_secret_key_slice(self.sha256_with_seed(&[SecretType::RedeemKey as u8]).as_ref())
        .expect("The probability of this happening is < 1 in 2^120")
}
```

This prevents spelling mistakes and means you can rename the key types internally without breaking anything.
This also fixes the length extension problem if we were to continue to use SHA256.

### Use keyed BLAKE2b for key derivation

In order to do key derivation I suggest using [blake2b](https://blake2.net/) like it's shown in the example here:

https://pynacl.readthedocs.io/en/latest/hashing/#key-derivation-example

Except without a salt.
We may use a personalization string if our implementation supports it.

The main reasons are:

- It's been well studied.
- It's immune to length extension attacks.
- It can produce hashes of variable length without truncating the output.
- It's fast. Since we don't store the results of what we hash but rather derive it on demand this is handy.

BLAKE2b's key is 64-bytes so I also suggest changing the secret seed we use to 64-bytes.

The implementation I recommend using is https://github.com/RustCrypto/hashes just because it's maintained by the rust crypto organisation.

### NodeID

I think we should derive the NodeID from the master seed simply by hashing something arbitrary.
We should leave the door open to having multiple top level ids (i.e. one for lightning, one for COMIT).

# Research Notes

## Lightning/LND

To help inform the above decisions I looked at LND because it's the flagship layer two protocol node and has to generate many keys and secrets.
It faces a similar problem to us in that you can't restore stuff you've learned from the other party (signatures on states, revocation keys etc) from the seed.

### Seed storage

The wallet seed is stored encrypted with a password in the node's `wallet.db` (or at least I think that's where it's hiding).
You have to unlock the wallet on lnd startup with the password via an RPC call (this is what the ln-cli does).
There is a (to be deprecated) `--noseedbackup` option  which encrypts it with the default password so you don't have unlock the wallet.
But this means you can't back up your seed so it's strictly for development.

Here's an interesting issue which discusses the possibility of making the password optional  (i.e. leaving the seed unencrypted): [#899](https://github.com/lightningnetwork/lnd/issues/899) (note they removed the `--noencryptwallet` thing mentioned there).

### Generation of HTLC secrets

LND derives new HTLC secrets simply using a random number generation.
This kind of makes sense because they allow the to specify the secret so there wouldn't be that much point in deriving it deterministically in cases where they don't choose it.

### Generation of Initial Public Key "basepoints" and node id

When starting a new channel the parties exchange the "basepoint" public keys (curve points) they're going to use.
They derive the keypair using BIP32 and send each other the public key:

```
m/201'/coinType'/keyFamily/0/index
```

Where `keyFamily` is the type of key being derived and `index` represents the particular channel.

The node's id is derived in the same way (except the `index` is always 0).

### Deriving the `per_commitment_point`

Each channel state has a `per_commitment_point` from both parities.
When updating the state you give the other party the private key for the previous `per_commitment_point` and give them your `next_per_commitment_point`.

They [use a clever trick](https://github.com/rustyrussell/ccan/blob/master/ccan/crypto/shachain/design.txt) to enforce that the private key for each `per_commitment_point` is derivable from the next.
This means you only have to store the most recent one.
Using this trick is enforced in the protocol.
You can only verify that the other person is doing deriving their key properly once they reveal the `per_commitment_point`'s private key.

### Deriving the Particular Public Keys of using the `per_commitment_point`.

The actual public keys used in the contracts are derived from these + hashing the `per_commitment_point` like:

```
pubkey = basepoint + SHA256(per_commitment_point || basepoint) * G
```

except for the revocation key which has a special derivation because the other party needs to learn the secret key once their `per_commitment_point` is revealed.
