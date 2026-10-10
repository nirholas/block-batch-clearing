# BlockBatchClearing

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/block-batch-clearing?style=flat&logo=github)](https://github.com/nirholas/block-batch-clearing/stargazers) [![License](https://img.shields.io/github/license/nirholas/block-batch-clearing?style=flat)](https://github.com/nirholas/block-batch-clearing/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/block-batch-clearing?style=flat)](https://github.com/nirholas/block-batch-clearing/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/block-batch-clearing/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/block-batch-clearing/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Every swap in a block clears at one price, and orders that want opposite sides fill against each other for free.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://block-batch-clearing.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/BlockBatchClearingHook.sol`](src/hooks/BlockBatchClearingHook.sol)
- **Licence:** Apache-2.0

## How it works

Almost all of what is called MEV is the value of being first. A continuous market prices each order at the state the one before it left behind, so the right to arrive earlier is worth money, and that right is sold. Batch auctions were proposed for exactly this in the equities literature, and the answer they give is not to police the ordering but to abolish it: collect every order in an interval and clear them all at one uniform price.

Being first in a batch is worth nothing, because there is no first. CoW Protocol runs that design with a network of solvers off-chain. Doing it on-chain has needed a solver too, since a uniform price for a batch against a bonded curve looks like a fixed point: the price depends on the net trade, and the net trade depends on the price.

For a constant-product pool it is not a fixed point. Requiring that both sides clear at one price `p` and that the invariant hold gives a quadratic in `p` with an exact solution, and this hook solves it in closed form. There is no solver, no auction, no bond and nobody to trust with the ordering.

What falls out of it is the part traders will notice. Orders in opposite directions settle against each other before the curve is touched at all, so matched flow pays no slippage whatsoever; only the imbalance moves the price. A block where buyers and sellers are evenly matched clears at the spot price for everybody, which is a thing a continuous AMM cannot do at any size.

The cost is that a swap does not return tokens. It returns a claim on a batch that has not cleared yet, and the trader collects afterwards. That is the honest price of not having an ordering to sell.

## Prior art

Frequent batch auctions are Budish, Cramton and Shim's answer to the equities latency race. CoW Protocol clears uniform-price batches with off-chain solvers, and Gnosis ran an on-chain batch exchange over an order book. On v4, TWAMM spreads one order over time and various hooks reorder or tax within a block. Clearing a whole block's flow against a bonded curve at one uniform price, solved in closed form on-chain with no solver and no auction, is the contribution here.

## Where it does not help

A swap returns nothing at the moment it is made, so this cannot be routed through by an aggregator expecting tokens back and is unusable for anything atomic. Orders are exact-input only, for the same reason an exact output cannot be promised before the price is known. A batch also clears when somebody touches the pool in a later block, so the last batch of a quiet period waits for the next swap or for anybody to call `clear`; the trader's funds are held meanwhile. And the uniform price is uniform within a block, which means the ordering advantage is gone but the choice of which block to be in is not.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AddLiquidityThroughHook()` | Liquidity belongs to the hook, not to the pool, so the pool's own path is closed. |
| `AlreadyBound()` | This hook serves one pool, bound the first time one initializes with it. |
| `AmountTooSmall()` | The deposit was too small to mint a share, or the withdrawal too small to return anything. |
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExactOutputUnsupported()` | A batch cannot promise an exact output before it knows the price everybody clears at. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InvalidFee()` | A fee at or above the whole trade is not a fee. |
| `NoReserves()` | The pool holds nothing, so there is nothing to clear against. |
| `NotCleared()` | The batch is still open, so there is no price yet. |
| `NothingToCollect()` | The order does not exist, or has already been collected. |
| `SafeCastOverflowedIntDowncast(uint8,int256)` | Value doesn't fit in an int of `bits` size. |
| `SafeCastOverflowedUintDowncast(uint8,uint256)` | Value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintToInt(uint256)` | A uint value doesn't fit in an int of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # BlockBatchClearing
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, batch-auction, uniform-price, order-flow, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/block-batch-clearing
cd block-batch-clearing
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/block-batch-clearing&type=Date)](https://www.star-history.com/#nirholas/block-batch-clearing&Date)

<!-- three.ws:growth -->
## Support the project

If block-batch-clearing saves you time, **[star it on GitHub](https://github.com/nirholas/block-batch-clearing)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=block-batch-clearing%3A%20Every%20swap%20in%20a%20block%20clears%20at%20one%20price%2C%20and%20orders%20that%20want%20opposite%20sides%20fill%20against%20each&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fblock-batch-clearing) · [Share on Bluesky](https://bsky.app/intent/compose?text=block-batch-clearing%3A%20Every%20swap%20in%20a%20block%20clears%20at%20one%20price%2C%20and%20orders%20that%20want%20opposite%20sides%20fill%20against%20each%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fblock-batch-clearing) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fblock-batch-clearing) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fblock-batch-clearing&t=block-batch-clearing%3A%20Every%20swap%20in%20a%20block%20clears%20at%20one%20price%2C%20and%20orders%20that%20want%20opposite%20sides%20fill%20against%20each) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fblock-batch-clearing&title=block-batch-clearing%3A%20Every%20swap%20in%20a%20block%20clears%20at%20one%20price%2C%20and%20orders%20that%20want%20opposite%20sides%20fill%20against%20each)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/block-batch-clearing` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/block-batch-clearing/issues) or [start a discussion](https://github.com/nirholas/block-batch-clearing/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/block-batch-clearing)](https://github.com/nirholas/block-batch-clearing/graphs/contributors)

<!-- /three.ws:growth -->
