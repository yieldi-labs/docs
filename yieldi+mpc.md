# Overview
Yieldi is an multi-chain MPC protocol with stored routes for yield venues. 
Yieldi manages custody of all assets via MPC EOAs. 
Yieldi nodes bond LP assets for liquidity. 
Incentive pendulum keeps 1:1 security on BOND:POOL

* If BOND=POOLs; then Nodes earn 100% of liquidity fees && yield claimed
* LPs should withdraw (reduce pooled assets)
* Yield owners should withdraw (reduced bridged assets)
* Nodes should bond more (for yield)

User Stories
* A user with BTC wants to deposit in a wrappedBTC yield vault on some other chain, and be streamed back BTC yield.
* A user with ETH wants to stake, and delegate security to an AVS to earn yield, streamed back in USDC
* A user with an ETH address wants to have a TON memecoin balance

Routes
* A searcher can set up a route, managed by Yieldi
  1) ETH -> staked
  2) Yield -> sold to USDC
  3) USDC -> streamed back to User
 
ETH Example
* Deposit ETH in Yieldi, call the yield route
* Yieldi swaps to STETH
* STETH staked in Eigen
* Yield earnt via points/EIGEN
* Yieldi claims yield, sells via {AMM}
* USDC held in Yieldi
* Yieldi streams USDC to user

BTC Example
* Lock BTC in Yieldi
* Delegate to AVS, earn yield
* Yield sold to BTC
* BTC streamed back to user

# Liquidity
Liquidity is provided by external assets matched to an internal, arbitrary unit of account - `Y`. 
`Y` is minted only to start the initial locked liquidity for new chains and pools. 
New liquidity providers have to stream-add and stream-remove at a small rate to stop slippage - 3-5BPS.  

# Swaps
All swaps are streaming - streamed from one asset pool to another via `Y`. They target a low slippage - 5-10BPS. 

# Remote Balances
Any EOA can hold a yield position in any aggregated vault, but the yield position is managed by Yieldi MPC. 
EOAs submit signed messages from their address to manage remote balances and positions - not L1 transactions. 

# Chains
Chains are added rapidly using just RPC methods. MPC EOAs manage each vault. 

# Memoless ERC20 deposits. 
ERC20's can be deposited to the system from any address. The user sends a signed message instructing the state machine how to manage it (before or after the deposit). 





