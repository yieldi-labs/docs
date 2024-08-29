# Overview

What would a full "drag and drop app-chain for Web2" look like?

Premises
1) A web 2 company has 100m users and want to go on-chain
2) Minimal concern to Security, Liquidity, Yield, Economics
3) Zero-reliance on CEX infra
4) Pay the lowest cost for security, maximum returns to equity stakeholders accessing cashflows from users paying fees

CashBack Application Token "CBAT"
1) Stake the Application Token in a vault on Yieldi
2) USDC is streamed back to stakers (from network fees or bootstrap incentives)

App-chain:
1) Gold standard Cosmos and highly-performant Tendermint (gordian?)
2) IBC support with in-built relayer to Yieldi
3) CW support for custom Apps
4) USDC Bridge support (from EVM or IBC), supply mint/burn managed by Yieldi
5) Configurable Token Supply and Emission Stream

Security
1) Register as AVS, get delegated security from ETH/BTC immediately
2) ETH/BTC stakers delegate security, earn "native yield" streamed to them
3) Incentive Pendulum detects Security Delegated vs USDC Bridged, keeps it 2:1 (Twice as much Security as TVL) via the Yield split. 

Liquidity
1) Launch Liquidity Auction (LLA) via Yieldi Liquidity Pool (anyone can bid on the initial supply of the App-token with USDC)
2) Yield streaming of ETH/BTC security rewards to LST (Security)
3) Yield streaming of USDC to App-token stakers (Equity)

# Lifecyle
1) New AVS publishes deck on Yieldi Marketplace (prospectus with forecasted user growth, rewards, fees)
2) Deploys Yieldi-based AVS infra (App-chain, IBC Relay to Yieldi)
3) Receives delegated security from ETH/BTC
4) Mints supply, sells 5-10% via a LLA on Yieldi to receive USDC and pricing.
5) App-stakers can buy the App-token and stake in the Real Yield Vault.
6) Begins emission, points program for users to go on-chain, accumulate a USDC balance
7) Yield streams to Security & App-stakers, Incentive Pendulum balances the two
8) Convert to fees from users
9) Sustainable



<img width="715" alt="image" src="https://github.com/user-attachments/assets/2e41b3fb-7e2d-47e6-92ed-e20bd03d00b0">

# Yieldi As an LSP itself

Yieldi can manage the staked ETH/BTC itself `yETH` `yBTC`

* Fork Eigenlayer for `yETH`
* Fork Babylon for `yBTC`

Main problem to solve is double-sign slashing, but Yieldi Operators can double up as the slashing committee and receive income when slashing. 

# Yieldi As an AVS itself

Yieldi itself uses its own Infra to be its own yield-streaming AVS

1) Security delegated to it
2) Yieldi Application Stakers get the cashflows from Yieldi yield-streaming the tokens of all other AVS (100-200 bps fees)
3) Incentive Pendulum balances Security with the total TVL on Yieldi (USDC, App tokens, Slashable balances)







