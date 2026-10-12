# khensu-devnet

A khensu.nu network on one machine for development: a gateway, three mixes and two mailboxes, each a `tatenan` process on a fixed UDP port, every node on the committee that agrees the hourly epoch header. Licensed AGPL-3.0-only. Needs `khensu-core` and `khensu-tatenan` cloned next to this repo and a Rust toolchain.

```sh
./devnet up                   # build, generate keys and leases (first run), sign the genesis header, start, write devnet.json
./devnet status               # processes, and what each node answers to ~node status
./devnet test 30              # loops through the mixes from a client for 30 s, then every node's loops, links and header
./devnet demo                 # alice and bob exchange messages through the devnet with khensu-cli
./devnet node mix-1 header    # any ~node command on one node: status, peers, receipts, measurements, loops, header, lease, log <filter>, drain
./devnet logs mix-1           # JSON logs; no argument follows all nodes
./devnet down                 # SIGTERM; mailboxes and headers are saved
```

| Node | Port (UDP) | Roles |
| --- | --- | --- |
| gateway-1 | 47000 | gateway, mix, committee |
| mix-1 | 47001 | mix, committee |
| mix-2 | 47002 | mix, committee |
| mix-3 | 47003 | mix, committee |
| mailbox-1 | 47004 | mailbox, mix, committee |
| mailbox-2 | 47005 | mailbox, mix, committee |

Six mixes give two per layer, so every route has a choice and a node's own loops avoid itself; the committee of six with a threshold of five means one node can be down and headers still get agreed. Mix layers are dealt per epoch from the leases.

Options for `up`:

- `--host <ip>`: put this address in the leases and listen on all interfaces, so an iPhone on the same network can reach the Mac (default `127.0.0.1`, enough for the simulator).
- `--fresh`: new founder key, node keys, leases, genesis header and empty mailboxes.
- `--foreground`: stay attached; Ctrl-C stops the devnet.

## How it runs

The first `up` makes a founder key (`run/founder/`), then for each node `tatenan keygen` writes `node.key` and a signed `node.lease` covering 24 epochs with mix keys for 168 more (`DEVNET_LEASE_EPOCHS`, `DEVNET_SPARE_EPOCHS`), and `tatenan directory` assembles the genesis header from the six leases, cosigned by the founder, into `devnet.json`. Nodes renew their own leases two epochs before they end, so a running devnet keeps going; a stopped one resumes from each node's saved header (`run/nodes/<name>/data/directory.json`). A devnet brought back after a boundary logs `header.behind` and the committee agrees the missing headers one by one; if it has been down longer than the last header's leases cover, nodes log `lease.ended`, clients cannot build packets, and the fix is `./devnet up --fresh`. Changing `--host` regenerates everything too.

Five minutes before each hour boundary the committee agrees the next header (watch `header.proposed`, `header.voted`, `header.final` in the logs); at the boundary every node issues forwarding receipts to its peers and installs the header; twenty seconds later it signs and submits its reports (`work.signed`, `work.submitted`), and `./devnet node mix-1 receipts` shows them. `./devnet node gateway-1 header` prints the installed header, the committee and the consensus state for the next one.

Logs are JSON lines in `run/logs/<name>.log` (stderr) and rotated daily in `run/nodes/<name>/data/log/`. `./devnet node <name> log tatenan=debug` changes a node's level while it runs.

## devnet.json

Directory v2: the network's rules, the genesis epoch header (base64) and the leases it covers (base64). The app loads it with `KhensuClient.loadDirectory(json:gateway:)` and follows later headers over the link; the format is in `khensu-core/ffi/SWIFT-API.md`.

## khensu-cli

```sh
cli=../khensu-tatenan/target/release/khensu-cli
export KHENSU_DIRECTORY=$PWD/devnet.json
$cli net-test --secs 30 --loops 100     # loop probes: sent, returned, rtt median/p90/max, headers accepted
$cli -s alice.state init --name alice
$cli -s bob.state init --name bob
$cli -s bob.state card --out bob.card
conv=$($cli -s alice.state chat bob.card)
$cli -s alice.state send "$conv" "hello"
$cli -s alice.state run --until-sent &
$cli -s bob.state run --messages 1 --loops 10
```

`run` also reads commands on stdin: `~send <conversation> <text>`, `~chat <card file>`, `~status`, `~loops <per 100 ticks>`, `~quit`. `--loops` sets the loop rate from the start. Client state is an AES-256-GCM blob with its key in `<state>.key` next to it, which is fine for a devnet and nothing else. Use one process per state file.
