# BitAssets

> **This is a patched fork**, tracking
> [LayerTwo-Labs/plain-bitassets](https://github.com/LayerTwo-Labs/plain-bitassets)
> at `eddfe8f` (v0.16.3). It carries four fixes:
>
> 1. **Deposits are credited to the right address.** The enforcer hands the
>    node a deposit address in the prefixed `s4_<address>_<checksum>` form,
>    and upstream calls `Address::from_str` on the whole string. Base58
>    decoding fails, and the deposit silently credits `11111111111111111111`
>    — unspendable. Ported from
>    [thunder-rust#142](https://github.com/LayerTwo-Labs/thunder-rust/pull/142).
> 2. **A `balance` RPC alias** for `bitcoin_balance`, which is the name
>    BitWindow's orchestrator calls. Without it the GUI's balance card spins
>    forever on a perfectly healthy chain.
> 3. **The first mint for a pair creates the pool.** The state layer already
>    creates a pool when the pair has none, but the `amm_mint` RPC and the
>    GUI's dex mint both read the pool state first through a call that errors
>    when it is missing, so *no pool could be created through any interface*.
> 4. **The base coin may be a pool side.** An AMM mint had to spend two
>    distinct BitAssets and a burn had to produce two, which made an
>    ECX/BitAsset pool impossible to create or unwind even though the rest of
>    the AMM handles it. Now the requirement counts only the sides of the pool
>    that actually are BitAssets.
>
> **Every node on a chain must run the same side of these fixes.** Fix 1
> changes which address a deposit credits, and fix 4 is a consensus change —
> this node accepts blocks an unpatched node rejects. A patched and an
> unpatched node will diverge the moment anyone deposits or seeds a pool
> against the base coin. Do not mix them on eCash alphanet slot 4.
>
> [Compare against upstream](https://github.com/LayerTwo-Labs/plain-bitassets/compare/master...Coinelius:plain-bitassets:master)

## Install

Check out the repo with `git clone`, and then

```
git submodule update --init
cargo build
```

Requires the nightly toolchain (see `rust-toolchain.toml`).

## Running on eCash alphanet (slot 4)

### 1. Get an L1 node and enforcer

A BitAssets node is useless without an eCash alphanet `bitcoind` and a
BIP300/301 enforcer to talk to.

The easy path is **BitWindow in Full mode** — it downloads and runs both for
you (`bitcoind` and `bip300301-enforcer` are in its `chains_config.json`
alongside `bitassets`) and syncs them. Be aware what that sync is: alphanet is
a fork of Bitcoin mainnet at height 963,648 and carries all the real history
before it, so expect a long initial sync and on the order of **800 GB** of
block data.

Running your own instead: the enforcer needs `--network-preset=alphanet` —
this is not optional, since without it the enforcer applies BIP300 rules to
963k blocks of pre-fork history and will fork your node — and
`--enable-block-template-server`.

### 2. Replace the binary BitWindow downloads

**Do this or the rest does not matter.** BitWindow fetches *upstream*
BitAssets into `~/.local/share/bitwindow/assets/bin/bitassets` and runs that,
not this fork. On upstream, every deposit is credited to
`11111111111111111111` and is unspendable, and there is no `balance` RPC, so
the GUI's balance card spins forever on a perfectly healthy chain. After
building:

```
cp target/release/plain_bitassets_app     ~/.local/share/bitwindow/assets/bin/bitassets
cp target/release/plain_bitassets_app_cli ~/.local/share/bitwindow/assets/bin/bitassets-cli
```

BitWindow also hardcodes the eCash RPC port to 18302, so bridge that to your
node's real RPC port if it differs.

### 3. Run the node

There is no alphanet network preset — `--network` accepts only
`signet | regtest | forknet`, and the default `signet` is what to use. The
mainchain side comes entirely from whichever enforcer you point at:

```
plain_bitassets_app \
    --headless \
    --mainchain-grpc-host 127.0.0.1 \
    --mainchain-grpc-port 50051
```

P2P is UDP 4004, RPC is 6004.

### 4. Find a peer

The two seed nodes compiled in are *signet* nodes, on a different mainchain.
They connect and then fail with `Error fetching mainchain ancestors`, which is
harmless and can be ignored. There is no alphanet seed, so add peers by hand:

```
plain_bitassets_app_cli --rpc-port 6004 connect-peer <host>:4004
```

Whoever you peer with must run the same side of the deposit-address fix above,
and needs UDP 4004 reachable.

## Operating notes

These are not obvious and each one costs hours to rediscover.

**Always pass `--timeout 600` or more.** The CLI defaults to 60 seconds, while
`mine` waits for the next mainchain block — averaging around 27 minutes on
alphanet — and `create-deposit` can wait on the enforcer wallet. Worse, a
timed-out call is *not* cancelled: the node-side task keeps holding
`miner.write()`, which silently blocks every later `mine` and `create-deposit`
until the service is restarted.

**A BMM bid is valid for exactly one mainchain block.** An M8 commits to
`prev_mainchain_block_hash`, so it can only be included in the block that
immediately follows it. Miss that block and the bid is permanently unminable
and simply sits in the mempool. Bidders must therefore re-bid once per
mainchain block.

**A dead bid strands what comes after it.** The enforcer wallet chains each
new transaction onto the previous one's change, so a deposit built after a
dead bid inherits it as an unconfirmed ancestor and can never confirm, no
matter what fee it pays. If a deposit will not confirm, check its ancestry
before assuming a fee problem.

**Sidechain "header height 0" is normal.** The orchestrator reports a
sidechain's sync from a single `getblockcount` probe and never populates a
header count, so every sidechain shows headers as 0.

**A BitAsset has no on-chain name.** The BitAssetId *is* `blake3(name)`, and
registration reveals that hash, not the string — that is the point of the
commit/reveal. `BitAssetData` has no name, ticker, or description field
either; it carries only `commitment`, `socket_addr_v4`, `socket_addr_v6`,
`encryption_pubkey` and `signing_pubkey`. So every client can only show you a
hash. Names resolve locally, by hashing a string you already know.

**There is no BitAsset balance RPC.** `bitcoin_balance` (and the `balance`
alias) report sats only. BitAsset and LP token holdings exist purely as
UTXOs, so a client has to sum `my_utxos` itself. Nothing displays them today.
