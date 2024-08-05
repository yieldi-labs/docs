# Technical Architecture
📜 Technical Architecture of YIELDI

<img width="698" alt="image" src="https://github.com/user-attachments/assets/c331ddbe-ce54-4ca7-b8de-20e960abdd5f">


# Overview

This document serves as a comprehensive guide to understanding the underlying design and structure of YIELDI, with a clear and detailed description of the various components, systems, and technologies that make up the solution.

## Terms:

- `AVS` Active Validator Set for the app-chain
- `IBC` Inter-Blockchain Communication protocol
- `LST` Liquid Staking Token
- `LSP` Liquid Staking Protocol

## What is YIELDI?

YIELDI offers a gas-efficient yield-collecting solution for the [Eigenlayer](https://eigenlayer.xyz/) ecosystem, and it will initially be deployed on [Thorchain](https://thorchain.org/) which has native ETH liquidity. Users can re-stake ETH with Eigenlayer and delegate their security to an app-chain (Active Validator Set or AVS). The AVS can stream yield back to the staker as native ETH using YIELDI, where stakers can view their accumulated yield, claim it, or even have it auto-streamed to their address on a gas-efficient interval. 

YIELDI also solves for price-discovery and liquidity of AVS tokens, and will increase the propensity of users to delegate LST to AVS operators if they are paid real-yield in native assets (ETH). Lastly it ensures that the full cycle of yield collection from AVS is conducted with minimal trust assumptions and third-party dependencies. 

## How does it work?

Stakers deposit into Eigenlayer's contracts. The AVS can read and compute the user's share of the yield in points, tokens or fees, then send it via IBC to THORChain on some interval. THORChain is able to swap to native ETH and hold in yield collector module with the owner set to the staker's L1 address. The Staker can then view their yield and claim it any time. Additionally, the Staker may opt-in for auto-streaming, which causes THORChain to stream out the entire balance on a gas-efficient interval. 

## What problem does it solve?

Stakers are much more likely to delegate their LSTs to AVS operators who can pay "real yield" which is realised in native ETH instead of an illiquid rewards token that does not yet have liquidity or price discovery. YIELDI solves the routing and collection of yield tokens, as well as ensuring they have liquidity against ETH. ETH stakers will simply see their rewards accrued in a native ETH balance which they can claim anytime. They will be able to compute their annualised yields and make informed decisions about their capital. 

# Diagrams

The following diagrams for each process provide a visual representation of the asset flow. 

## Eigenlayer: Restaking

<img width="699" alt="image" src="https://github.com/user-attachments/assets/fd577da5-3862-4b40-8b5e-7f240aaf04fa">

## AVS: Yield-streaming

<img width="698" alt="image" src="https://github.com/user-attachments/assets/098b95ac-d7cb-40d6-8a8b-561b978ed612">

## YIELDI: Yield-collecting

<img width="697" alt="image" src="https://github.com/user-attachments/assets/76f8348e-5670-437b-9232-43566dee2248">

## User: Yield-claiming

<img width="697" alt="image" src="https://github.com/user-attachments/assets/b36be643-dbda-4ef3-9e77-50d55a111c49">

# Implementation

## Eigenlayer Contracts

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

## AVS Reference Implementation

The AVS deposits the yield as an ERC-20 token. The AVS is then able to compute the share of the yields to pay to each user and send it.  

```js
contract YieldManager {
    uint256 public accumulatedYield; // Temporary balance of yield (points, tokens, fees)

    function depositYield(uint256 amount) public; // Deposit yield for accumulation

    function sendYield(address user, uint256 amount) onlyOperator; //AVS sends yield per user

    function batchSendYield(address[] memory users, uint256[] memory amounts) onlyOperator; //AVS batch sends yield
}
```

## EVM to IBC General Message Parsing

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

### IBC chains. 

AVS which are natively IBC compatible simply need to emite the IBC packet and skip the EVM Gateway contract.

## YIELDI WASM Contracts

### IBC Swap to TOR
YIELDI receives the IBC payload and executes a [Swap](https://github.com/Team-Kujira/kujira-rs/blob/master/packages/kujira-fin/src/execute.rs) to native ETH. 

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
The callback would then stream to ETH to the user's ETH Yield Account: 

```md
MsgDeposit{
       yield:eth~eth:ethAddress
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

The user can Claim their yield at anytime by interating with [THORChain's Router](https://gitlab.com/thorchain/thornode/-/blob/develop/chain/ethereum/contracts/THORChain_Router.sol)
```md
function depositWithExpiry(address payable vault, address asset, uint amount, string memory memo, uint expiration) public

MEMO: CLAIM // Claim all the yield
MEMO: CLAIM:5000 // Claim 50% of the yield
```

### Auto-stream

THORChain has a [Collector Module](https://gitlab.com/thorchain/thornode/-/merge_requests/2978) that holds assets until they reach a value that is 100x the `outboundGasCost` for that chain.
On a regular interval, it will auto-send the full balance to the user, at which it will execute at a minimum of 99% (1% would be the max gas cost). This efficiently sends native yield directly back to the user. 

The user has an option to specify their stream interval when interacting with THORChain:

```
MEMO: CLAIM::100 // Claim all the yield and set auto-stream to 100x the gas cost (99% execution)
MEMO: CLAIM::1000 // Claim all the yield and set auto-stream 1000x the gas cost (99.9% execution)
MEMO: CLAIM::0 // Claim all the yield and set auto-stream off
```

# Economics

AVS need two further things to ensure their asset is correctly priced. 

1) Initial liquidity in the `TOR:AVS` token pool in YIELDI
2) Ongoing liquidity incentives to ensure sufficient liquidity for users

Without adequate liquidity and an existing price, the user is not able to correctly determine the risk-reward for delegating to a particular operator. 
Over time, the operators with the most liquidity, stable price and economic activity on their chains would pay their highest risk-reward returns for their users. 

## One-time Liquidity Auction

> AVS operators need a liquidity venue for their assets with an established price

A liquidity auction can be conducted by the AVS when setting up the channel. 

1) Mint 5-10% of the supply into YIELDI as a CW-20 token
2) Offer this in the new `TOR:AVS` token pool
3) Over 7-30 days, anyone can deposit `TOR` to match the `AVS` token and infer the launch price of the asset.
4) When the pool goes live, the AVS will own 50% of the pool, and Liquidity Auction participants will own the other 50%. 

## Liquidity Mining

> AVS should incentivise sufficient liquidity to stablise price and ensure best execution

The AVS should continually stream yield incentives to the `TOR:AVS` pool as it will be the primary liquidity venue for the AVS token. 

1) The AVS can stream token incentives through the IBC channel with destination the `TOR:AVS` pool.
2) The incentives are added into the pool which are credited to the pool LPs.

# Summary

YIELDI - a yield-collecting service for the Eigenlayer Ecosystem is outlined. The following are the discrete components:

1) AVS: Support yield streaming via an Axelar-like General Message Parsing Gateway Contract
2) AVS: Maintain an IBC channel to THORChain
3) AVS: Conduct a Liquidity Auction with incentives to correctly price and build liquidity for the yield token
4) YIELDI: Deploy AVS pools and process inbound yield swaps to native ETH
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






