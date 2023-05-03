# Dipassio Solidity 
# Audits Tech


# Oracle manipulation

This repository investigates price oracle manipulation, a typical DeFi attack method. This should not be used in production, whether intentionally or not. It is only intended for instructional purposes and comes with no warranties or guarantees.

## Context

Oracle manipulation is a pretty typical DeFi attack method that has given rise to a few notable exploits. The problem arises from employing the quantity of tokens on each side of an AMM liquidity pool to establish the spot exchange rate as a pricing oracle. As an illustration, the spot price for an AMM liquidity pool of 1 ETH and 3000 USDC is $3000 USDC per 1 ETH.

The spot price of this kind of naive price oracle can be manipulated by bad actors by making large trades, which can lead to a variety of inventive exploits for smart contracts that depend on the oracle for critical operations. This vulnerability is exacerbated if the oracle is used with a shallow liquidity pool. These kind of attacks are made easier to carry out thanks to flash loans, which make them available to organizations with small pools of funds.

These articles walk through a few instances of real DeFi attacks including the manipulation of pricing oracles:

- https://hackernoon.com/how-dollar100m-got-stolen-from-defi-in-2021-price-oracle-manipulation-and-flash-loan-attacks-explained-3n6q33r1
- https://medium.com/meter-io/the-bzx-attacks-what-went-wrong-and-the-role-oracles-played-in-the-exploits-264619b9597d

## Details

These contracts are meant to serve as a theoretical illustration of how a loan protocol could be drained by manipulating a price oracle of that kind. The demonstration simulates the use of a flash loan and uses a basic implementation of an AMM and a lending protocol. There are many more restrictions in practice, like borrowing limits, fees, and other restrictions. The input and initialization settings are arbitrary and simply intended as a proof of concept.

Setup:

- Simple ETH/USDC AMM liquidity pool
  - Initialized with 1 ETH and 3000 USDC, implying a price of $3000/ETH
- Basic lending protocol
  - Accepts USDC deposits and lends ETH based on a collateralization ratio of 0.8
  - Uses a (vulnerable) price oracle that retrieves the USDC/ETH price based on the above AMM pool
  - Initialized with 5 ETH as reserves

Execution:

1. Flash loan 2 ETH
2. Swap 2 ETH for USDC in AMM (receive 2000 USDC)
3. Deposit 2000 USDC into lending protocol
4. Borrow max amount of ETH against deposited USDC (4.8 ETH)
5. Repay flash loan of 2 ETH and keep profits (2.8 ETH)

How it works:

Even though the AMM itself is decentralized, the naive usage of an AMM pool as a pricing oracle effectively creates a centralized oracle that is open to manipulation.

As a relatively large trade on a pool with very little liquidity, the swap in step 2 has a significant impact on the ratio of tokens in the pool and, as a result, on the relative price of the token the oracle returns. The pool now has 3 ETH and 1000 USDC, translating to a spot price of 1 ETH = 1000/3 USDC, or $333 per ETH, after the swap.

At a price of $3000 per ETH, you should only be able to borrow 0.8 x $2000 ($2000 / $3000) = 0.528 ETH when using USDC as collateral. However, because the manipulated AMM pool reserves' price oracle for the lending protocol returns $333 per ETH, it allows you to borrow up to 0.8 * ($2000 / $333) = 4.8 ETH.

The attack pattern is shown in `test/OracleAttack.test.js`, which runs through the steps to execute the attack in mulitple separate transactions for step-by-step explanations, as well as all in one transaction through `Attacker.sol`. 

## Mitigation

Using a decentralized price oracle that calculates the genuine price by applying some sort of averaging across numerous (deep liquidity) pools is a typical strategy to avoid these kinds of centralized points of vulnerability. This is far more difficult to manipulate because it is almost impossible to find enough money, especially with deeper liquidity pools, to dramatically alter the price of many pools.

Utilizing time-weighted average price (TWAP) oracles is an alternative strategy that lowers the risk of single-transaction flash loan attacks but trades some accuracy during high volatility periods. TWAP oracles take the asset's average price over a predetermined period of time. Although it is more difficult, manipulating these mechanisms across multiple blocks is still possible.
