# Basic model to calculate expiry times

* Status: proposed
* Authors: [Lloyd Fournier](@LLFourn), [Lucas Soriano](@luckysori)
* Date: 2019-03-05

Tracking issue: [#14](https://github.com/comit-network/RFCs/issues/14)

## Context

The protocol defined in [RFC003](https://github.com/comit-network/RFCs/blob/master/RFC-003-SWAP-basic.md) introduces the concept of expiry times, which mark the time when assets can be refunded to their original owners.
Values that are reasonable and that ensure that the protocol is secure for both parties must be estimated.

## Research

### Findings

Here are the results of initial discussions:

- The difference between `alpha_expiry` and `beta_expiry` has an effect on the security of the model for *Bob*.
This does not affect Alice since she will have already redeemed `beta_asset` by the time this is relevant.
- `beta_expiry` has an effect on the time that *Alice* will have to redeem `beta_asset` and therefore on the likelihood that the protocol will be completed successfully.
This does _not_ have any direct effect on the security of the protocol, but rather on its *execution guarantees*.

Given this, the topic of this spike changed from creating a security model to, more generally, creating a basic model to calculate expiry times.

### Alpha expiry

There are two ways in which `alpha_expiry`, <img src="https://latex.codecogs.com/gif.latex?E_{\alpha}"/>, has an effect on the protocol:

1. How long Bob will (at least) have to redeem `alpha_asset` if Alice redeems `beta_asset`.
2. How long Alice will have to wait to be able to refund `alpha_asset` if the protocol isn't carried out to completion.

It would therefore be ideal to find a value of `alpha_expiry` that lets Bob redeem `alpha_asset` with high probability while also ensuring Alice doesn't have to wait too long to be able to be refunded if it comes to that.

#### Assumptions

- Bob provides <img src="https://latex.codecogs.com/gif.latex?T_{\alpha}"/>, the time in which he is overwhelmingly confident he will be able to confirm a transaction on `alpha_ledger`.
He has direct influence over this since he can choose the fee.
- Bob provides <img src="https://latex.codecogs.com/gif.latex?k"/>, the number of additional confirmations he would need to have on his redeem transaction before he considers `alpha_asset` to be his own.
- Block time on `alpha_ledger` can be modelled as a random variable that follows an exponential distribution with parameter <img src="https://latex.codecogs.com/gif.latex?\lambda"/>.
- A confirmation is given by the arrival of a new block.
- The value of `beta_expiry`, <img src="https://latex.codecogs.com/gif.latex?E_{\beta}"/>, is constant.

#### Model

To determine `alpha_expiry`, the time <img src="https://latex.codecogs.com/gif.latex?\Delta_{R}"/> Bob will need to get his desired number of confirmations on his redeem transaction of `alpha_asset` will need to be estimated.

The time it would take for Bob's redeem transaction to obtain <img src="https://latex.codecogs.com/gif.latex?k"/> confirmations on `alpha_ledger` can be modelled as a sum of <img src="https://latex.codecogs.com/gif.latex?k"/> mutually independent exponential random variables with the same rate parameter <img src="https://latex.codecogs.com/gif.latex?\lambda"/>, which is equal to an <img src="https://latex.codecogs.com/gif.latex?\textrm{Erlang}(k,\lambda)"/> distribution.

Letting <img src="https://latex.codecogs.com/gif.latex?C_{k}\sim\textrm{Erlang}(k,\lambda)"/>,

<img src="https://latex.codecogs.com/gif.latex?\Delta_{R}=T_{\alpha}&plus;Q_{C_{k}}(p)"/>,

where <img src="https://latex.codecogs.com/gif.latex?Q_{C_{k}}(p)"/> is the [quantile function](https://en.wikipedia.org/wiki/Quantile_function) of <img src="https://latex.codecogs.com/gif.latex?C_{k}"/> for a probability <img src="https://latex.codecogs.com/gif.latex?p"/>. 
This is, a function returning the minimum time taken for <img src="https://latex.codecogs.com/gif.latex?C_{k}"/> to take place with probability <img src="https://latex.codecogs.com/gif.latex?p"/>.

Quantile functions for Erlang random variables can only be approximated via [numerical methods](https://docs.scipy.org/doc/scipy-0.16.1/reference/generated/scipy.stats.erlang.html).

Knowing all this, `alpha_expiry` can be estimated:

<img src="https://latex.codecogs.com/gif.latex?E_{\alpha}=E_{\beta}&plus;T_{\alpha}&plus;Q_{C_{k}}(p)"/>,

where the value of <img src="https://latex.codecogs.com/gif.latex?p"/> is directly proportional to the security of the model (and the size of `alpha_expiry`).

### Beta expiry

There are two ways in which `beta_expiry`, <img src="https://latex.codecogs.com/gif.latex?E_{\beta}"/>, has an effect on the protocol:

1. How long Alice will (at least) have to redeem `beta_asset` if Bob funds `beta_asset`.
2. How long Bob will have to wait to be able to refund `beta_asset` if Alice has yet to redeem it.

Consequently, it would be ideal to find a value of `beta_expiry` that lets Alice redeem `beta_asset` with high probability, allowing the protocol to continue, while also ensuring Bob doesn't have to wait too long to be able to be refunded if it comes to that.

#### Assumptions

- Alice provides <img src="https://latex.codecogs.com/gif.latex?T^{\alpha}_{A}"/>, the time in which she is overwhelmingly confident she will be able to confirm a transaction on `alpha_ledger`.
She has direct influence over this since she can choose the fee.
- Bob provides <img src="https://latex.codecogs.com/gif.latex?T^{\beta}_{B}"/>, the time in which he is overwhelmingly confident he will be able to confirm a transaction on `beta_ledger`.
He has direct influence over this since he can choose the fee.
- Alice provides <img src="https://latex.codecogs.com/gif.latex?T^{\beta}_{A}"/>, the time in which she is overwhelmingly confident she will be able to confirm a transaction on `beta_ledger`.
She has direct influence over this since she can choose the fee.
- Alice provides <img src="https://latex.codecogs.com/gif.latex?k"/>, the number of additional confirmations she would need to have on her redeem transaction before she considers `beta_asset` to be her own.
- Block time on `beta_ledger` can be modelled as a random variable that follows an exponential distribution with parameter <img src="https://latex.codecogs.com/gif.latex?\lambda"/>.
- A confirmation is given by the arrival of a new block.

#### Model

To determine `beta_expiry`, the random variable <img src="https://latex.codecogs.com/gif.latex?R_{\beta}"/>, which models the time Alice will need to get her desired number of confirmations on her redeem transaction of `beta_asset`, will need to be approximated.

The time it would take for Alice's redeem transaction to obtain <img src="https://latex.codecogs.com/gif.latex?k"/> confirmations on `beta_ledger` can be modelled as a sum of <img src="https://latex.codecogs.com/gif.latex?k"/> mutually independent exponential random variables with the same rate parameter <img src="https://latex.codecogs.com/gif.latex?\lamda"/>, which is equal to an <img src="https://latex.codecogs.com/gif.latex?\textrm{Erlang}(k,\lambda)"/> distribution.

Letting <img src="https://latex.codecogs.com/gif.latex?C_{k}\sim\textrm{Erlang}(k,\lambda)"/>,

<img src="https://latex.codecogs.com/gif.latex?E_{\beta}=t_{0}&plus;T^{\alpha}_{A}&plus;T^{\beta}_{B}&plus;T^{\beta}_{A}&plus;Q_{C_{k}}(p)"/>, [1]

where <img src="https://latex.codecogs.com/gif.latex?t_{0}"/> is the time when all the protocol parameters are set and <img src="https://latex.codecogs.com/gif.latex?Q_{C_{k}}(p)"/> is the [quantile function](https://en.wikipedia.org/wiki/Quantile_function) of <img src="https://latex.codecogs.com/gif.latex?C_{k}"/> for a probability <img src="https://latex.codecogs.com/gif.latex?p"/>.
This is, a function returning the minimum time taken for <img src="https://latex.codecogs.com/gif.latex?C_{k}"/> to take place with probability <img src="https://latex.codecogs.com/gif.latex?p"/>.

As mentioned in the previous section, quantile functions for Erlang random variables can only be approximated via [numerical methods](https://docs.scipy.org/doc/scipy-0.16.1/reference/generated/scipy.stats.erlang.html).


With this knowledge, `beta_expiry` can be estimated using [1].

### Example

#### Code

```python
from scipy.stats import erlang

# beta expiry
# E_{\beta} = t_{0} + T^{\alpha}_{A} + T^{\beta}_{B} + T^{\beta}_{A} + Q_{C_{k}}(p)

# Time when all parameters will be set and the protocol will commence
t_0 = 0
# Time in which Alice is overwhelmingly confident she will be able to confirm a Bitcoin (alpha ledger) transaction
alice_alpha_time = 20
# Time in which Bob is overwhelmingly confident he will be able to confirm an Ethereum (beta ledger) transaction
bob_beta_time = 2
# Time in which Alice is overwhelmingly confident she will be able to confirm an Ethereum (beta ledger) transaction
alice_beta_time = 1.5
# Number of additional confirmations Alice needs on her redeem transaction of beta_asset
alice_confirmations = 40
# Ethereum (beta ledger) block time
beta_block_time = 0.25
# Time taken for k blocks to be generated on Ethereum (beta ledger) with probability 0.95
time_for_confirmations_beta = erlang.ppf(0.999999, alice_confirmations, 0, beta_block_time)
print("Time for %i confirmations on beta ledger: %f" % (alice_confirmations, time_for_confirmations_beta))

beta_expiry = t_0 + alice_alpha_time + bob_beta_time + alice_beta_time + time_for_confirmations_beta
print("Beta expiry: %f" % beta_expiry)

# alpha expiry
# E_{\alpha} = E_{\beta} + T^{\alpha}_{B} + Q_{C_{k_{B}}}(p)

# Time in which Bob is overwhelmingly confident he will be able to confirm a Bitcoin (alpha ledger) transaction
bob_alpha_time = 30
# Number of additional confirmations Alice needs on her redeem transaction of beta_asset
bob_confirmations = 6
# Ethereum (beta ledger) block time
alpha_block_time = 10
# Time taken for k blocks to be generated on Ethereum (beta ledger) with probability 0.95
time_for_confirmations_alpha = erlang.ppf(0.999999, bob_confirmations, 0, alpha_block_time)
print("Time for %i confirmations on alpha ledger: %f" % (bob_confirmations, time_for_confirmations_alpha))

alpha_expiry = beta_expiry + bob_alpha_time + time_for_confirmations_alpha
print("Alpha expiry: %f" % alpha_expiry)
```

#### Output

```
Time for 40 confirmations on beta ledger: 19.385065
Beta expiry: 42.885065
Time for 6 confirmations on alpha ledger: 254.126261
Alpha expiry: 327.011326
```
