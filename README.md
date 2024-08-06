# Technical Architecture
📜 Technical Architecture of Yieldi

<img width="667" alt="image" src="https://github.com/user-attachments/assets/17121a21-166b-4603-8340-ad9c22e64e34">



# Table of Contents

- [Overview](#overview)
    - [Terms](#Terms)
    - [What is Yieldi?](#what-is-Yieldi?)
    - [How does it work?](#How-does-it-work?)
    - [What problem does it solve?](What-problem-does-it-solve?)
- [Diagram](#diagram)
    - [LSP: Staking-Restaking](#LSP-Staking-Restaking)
    - [AVS: Yield-streaming](#AVS-Yield-streaming)
    - [User: Yield-claiming](#User-Yield-claiming)
    - [User: Yield-claiming](#User-Yield-claiming)
- [Implementation](#development)
    - [LSP Staking](#LSP-Staking)
    - [AVS Reference Implementationn](#AVS-Reference-Implementation)
    - [Yieldi WASM Contracts](#Yieldi-WASM-Contracts)
    - [THORChain Yield Accounts](#THORChain-Yield-Accounts)
- [Economics](#Economics)
    - [One-time Liquidity Auction](#One-time-Liquidity-Auction)
    - [Liquidity Mining](#Liquidity-Mining)
    - [BTC LSP Support](#BTC-LSP-Support)
- [Summary](#Summary)
- [Technical Readiness](#Technical-Readiness)


# Overview

This document is a guide to understanding the design of Yieldi, with a clear and detailed description of the various components, systems, and technologies that make up the solution.

## Terms:

- `AVS` Active Validator Set for an app-chain
- `IBC` Inter-Blockchain Communication protocol
- `LST` Liquid Staking Token
- `LSP` Liquid Staking Protocol

## What is Yieldi?

Yieldi offers a gas-efficient yield-streaming solution for the [Eigenlayer](https://eigenlayer.xyz/) and [Babylon](https://babylonlabs.io)) ecosystem, and it will initially be deployed on [Thorchain](https://thorchain.org/) which has native ETH/BTC liquidity. Users can re-stake ETH with Eigenlayer, stake BTC with Babylon and delegate their security to an AVS. The AVS can then stream yield back to the staker natively using Yieldi. 

Yieldi also solves for price-discovery and liquidity of AVS tokens, and will increase the propensity of users to delegate LST to AVS operators if they are paid real-yield in native assets (ETH). It provides the lowest cost of security, because it offers the highest stability and least friction to the staker. Lastly it ensures that the full cycle of yield collection from AVS is conducted with minimal trust assumptions and third-party dependencies. 

## How does it work?

Stakers deposit into the LSP with native Assets (ETH/BTC). The AVS can read and compute the user's share of the yield, then send it via IBC to THORChain on some interval. THORChain is able to swap to native assets and hold in yield collector module with the owner set to the staker's L1 address. The Staker can then view their yield and claim it any time. Additionally, the Staker may opt-in for auto-streaming, which causes THORChain to stream out the entire balance on a gas-efficient interval. 

## What problem does it solve?

Stakers are much more likely to delegate their LSTs to AVS operators who can pay "real yield" which is realised in native ETH/BTC instead of an illiquid rewards token. Stakers will see their rewards accrued in a native balance which they can claim at any time. They will be able to compute their annualised yields and make informed decisions about their capital. 

Because the yield is lower risk, and in an asset delivered to the user, removing friction, the cost of yield will be much lower. Thus AVS's will naturally prefer yield-streaming because it will require less inflation and they can transition to the fee regime faster, avoiding security gaps. 

# Diagram

The following diagrams for each process provide a visual representation of the asset flow. 

## LSP Staking-Restaking

<img width="775" alt="image" src="https://github.com/user-attachments/assets/2a6123ce-326b-4950-9446-a1c01b27e284">


## AVS Yield-streaming

<img width="774" alt="image" src="https://github.com/user-attachments/assets/c9f33042-0d10-4c0f-96e4-9dd7f2862ba2">


## Yieldi Yield-collecting

<img width="772" alt="image" src="https://github.com/user-attachments/assets/d1854e2d-b923-424e-8b13-df587f55bc6e">


## User Yield-claiming

<img width="775" alt="image" src="https://github.com/user-attachments/assets/fd3783df-ded1-492e-989f-70ba6b5a2744">


# Implementation

## LSP Staking

When the user deposits their LST and delegates it to an AVS operator in a token pool, their address and amount is stored in the [Eigenlayer contract](https://www.blog.eigenlayer.xyz/ycie/): 

```js
contract TokenManager {
    mapping(address => address) tokenPoolRegistry;
    mapping(address => mapping(address => uint256)) stakerPoolShares;
    
    function stakeToPool(address pool, uint256 amount);
    function withdrawFromPool(address pool);
}

contract TokenPool {
    uint256 public totalShares;
    function stake(uint256 amount) TokenManagerOnly;
    function withdraw(uint256 shares) TokenManagerOnly;
}

contract DelegationManager {
    // ...
    mapping(address => mapping(address => uint256)) operatorPoolShares;
    // ...
}
```

Babylon users will lock to a [BIP322 script](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki) parsed by [Babylon Chain](https://github.com/babylonchain/simple-staking). 

```
 signMessageBIP322 = async (message: string): Promise<string> => {
    if (!this.walletInfo) {
      throw new Error("Wallet not connected");
    }
    return await window?.wallet?.bitcoinSignet?.signMessage(
      message,
      "bip322-simple",
    );
  };
```

## AVS Reference Implementation

The AVS can use `gRPC` methods to retrieve and compute the share of the yields to pay to each user from the LSP. 

The AVS then mints/deposits the rewwards and sends it to the destination user. 

```js
contract YieldManager {
    uint256 public accumulatedYield; // Temporary balance of yield (points, tokens, fees)

    function depositYield(uint256 amount) public; // Deposit yield for accumulation

    function sendYield(address user, uint256 amount) onlyOperator; //AVS sends yield per user

    function batchSendYield(address[] memory users, uint256[] memory amounts) onlyOperator; //AVS batch sends yield
}
```

### EVM to IBC

The AVS must have a deployed [gateway contract](https://docs.axelar.dev/dev/cosmos-gmp) which can lock tokens and emit an IBC-compatible message. 
The IBC relayer mints IBC-tokens and then forwards to THORChain via an IBC channel. 

Only the Staker's L1 address and the yield amount is sent (in yield tokens). 

```js
bytes memory argValue = abi.encode(recipients); // A standard EVM payload
bytes memory payload  = abi.encode(
    "multi_send", // CosmWasm method name
    StringArray.fromArray1(["recipients"]), // argument name
    StringArray.fromArray1(["string[]"]), // argument type
    argValue // argument value
);
bytes memory payloadToCW = abi.encodePacked(
    bytes4(uint32(1)), // version number
    payload
);
function callContractWithToken(
    string memory destinationChain,
    string memory contractAddress,
    bytes memory payloadToCW,
    string memory symbol,
    uint256 amount
) external;
```

### IBC Chains 

AVS which are natively IBC compatible simply need to emit the IBC packet. 
They set the destination IBC channel as THORChain's and pass the final user L1 address through the memo" 

```go
// send from AVS to TC
    memo = "yield+:asset:userAddress"
	msg = types.NewMsgTransfer(pathAVStoTC.EndpointA.ChannelConfig.PortID, pathAVStoTC.EndpointA.ChannelID, coinSentFromAVSToTC, suite.chainAVS.SenderAccount.GetAddress().String(), suite.chainTC.SenderAccount.GetAddress().String(), memo, timeoutHeight, 0)
	res, err = suite.chainB.SendMsgs(msg)
	suite.Require().NoError(err) // message committed
```

## Yieldi WASM Contracts

### IBC Swap to TOR
Yieldi receives the IBC payload and executes a [Swap](https://github.com/Team-Kujira/kujira-rs/blob/master/packages/kujira-fin/src/execute.rs) to native ETH/BTC. 

```rs
ExecuteMsg
Swap {
        offer_asset: Option<Coin>,
        belief_price: Option<Decimal256>,
        max_spread: Option<Decimal256>,
        to: Option<Addr>,
        #[serde(skip_serializing_if = "Option::is_none")]
        callback: Option<CallbackData>,
    }
```

### Deposit to ETH Yield Account
The callback would then stream to ETH/BTC to the user's Yield Account: 

```md
MsgDeposit{
       yield+:{nativeAsset}:{userAddress}
}
```

## THORChain Yield Accounts

The user now has a Yield Account on THORChain. This can be queried anytime to see accrued yield:

```md
https://thornode.ninerealms.com/thorchain/yield/account/0xb00E81207bcDA63c9E290E0b748252418818c869
{
asset: "ETH~ETH",
units: "1040077124",
owner: "0xb00E81207bcDA63c9E290E0b748252418818c869",
auto-stream: "0",
last_withdraw_height: 17111060
}
```

The user can `CLAIM` their yield at anytime by interating with [THORChain's Router](https://gitlab.com/thorchain/thornode/-/blob/develop/chain/ethereum/contracts/THORChain_Router.sol)
```md
function depositWithExpiry(address payable vault, address asset, uint amount, string memory memo, uint expiration) public

MEMO: CLAIM // Claim all the yield
MEMO: CLAIM:5000 // Claim 50% of the yield
```

### Auto-stream

THORChain has a [Collector Module](https://gitlab.com/thorchain/thornode/-/merge_requests/2978) that holds assets until they reach a value that is 100x the `outboundGasCost` for that chain. On a regular interval, it will auto-send the full balance to the user. This efficiently sends native yield directly back to the user. 

The user has an option to specify their stream interval when interacting with THORChain:
```
MEMO: CLAIM::100 // Claim all the yield and set auto-stream to 100x the gas cost (99% execution)
MEMO: CLAIM::1000 // Claim all the yield and set auto-stream 1000x the gas cost (99.9% execution)
MEMO: CLAIM::0 // Claim all the yield and set auto-stream off
```

# Economics

## Problems

AVS's have two other problems that Yieldi solves:
1) Price discovery of the AVS token
2) On-chain liquidity of the AVS token

<img width="774" alt="image" src="https://github.com/user-attachments/assets/b6fdc536-ff02-47fa-9a92-efdfac230bb9">


Without an on-chain price of the AVS token, the user struggles to understand the cost of capital and risk in delegating their LRT/LST.
For the AVS, illiquidity on launch will create price volatility and may struggle to transition to the fee regime effectively. If the AVS cannot transition to the fee regime they may suffer in security.  

Thus AVS require the following features provided by Yieldi:

1) Initial liquidity in the `UST:AVS` token pool in Yieldi
2) Ongoing liquidity incentives to ensure sufficient liquidity for users

Over time, the operators with the most liquidity, stable price and economic activity on their chains can attract the highest quality delegated security.  

## One-time Liquidity Auction

> AVS operators need a liquidity venue for their assets with an established price

A liquidity auction can be conducted by the AVS when setting up the channel. 

1) Mint 5-10% of the supply into Yieldi as a CW-20 token
2) Offer this in the new `UST:AVS` token pool
3) Over 7-30 days, anyone can deposit `TOR` to match the `AVS` token and infer the launch price of the asset.
4) When the pool goes live, the AVS will own 50% of the pool, and Liquidity Auction participants will own the other 50%.

> UST is a stablecoin on the THORChain protocol. 

## Liquidity Mining

> AVS should incentivise sufficient liquidity to stablise price and ensure best execution

The AVS should continually stream yield incentives to the `UST:AVS` pool as it will be the primary liquidity venue for the AVS token. 

1) The AVS can stream token incentives through the IBC channel with destination the `TOR:AVS` pool.
2) The incentives are added into the pool which are credited to the pool LPs.

## BTC LSP Support

Yield collection for BTC LSPs are also possible. 
1) BTC locked in a script are parsed by the BTC LSP
2) This can be delegated to a AVS which would be IBC compatible
3) The AVS streams yield through IBC to Yieldi pools on THORChain where it is swapped to BTC
4) The BTC is held in a yield account until it is ready to be streamed or claimed by the BTC user

## Trust Assumptions

Users stake their LST with the LSP, which is secured by the base-layer protocol (ETH, BTC). When delegated to an AVS, the LSP monitors for double-signing behaviour and slashes. Yieldi does not hold or care about the staked balance, or slashing concerns. 

Only the streaming yield is routed via THORChain and deposited into a Yield Account temporarily. This Yield Account is secured by THORChain's economic guarantees and it's Incentive Pendulum. 
The user can claim or request their yield is streamed to them, reducing the period of time they have to trust the protocol with. 

Yieldi can also be deployed on any cross-chain liquidity protocol which has the following technical requirements:
1) Is IBC compatible
2) Has token-pools
3) Has ETH and BTC layer 1 liquidity available

# Summary

Yieldi - a yield-collecting service for the Eigenlayer Ecosystem is outlined. The following are the discrete components:

1) AVS: Support yield streaming via an Axelar-like General Message Parsing Gateway Contract
2) AVS: Maintain an IBC channel to THORChain
3) AVS: Conduct a Liquidity Auction with incentives to correctly price and build liquidity for the yield token
4) Yieldi: Deploy AVS pools and process inbound yield swaps to native ETH
5) THORChain: Support Yield Accounts and allow users to query balances, claim and set auto-stream

# Technical Readiness

1) Users can deposit LST into Eigenlayer ✅
2) Users can delegate LST to AVS Operator ✅
3) AVS can compute Users share of the yield via `gRPC` ✅
4) AVS can deploy gateway contract to emit IBC-compatible event ✅
5) AVS can send IBC tokens to THORChain ⏳
6) Yieldi can swap IBC tokens to ETH on THORChain ⏳
7) Yieldi can deposit ETH into Yield Account ✅
8) Users can claim accrued ETH in a Yield Account ✅
9) THORChain can auto-stream yield ✅






