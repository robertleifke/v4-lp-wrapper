# Bunni

Modified version of bunni to offer fungible ERC-20 tokens on Uniswap V4 LP shares.

## Solution

Bunni enables projects to create ERC-20 LP tokens on Uniswap V3 and plug them into existing liquidity incentivization contracts designed for Uniswap V2/Sushiswap.

1. Project creates Bunni LP token contract and staking contract for incentivizing LPs
2. LPs provide liquidity using Bunni and get LP tokens
3. Users stake LP tokens in staking contract and earn incentives

It's just like how liquidity mining works using good'ol Uniswap V2!

By using Bunni instead of Uniswap V2, projects can merge their liquidity into one Uniswap V3 pool instead of splitting it between Uniswap V2 and V3. This means:

- Traders experience less slippage, since the liquidity is no longer fractured
- LPs feel more comfortable providing liquidity & using more complex strategies on Uniswap V3, since they know most of the trading volume will go through Uniswap V3.

## Features

- ERC-20 LP token as fractional shares of a Uniswap V3 LP position
- `compound()` for compounding trading fees back into the liquidity pool
- Arbitrary price range (though we expect most projects to use full range due to its simplicity)
- Hub-spoke architecture where the LP logic is built into the hub and the LP tokens are basic ERC-20 tokens with minimal additions. This makes deploying new Bunni LP tokens much cheaper, as well as concentrates user token approvals to the hub to save gas on approvals.

## Get Started

Install [Foundry](https://github.com/foundry-rs/foundry), then

```
npm i -D
npm run prepare
forge install
forge test
```

## Known issues

### Frontrunning the first deposit may steal 1/4 of the deposit

When a new Bunni token is created and the first deposit is made to the Bunni token, it is possible for an attacker to frontrun it to steal ~1/4 of the deposit. Let the user's deposit liquidity amount be `L`.

- Attacker mints `MIN_INITIAL_SHARES` Bunni tokens.
- Attacker burns `MIN_INITIAL_SHARES - 1` Bunni tokens, resulting in a total supply of 1.
- Attacker mints `L/2 + 1` liquidity to the Bunni token's Uniswap v3 position. This is possible because anyone can increase the liquidity of anyone else on Uniswap.
- The user deposits `L` liquidity, giving them `L / (L/2 + 1) = floor(1.99...) = 1` Bunni tokens.
- Now if the user burns the 1 Bunni token, they would receive `1/2 * (L + L/2 + 1) = floor(3L/4 + 1/2) = 3L/4` liquidity, meaning they lost `L/4` liquidity.

While this attack is theoretically possible, it does not pose a practical problem.

- The [Bunni website](https://bunni.pro) combines Bunni token creation and the first deposit into a single multicall, meaning it's impossible for an attacker to insert a transaction in between the two actions.
- The [Bunni Zap](https://github.com/timeless-fi/bunni-zap) contract provides a `sharesMin` parameter, allowing the user to specify the minimum number of Bunni tokens received. This means if an attacker attempted to perform the aforementioned attack, the transaction will simply revert.

However, smart contract integrators should be aware of this problem to avoid losing funds.

### Compound calls can be sandwiched

An attacker can perform a sandwich attack on compound calls by submitting the following package of transactions:

- Buy a token from the Uniswap pool underlying the Bunni token
- Compound fees earned by the Bunni token LPs into more liquidity
- Sell the token back for slightly more tokens than started with

This is the result of the intentional design decision to keep compounding permissionless, so there's no perfect solution to it. Realistically it's not going to be a problem since the amount of funds being compounded is usually small (on the same order of magnitude as the gas cost of compounding).
