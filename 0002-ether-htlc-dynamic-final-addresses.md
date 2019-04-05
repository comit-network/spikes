# Ether HTLC with dynamic final addresses

* Authors: Lloyd Fournier
* Date: 2019-04-04

Issue: [#40](https://github.com/comit-network/RFCs/issues/40)

## Context

The RFC003 [Bitcoin HTLC](https://github.com/comit-network/RFCs/blob/master/RFC-005-SWAP-Basic-Bitcoin.md) and [Ether HTLC](https://github.com/comit-network/RFCs/blob/master/RFC-007-SWAP-Basic-Ether.md) have different functionality.
Bitcoin's UTXO model means you specify which identity will finally own the funds in the redeem and refund transactions.
In the Ethereum HTLC this is embedded in the contract from the beginning.

Both contracts are representative of the paradigms of each blockchain but their differences get in the way writing generic code for interacting with them.
The nature of Bitcoin allows the COMIT node to handle all the keys involved in the HTLC and simply ask the user for where they want the funds when the time comes.
To me this is good design and what we want in COMIT even outside of atomic swap protocols:

1. The users negotiate a protocol with no keys required from the users' wallets
2. The protocol execution starts with users funding the initial contracts from their wallets
3. The COMIT nodes execute the protocol (via the user's instruction and/or automatically).
4. The protocol execution completes by the users' getting the final assets they own into their wallets.

As a rule of thumb, the COMIT node should own the private keys of whatever identities are embedded in the contracts on the blockchains.

Since Ethereum is flexible we can restructure it to mimic how Bitcoin works and fit it into the above flow.

## Research

Bitcoin requires you witness the HTLC output with the secret and a signature (or a just the refund signature after a timeout).
The transaction that has the witness will specify where to redeem the funds in its output.

To follow the same pattern in Ethereum, our redeem transaction will need in its data:

1. The secret
2. The address to give the money to (the final redeem address)
3. a signature under the redeem identity from the COMIT node

The refund would be the same without (1)

### The Final Address

The decision we have to make here is whether the final address is:

1. Provided in the data
2. The sender address

(2) is nice because it means that there is no way to mess it up and send a transaction to an address you don't own. It also saves a bit of gas.

On the other hand, that might be a feature.
You may want to send the asset in the HTLC to a contract or something, so I'm leaning towards (1).


### The signature

The key things when deciding what to sign are:

1. It must not be possible to front run the transaction, learn the secret, and then create another transaction that redeems to a different address.
2. It must not be possible to re-use the signature on a different chain or a different contract on the same chain.

To ensure (1) we need to sign the final address (redeem or refund) with the private key of the corresponding identity.
To ensure (2) we should sign the HTLC's contract address, the network id and chain id.

So I propose that we sign the following messages for each outcome:

- Redeem: `network_id || chain_id || htlc_contract_address || final_redeem_address`
- Refund: `network_id || chain_id || htlc_contract_address || final_refund_address`

We have to hash the message before signing it.
In this case we can choose any 256 bit output hash function because we control both the singing code (in COMIT) and the verifier (in the contract).
Our choices are `SHA-256` and `Keccak-256`.
I think we should just use `Keccak-256` because it's less gas on EVM.

### Verifying the signature

To verify the signature we use the `ecrecover` precompiled contract.
Note that this means the signature we need to send is the 65 byte recoverable ECDSA signature.
(A recoverable signature is one where you can figure out the public key from the signature itself)

Here's the rust-styled pseudocode for the contract:

``` rust
let (expiry, redeem_identity, refund_identity, secret_hash) = embedded_somehow...;

// We're redeemimg with secret
if CALLDATASIZE == (20 + 65 + 32) {
    let (final_redeem_address, signature, secret) = CALLDATA();

    if sha256(secret) == secret_hash {
        let message_hash = sha3(network_id, chain_id, address(), final_redeem_address);
        let (v,r,s) = signature;
        if redeem_identity == ecrecover(message_hash, v, r, s) {
            log("Redeemed(secret)"); // could also log the address it was redeemed to
            selfdescrtruct(final_redeem_address)
        }
    }
    else {
        return
    }
} else {
    let (final_refund_address, signature) = CALLDATA();

    if timestamp() > expiry {
        log("Refunded");
        selfdescrtruct(final_refund_address);
    }
    else {
        return
    }
}
```


### Gas cost increase

This change would increase the gas needed:

1. The transaction costs 5780 gas more because of data
2. `ecrecover` costs 3000

Note that the current contract redeem costs around 31000 (or 6000 if you selfdestruct to an existing address I think...).
The current deployment costs 111,952 which shouldn't be impacted too much by this change.
