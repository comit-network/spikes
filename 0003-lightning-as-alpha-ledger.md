# LND as Alpha Ledger as decribed in RFC003

* Authors: Philipp Hoenisch
* Date: 2019-04-24

Tracking issue: TBA

## Context

The **Hold invoice feature** was introduced in the following PR: [lightningnetwork/lnd/pull/2022](https://github.com/lightningnetwork/lnd/pull/2022).
> A hold invoice is a new type of invoice that triggers a different flow at the receiver end.
> Instead of immediately locking in and settling the htlc when the payment arrives, the htlc for a hold invoice is only locked in and not yet settled.
> At that point, it is not possible anymore for the sender to revoke the payment, but the receiver still can choose whether to settle or cancel the htlc and invoice.

This allows us to trade Bitcoin on the Lightning Network (LN) for an asset _X_ on a different ledger where LN is the _alpha ledger_ as per the definition in [RFC003](https://github.com/comit-network/RFCs/blob/master/RFC-003-SWAP-Basic.md).
For the other way around, Bitcoin on the Lightning Network is the _beta asset/ledger_, we can make use of `createinvoice`.

### Standard _hold invoice_ process

> Assumption: Alice wants to pay Bob _X_ Bitcoin using the LN but Bob does not generate the preimage - Alice does it.
1) Alice generates a secret preimage `p` and sends the hash of it `h(p)` to Bob.
2) Bob calls `addholdinvoice` with an amount `X` and the hashed preimage `h(p)` ([more details](https://github.com/lightningnetwork/lnd/blob/aa1cd04dbf07a9195d5ada752f383988d8d01fa7/cmd/lncli/invoicesrpc_active.go#L142)).
3) Bob sends the generated invoice to Alice.
4) Alice now can pay the invoice using `sendpayment`.
5) As soon as Alice is happy (because she is in a good mode), she tells Bob the secret (or he learns it otherwise, see below).
6) Bob settles the payment by calling `settleInvoice` ([more details](https://github.com/lightningnetwork/lnd/blob/aa1cd04dbf07a9195d5ada752f383988d8d01fa7/cmd/lncli/invoicesrpc_active.go#L53)).

The payment is complete.

## Research

> Assumption: Bob wants to receive _X_ Bitcoin (**A** Asset) on the LN (**alpha** Ledger) for _Y_ Ether (**B** Asset) on Ethereum (**beta** Ledger)
> Alice magically finds out about this _whish_ or using a [COMIT link](fill in some link).
>
> * Alpha Ledger: Lightning Network
> * Alpha Asset: Bitcoin on the Lighting Network
> * Beta Ledger: Ethereum
> * Beta Asset: Ether

### Step-by-step description

1) Alice generates a secret preimage `p` and sends a swap request to Bob, this includes (assets with amounts are emitted):

| Name                    | JSON Encoding       | Description                                                                                                     |
|:-------------------------|:---------------------|:--------------------------------------------------------------------------------------------------------------|
| `alpha_expiry`          | `u32`               | The UNIX timestamp of the time-lock for LN (alpha Ledger)                                                       |
| `beta_expiry`           | `u32`               | The UNIX timestamp of the time-lock on the beta HTLC                                                            |
| `alpha_refund_identity` | `α::Identity`       | This can be empty as the refund identity will be handled by LN.                                                 |
| `beta_redeem_identity`  | `β::Identity`       | The identity on beta that **B** will be transferred to when the beta-HTLC is activated with the correct secret  |
| `secret_hash`           | `hex-encoded-bytes` | The output of calling `hash_function` on the secret                                                             |

2) Bob _accepts_ the request and performs the following step:
    * Create a hold incoice using `addholdinvoice` with the amount `X` and the hashed preimage `secret_hash` ([more details](https://github.com/lightningnetwork/lnd/blob/aa1cd04dbf07a9195d5ada752f383988d8d01fa7/cmd/lncli/invoicesrpc_active.go#L142)).
    * Bob _subscribes_ to the invoice and waits for the payment either using
      * `SubscribeSingleInvoice` - this is not available through the CLI but as RPC; or
      * `LookupInvoice` - this is available trough the CLI but needs to be polled regularly.
3) Bob response to Alice's request with the following information:

| Name                    | JSON Encoding | Description                                                                                               |
|-------------------------|---------------|-----------------------------------------------------------------------------------------------------------|
| `alpha_redeem_identity` | `α::Identity` | The identity on alpha that **A**, this is the LN `invoice`  |
| `beta_refund_identity`  | `β::Identity` | The identity on beta that **B** will be transferred to when the beta-HTLC is activated after `beta_expiry`      |

4) Alice now starts the [exection phase](https://github.com/comit-network/RFCs/blob/master/RFC-003-SWAP-Basic.md#1-alice-deploys-%CE%B1-htlc) by paying the invoice using the LND command `sendpayment`.
5) Bob gets notified about funding of alpha (i.e. the invoice has been paid but cannot be settled yet), and continues with [deploying beta-HTLC](https://github.com/comit-network/RFCs/blob/master/RFC-003-SWAP-Basic.md#2-bob-deploys-%CE%B2-htlc), i.e. he deploys a HTLC on Ethereum.
6) As soon as beta has enough confirmations for Alice, she redeems the beta-HTLC using her secret.
7) Bob gets notified about this, learns the secret and can now settle the LND invoice by invoking the LND command `settleInvoice` ([more details](https://github.com/lightningnetwork/lnd/blob/aa1cd04dbf07a9195d5ada752f383988d8d01fa7/cmd/lncli/invoicesrpc_active.go#L53)).

The trade is complete.

## Spike Outcome

### Lightning Network: a new ledger
Similiar to the ledger definitions for [Bitcoin](https://github.com/comit-network/RFCs/blob/master/RFC-004-Bitcoin.md) and [Ethereum](https://github.com/comit-network/RFCs/blob/master/RFC-006-Ethereum.md) we need to handle the Lightning Network differently.
This is required because the comit-node and btsieve need to perform different actions accordingly.
We are always talking about Ledgers and Assets, (e.g. _Bitcoin_ Asset on the _Bitcoin_ Ledger, _Ether_ Asset on the _Ethereum_ Ledger, _Erc20_ on the _Ethereum_ Ledger, ...), Hence,
if we follow this approach, for supporting LN (through LND) we will need to introduce a new pair of Ledger and Asset:

* Ledger: the **Lightning Network**. _Ledgers_ are used as _settlement layers_ for our HTLCs. In the case of LND, this layer is the Lightning Network.
* Asset: **Bitcoin**. Since LN is a layer-2 network on top of Bitcoin, the asset should also be Bitcoin.

### Dealing with timeouts
When calling `addholdinvoice` an [`expiry`](https://github.com/lightningnetwork/lnd/blob/aa1cd04dbf07a9195d5ada752f383988d8d01fa7/cmd/lncli/invoicesrpc_active.go#L142) (in seconds) can be given. However, this timeout is relative and will get accumulated across the payment path.
Hence, Alice will need to know the path and this value in advance before sending a swap request to Bob as she won't (or should not) be able to change neither `alpha_expiry` nor `beta_expiry` after the swap request.

### Responsabilitites

A main goal of COMIT is to keep the autonomy to the user and let him/her decide when to deploy a HTLC, redeem or refund a HTLC, etc.
If a trade involves LN using LND we can aproach these things differently:


* Action
    * Create hold invoice
* Responsibility
    * LND
* Invoker
    * User, comit-i, comit-node
* Description
    * `addholdinvoice` is available as a RPC command or through the LND CLI. Althoug dealing with this is rather cumbersome, to keep the autonomy with the user, and to not introduce LND dependency into the comit-node, we this should be possible through comit-i.
* Conclusion:
    * comit-i needs LND support


---


* Action
    * Pay invoice
* Responsibility:
    * LND or LN Wallet
* Invoker
    * User, comit-i
* Description
    * To keep the autonomy to the user when to initiate a trade, we should return the invoice information through our API to the user (e.g. expose it through comit-i ) and let him/her pay the invoice.
* Conclusion
    * comit-i needs LND support



---


* Action:
    * Settle Invoice
* Responsibility
    * LND or LN Wallet
* Invoker
    * User, comit-i, comit-node
* Description
    * As soon as the secret has been learned, the HTLC on the LN should be settled using the command `settleinvoice`, this can either be done by the user (and exposed through comit-i) or done automatically through the comit-node. Since we have the extra _redeem_ step for Bitcoin and Ethereum (as well for Erc20) which needs to be performed from the user, we should leave the settlement of the invoice to the user (e.g. expose this information through comit-i).
* Conclusion
    * comit-i needs LND support



---


* Action
    * Monitor LN
* Responsibility
    * LND
* Invoker
    * Btsieve
* Description
    * Similar to other Ledgers we need to monitor LN for the payment (and later on settlement) of an invoice. To keep our current abstraction layer, this should be done through btsieve
* Conclusion
    * btsieve needs LND support


### COMIT link relation
Assumption: Bob is the creator of the link and is willing to receive Bitcoin for Ether for 1:10.

Bob cannot yet create a hold invoice, i.e. `addholdinvoice` as he does not know the secret yet, hence, all he can add in the link is the information about Ledgers, Assets and exchange rates.

### Fall-back mechanism of LN
LN allows to specificy a fallback address (_fallback_addr_) in when creating calling `addholdinvoice`.
We could use this information to fall back to an on-chain HTLC trade if no route can be found between Alice and Bob.
