# Technical Architecture
📜 Technical Architecture of Hodly

<img width="918" alt="image" src="https://github.com/user-attachments/assets/634d7f15-c7dd-4617-8558-b510d64f756c">


# Overview

This document serves as a comprehensive guide to understanding the underlying design and structure of Hodly, with a clear and detailed description of the various components, systems, and technologies that make up the solution.

## What is Hodly?

Hodly offers a gas-efficient yield-collecting solution for the [Eigenlayer](https://eigenlayer.xyz/) ecosystem, and it will initially be deployed on [Thorchain](https://thorchain.org/) which has native ETH liquidity. Users can re-stake ETH with Eigenlayer and delegate their security to an app-chain (Active Validator Set or AVS). The AVS can stream yield back to the staker as native ETH using Hodly, where stakers can view their accumulated yield, claim it, or even have it auto-streamed to their address on a gas-efficient interval. 

Hodly also solves for price-discovery and liquidity of AVS tokens, and will increase the propensity of users to delegate LST to AVS operators if they are paid real-yield in native assets (ETH). Lastly it ensures that the full cycle of yield collection from AVS is conducted with minimal trust assumptions and third-party dependencies. 

## How does it work?

Stakers deposit into Eigenlayer's contracts. The AVS can read and compute the user's share of the yield in points, tokens or fees, then send it via IBC to THORChain on some interval. THORChain is able to swap to native ETH and hold in yield collector module with the owner set to the staker's L1 address. The Staker can then view their yield and claim it any time. Additionally, the Staker may opt-in for auto-streaming, which causes THORChain to stream out the entire balance on a gas-efficient interval. 

## What problem does it solve?

ETH stakers are much more likely to delegate their LSTs to AVS operators who can pay "real yield" which is realised in native ETH instead of an illiquid rewards token that does not yet have liquidity or price discovery. Hodly solves the routing and collection of yield tokens, as well as ensuring they have liquidity against ETH. ETH stakers will simply see their rewards accrued in a native ETH balance which they can claim anytime. They will be able to compute their annualised yields and make informed decisions about their capital. 

## Terms:

- `AVS` Active Validator Set for the app-chain
- `IBC` Inter-Blockchain Communication protocol
- `LST` Liquid Staking Token
- `LSP` Liquid Staking Protocol

# Diagrams

The following diagrams for each process provide a visual representation of the asset flow. 

## Eigenlayer: Restaking

Diagram of Eigenlyer Restaking Flow

<img width="760" alt="image" src="https://github.com/user-attachments/assets/af839f2c-866f-4324-a810-c20315589acb">


## AVS: Yield-streaming

Diagram of AVS Yield Streaming

<img width="758" alt="image" src="https://github.com/user-attachments/assets/23e89e42-2652-40c7-8b05-b979309bbdcf">


## Hodly: Yield-collecting

Diagram of Hodly Yield Collection

<img width="698" alt="image" src="https://github.com/user-attachments/assets/c5feef01-28d0-4490-b43f-360194d0e308">



## User: Yield-claiming

Diagram of User Yield Claiming

<img width="754" alt="image" src="https://github.com/user-attachments/assets/01d09e39-50d6-4471-b682-3450cef3f969">



# Implementation

## Eigenlayer Contracts

When the user deposits their ETH and delegates it to an AVS operator in a token pool, their address and amount is stored in the [Eigenlayer contract](https://www.blog.eigenlayer.xyz/ycie/): 

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

## Axelar EVM to IBC General Message Parsing

The AVS must have a deployed [Axelar Gateway contract](https://docs.axelar.dev/dev/cosmos-gmp) or similiar. 

The AVS calls the Gateway contract with the yield tokens, which then forwards it via Axelar to THORChain. 

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

### Optional: AVS Runs the Relayer. 

The AVS can run an EVM<>IBC Relayer to skip Axelar and directly interact with THORChain for simplicity. 

The AVS gateway contract would hold the yield tokens forever, and the IBC Channel would mint them to forward to THORChain. 

## Hodly WASM Contracts

### IBC Swap to TOR
Hodly receives the IBC payload and executes a [Swap](https://github.com/Team-Kujira/kujira-rs/blob/master/packages/kujira-fin/src/execute.rs) to THORChain's TOR stablecoin. 

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

### L1 swap to ETH Yield Account
The callback would then redeem the TOR into L1 RUNE, then stream to ETH to the user's ETH Yield Account: 

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
MEMO: CLAIM:10000:100 // Claim all the yield and set auto-stream to 100x the gas cost
MEMO: CLAIM::0 // Claim all the yield and set auto-stream off
```

### Auto-stream

THORChain has a [Collector Module](https://gitlab.com/thorchain/thornode/-/merge_requests/2978) that holds assets until they reach a value that is 100x the outboundGasCost for that chain.
At this point, it will auto-send the full balance to the user, at which it will execute at a minimum of 99% (1% would be the max gas cost). This efficiently sends native yield directly back to the user. 

# Economics

AVS need two further things to ensure their asset is correctly priced. 

1) Initial liquidity in the `TOR:AVS` token pool in Hodly
2) Ongoing liquidity incentives to ensure sufficient liquidity for users

## One-time Liquidity Auction

A liquidity auction can be conducted by the AVS when setting up the channel. 

1) Mint 5-10% of the supply into Hodly as a CW-20 token
2) Offer this in the new `TOR:AVS` token pool
3) Over 7-30 days, anyone can deposit `TOR` to match the `AVS` token and infer the launch price of the asset.
4) When the pool goes live, the AVS will own 50% of the pool, and Liquidity Auction participants will own the other 50%. 

## Liquidity Mining

The AVS should continually stream yield incentives to the `TOR:AVS` pool as it will be the primary liquidity venue for the AVS token. 

1) The AVS can stream token incentives through the IBC channel with destination the `TOR:AVS` pool.
2) The incentives are added into the pool which are credited to the pool LPs.

# Summary

Hodly - a yield-collecting service for the Eigenlayer Ecosystem is outlined. The following are the discrete components:

1) AVS: Support yield streaming via an Axelar-like General Message Parsing Gateway Contract
2) AVS: Maintain an IBC channel to THORChain
3) AVS: Conduct a Liquidity Auction with incentives to correctly price and build liquidity for the yield token
4) Hodly: Deploy AVS pools and process inbound yield swaps to native ETH
5) THORChain: Support Yield Accounts and allow users to query balances, claim and set auto-stream









