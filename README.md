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

## Running against eCash alphanet (slot 4)

There is no alphanet network preset — `--network` accepts only
`signet | regtest | forknet`, and the default `signet` is what to use. The
mainchain side comes entirely from whichever enforcer you point at:

```
plain_bitassets_app \
    --headless \
    --mainchain-grpc-host 127.0.0.1 \
    --mainchain-grpc-port 50051
```

P2P is UDP 4004, RPC is 6004. The two seed nodes compiled in are signet
nodes; they will connect and then fail with
`Error fetching mainchain ancestors`, which is harmless — they are on a
different mainchain. Add peers by hand:

```
plain_bitassets_app_cli --rpc-port 6004 connect-peer <host>:4004
```
