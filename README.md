# BitAssets

> **This is a patched fork**, tracking [LayerTwo-Labs/plain-bitassets](https://github.com/LayerTwo-Labs/plain-bitassets)
> at `eddfe8f` (v0.16.3). It carries one fix, ported from
> [thunder-rust#142](https://github.com/LayerTwo-Labs/thunder-rust/pull/142):
> the node can now read the `s4_<address>_<checksum>` deposit address the
> enforcer hands it. Upstream calls `Address::from_str` on that whole string,
> base58 decoding fails, and the deposit silently credits
> `11111111111111111111` — unspendable.
>
> **Every node on a chain must run the same side of this fix.** It changes
> which address a deposit credits, so a patched and an unpatched node will
> diverge the moment anyone deposits. Do not mix them on eCash alphanet
> slot 4.
>
> [Compare against upstream](https://github.com/LayerTwo-Labs/plain-bitassets/compare/master...Coinelius:plain-bitassets:master)
> — the diff is two files.

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
