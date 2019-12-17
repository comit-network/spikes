# Basic model to calculate expiry times

Tracking issue: [#14](https://github.com/comit-network/RFCs/issues/14)

## Context

The protocol defined in [RFC003](https://github.com/comit-network/RFCs/blob/master/RFC-003-SWAP-basic.md) introduces the concept of expiry times, which mark the time when assets can be refunded to their original owners.
Values that are reasonable and that ensure that the protocol is secure for both parties must be estimated.

## General Findings

- The difference between `alpha_expiry` and `beta_expiry` has an effect on the security of the model for *Bob*. This does not affect Alice since she will have already redeemed `beta_asset` by the time this is relevant.
- `beta_expiry` has an effect on the time that *Alice* will have to redeem. Therefore, it only affects the likelihood that Alice decides not to redeem the beta HTLC because she perceives it to be too risky (we ignored the [option problem](https://coblox.tech/docs/financial_crypto19.pdf) for this analysis).
- `alpha_expiry` and `beta_expiry` can be modelled independently because `beta_expiry` is just a parameter to the model for `alpha_expiry`
- We couldn't produce precise expressions of the expiry times, but we were able to approximate them using numerical methods in python

## Model

The basic model we came up with is here: https://www.overleaf.com/read/sswbyytnfjhy or here in [PDF format](assets/basic_expiry_model.pdf).  (It was too difficult to express in markdown.) 


The model can produce meaningful estimates (subject to the limitations mentioned) but we need to figure out how to actually evaluate it and come up with a quantile function for the random variables representing the time to redeem.
I've left this for another spike, but an initial naive attempt can be seen below.

## Outdated Example Estimation code

** THIS CODE WAS MADE WHEN THE MODEL WAS MORE NAIVE THAN IT WAS IN THE LINK. THE RESULTS DON'T REFLECT ANYTHING BUT IT IS KEPT HERE IN CASE IT'S USEFUL LATER ON! **

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
# Probability of Bob redeeming beta asset before beta expiry
beta_redeem_probability = 0.999999
# Time taken for k blocks to be generated on Ethereum (beta ledger) with beta_redeem_probability
time_for_confirmations_beta = erlang.ppf(beta_redeem_probability, alice_confirmations, 0, beta_block_time)
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
# Probability of Alice redeeming alpha asset before alpha expiry
alpha_redeem_probability = 0.999
# Time taken for k blocks to be generated on Ethereum (beta ledger) with alpha_redeem_probability
time_for_confirmations_alpha = erlang.ppf(alpha_redeem_probability, bob_confirmations, 0, alpha_block_time)
print("Time for %i confirmations on alpha ledger: %f" % (bob_confirmations, time_for_confirmations_alpha))

alpha_expiry = beta_expiry + bob_alpha_time + time_for_confirmations_alpha
print("Alpha expiry: %f" % alpha_expiry)
```

#### Output

```
Time for 40 confirmations on beta ledger: 19.385065
Beta expiry: 42.885065
Time for 6 confirmations on alpha ledger: 164.547452
Alpha expiry: 237.432517
```
