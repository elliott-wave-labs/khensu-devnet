# khensu-devnet

A khensu.nu network on one machine for development: a gateway, three mixes (one per layer) and a mailbox, each a `tatenan` process on a fixed UDP port. Licensed AGPL-3.0-only. Needs `khensu-core` and `khensu-tatenan` cloned next to this repo and a Rust toolchain.

```sh
./devnet up                 # build, generate keys (first run), start, write devnet.json
./devnet demo               # alice and bob exchange messages through the devnet with khensu-cli
./devnet status
./devnet logs mix-1         # JSON logs; no argument follows all nodes
./devnet down               # SIGTERM; the mailbox is saved
```

| Node | Port (UDP) | Role |
| --- | --- | --- |
| gateway-1 | 47000 | gateway |
| mix-1 | 47001 | mix, layer 1 |
| mix-2 | 47002 | mix, layer 2 |
| mix-3 | 47003 | mix, layer 3 |
| mailbox-1 | 47004 | mailbox |

Options for `up`:

- `--host <ip>`: put this address in the directory and listen on all interfaces, so an iPhone on the same network can reach the Mac (default `127.0.0.1`, enough for the simulator).
- `--fresh`: new keys and empty mailbox.
- `--foreground`: stay attached; Ctrl-C stops the devnet.

Keys are generated once into `run/nodes/<name>/` with one mix key per epoch for a week (`DEVNET_EPOCHS` changes that). After that the nodes warn and clients can no longer build packets: run `./devnet down && ./devnet up --fresh`. Changing `--host` regenerates the keys too.

## devnet.json

`devnet.json` lists every node's endpoint, ML-DSA-87 key, ML-KEM-1024 link key and per-epoch mix keys, plus the network parameters (2 s ticks, 4 small packets per tick, 250 ms mean mix delay). The app loads it with `KhensuClient.loadDirectory(json:gateway:)`; the format is in `khensu-core/ffi/SWIFT-API.md`.

## khensu-cli

```sh
cli=../khensu-tatenan/target/release/khensu-cli
export KHENSU_DIRECTORY=$PWD/devnet.json
$cli -s alice.state init --name alice
$cli -s bob.state init --name bob
$cli -s bob.state card --out bob.card
conv=$($cli -s alice.state chat bob.card)
$cli -s alice.state send "$conv" "hello"
$cli -s alice.state run --until-sent &
$cli -s bob.state run --messages 1
```

`run` also reads commands on stdin: `send <conversation> <text>`, `chat <card file>`, `quit`. Client state is an AES-256-GCM blob with its key in `<state>.key` next to it, which is fine for a devnet and nothing else. Use one process per state file.
