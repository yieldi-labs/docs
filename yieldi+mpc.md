# Overview
Yieldi is an multi-chain MPC protocol with stored routes for yield venues. 
Yieldi manages custody of all assets via MPC EOAs. 
Yieldi nodes bond LP assets for liquidity. 
Incentive pendulum keeps 1:1 security on BOND:POOL

* If BOND=POOLs; then Nodes earn 100% of liquidity fees && yield claimed
* LPs should withdraw (reduce pooled assets)
* Yield owners should withdraw (reduced bridged assets)
* Nodes should bond more (for yield)

Usecases
* A user with BTC wants to deposit in a wrappedBTC yield vault on some other chain, and be streamed back yield.
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



