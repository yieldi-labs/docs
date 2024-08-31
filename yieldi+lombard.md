# Context

## STETH As Case Study
* ETH: [$300bn](https://www.coingecko.com/en/coins/ethereum)
* WETH: [$7bn](https://www.coingecko.com/en/coins/weth)
* STETH: [$25bn](https://www.coingecko.com/en/coins/lido-staked-ether)

STETH is the yield-bearing staked ETH derivative with counter-party and slashing risk, yet is 4 times larger than wrapped ETH, the non-yield bearing 0-risk Wrapped ETH. 

[ETH Economics](https://defillama.com/chain/Ethereum)
* ETH DeFi TVL: ~$20bn 
* ETH Daily Fees: $1m
* STETH Yields: ~4% APR

## BTC As Opportunity
* BTC: [1.1tn](https://www.coingecko.com/en/coins/bitcoin)
* WBTC: [9bn](https://www.coingecko.com/en/coins/wrapped-bitcoin)
* LBTC:???

What is the yield-bearing staked BTC derivative? This is a $30bn-$50bn opportunity today. 

# Premises
1) App-chains, L2s, AVS's to onboard $20bn+ in TVL to sustain Fees/Rewards to pay yield to $20bn+ in lBTC
2) Target Yield 2% requires $1m/day in fees/rewards
3) Fees/rewards streamed back to lBTC to rebase

# Technical Architecture
1) Lombard provices the BTC Staking -> LBTC issuance
2) Lombard provides AVS Delegation Dashboard
3) Yieldi provides bridging for TVL
4) Noble provides the USDC supply
5) Yieldi yield-streams fees/tokens via THORChain back to Lombard
6) Lombard re-bases the LBTC with the yield (will need wLBTC a non-rebasing alternative)

<img width="674" alt="image" src="https://github.com/user-attachments/assets/af7b9d65-711d-4152-8469-490afe731b73">

