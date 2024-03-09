# Operating IBC/Interoperability infrastructure
Operate a shielded expedition-compatible **Osmosis** testnet relayer

### Channel
- **Namada** `channel-624`
- **Osmosis** `channel-6133`

## Set up an Osmosis node
`source <(curl -sL https://get.osmosis.zone/run)` (without cosmovisor)
*Osmosis node on the same server as Namada node*
Source:
- https://get.osmosis.zone/
- https://docs.osmosis.zone/osmosis-core/relaying/relayer-guide/


## Osmosis
### Create wallet for relayer
```
osmosisd keys add relayer_osmosis

- address: osmo1hnad4xdyyv24na559zt2u8svhcvndrenhcrqrj
  name: relayer_osmosis
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"AuVFM25UMwMK/oa193BmPMFl56tG3tmH3lhImxUJ8a6K"}'
  type: local
```

### Request tokens from faucet
https://faucet.testnet.osmosis.zone/
And check balance
```
osmosisd query bank balances osmo1hnad4xdyyv24na559zt2u8svhcvndrenhcrqrj

balances:
- amount: "100000000"
  denom: uosmo
pagination:
  next_key: null
  total: "0"
```

### Configurate node
- In `~/.osmosis/config/app.toml`:
```
[grpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the gRPC server address to bind to.
address = "https://grpc.testnet.osmosis.zone/"
```
- In  `~/.osmosis/config/config.toml` change ports:
```
[rpc]

# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26757"

proxy_app = "tcp://127.0.0.1:26758"

# pprof listen address (https://golang.org/pkg/net/http/pprof)
pprof_laddr = "localhost:6062"

[p2p]

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26756"
```

And start node
`systemctl restart osmosisd`

## Namada
### Create wallet for relayer
```
namadaw gen --alias relayer_namada \
--memo tpknam1qzn899fmms07r5lssg8as02udq65y6skn3e265eyjv2z04ujy0rkzyk393u

namadaw find --alias relayer_namada
Found transparent keys:
  Alias "relayer_namada" (encrypted):
    Public key hash: F97A8D1355E6561C33B3AA67EBC077758739388E
    Public key: tpknam1qz85lqjsqqcehennxtsmn38gkt6mc86gevunwdvsa0wf8jdexkumxy2xlwq
Found transparent address:
  "relayer_namada": Implicit: tnam1qruh4rgn2hn9v8pnkw4x067qwa6cwwfc3cgvw62h
```

### Transfer money from own Namada wallet
```
namadac transfer --source mkhananov_wallet --target relayer_namada --token tnam1qxvg64psvhwumv3mwrrjfcz0h3t3274hwggyzcee --amount 100
```

## Set up Hermes
### Install
```
export TAG="v1.7.4-namada-beta7"
export ARCH="x86_64-unknown-linux-gnu" # or "aarch64-apple-darwin"
curl -Lo /tmp/hermes.tar.gz https://github.com/heliaxdev/hermes/releases/download/${TAG}/hermes-${TAG}-${ARCH}.tar.gz
tar -xvzf /tmp/hermes.tar.gz -C /usr/local/bin
```
Source: https://docs.namada.net/operators/ibc#from-binaries

### Configurate
```
mkdir -p ~/.hermes
cd .hermes
vi config.toml
```

#### config.toml content
```
[[chains]]
id = 'shielded-expedition.88f17d1d14' 
type = 'Namada'
rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:9090'
event_source = { mode = 'push', url = 'ws://127.0.0.1:26657/websocket', batch_delay = '500ms' }
account_prefix = ''
key_name = 'relayer_namada'
store_prefix = 'ibc'
gas_price = { price = 0.0001, denom = 'tnam1qxvg64psvhwumv3mwrrjfcz0h3t3274hwggyzcee' }
rpc_timeout = '30s'

[[chains]]
id = 'osmo-test-5'
type = 'CosmosSdk'
rpc_addr = 'http://127.0.0.1:26757'
grpc_addr = 'https://grpc.testnet.osmosis.zone/'
event_source = { mode = 'push', url = 'ws://127.0.0.1:26757/websocket', batch_delay = '500ms' }
account_prefix = 'osmo'
key_name = 'relayer_osmosis'
address_type = { derivation = 'cosmos' }
store_prefix = 'ibc'
default_gas = 400000
max_gas = 120000000
gas_price = { price = 0.0025, denom = 'uosmo' }
gas_multiplier = 1.2
max_msg_num = 30
max_tx_size = 1800000
clock_drift = '15s'
max_block_time = '30s'
trusting_period = '4days'
trust_threshold = { numerator = '1', denominator = '3' }
rpc_timeout = '30s'
```

### Add keys
Namada from wallet.toml
```
/usr/local/bin/hermes --config $HOME/.hermes/config.toml keys add --chain shielded-expedition.88f17d1d14 --key-file $HOME/.local/share/namada/shielded-expedition.88f17d1d14/wallet.toml
```

Osmosis from file with seed phrase
```
cd ~/.hermes
vi seed ## insert seed phrase in file and save

/usr/local/bin/hermes --config $HOME/.hermes/config.toml keys add --chain osmo-test-5 --mnemonic-file seed
```

## Create channel
```
/usr/local/bin/hermes --config $HOME/.hermes/config.toml \
  create channel \
  --a-chain shielded-expedition.88f17d1d14 \
  --b-chain osmo-test-5 \
  --a-port transfer \
  --b-port transfer \
  --new-client-connection --yes
```

Result
```
SUCCESS Channel {
    ordering: Unordered,
    a_side: ChannelSide {
        chain: BaseChainHandle {
            chain_id: ChainId {
                id: "shielded-expedition.88f17d1d14",
                version: 0,
            },
            runtime_sender: Sender { .. },
        },
        client_id: ClientId(
            "07-tendermint-2166",
        ),
        connection_id: ConnectionId(
            "connection-1029",
        ),
        port_id: PortId(
            "transfer",
        ),
        channel_id: Some(
            ChannelId(
                "channel-624",
            ),
        ),
        version: None,
    },
    b_side: ChannelSide {
        chain: BaseChainHandle {
            chain_id: ChainId {
                id: "osmo-test-5",
                version: 5,
            },
            runtime_sender: Sender { .. },
        },
        client_id: ClientId(
            "07-tendermint-2600",
        ),
        connection_id: ConnectionId(
            "connection-2442",
        ),
        port_id: PortId(
            "transfer",
        ),
        channel_id: Some(
            ChannelId(
                "channel-6133",
            ),
        ),
        version: None,
    },
    connection_delay: 0ns,
}
```

## IBC-tranfer
### Namada -> Osmosis
- Check balances before transfer
```
namadac balance --owner relayer_namada
naan: 96

osmosisd query bank balances osmo1hnad4xdyyv24na559zt2u8svhcvndrenhcrqrj
balances:
- amount: "99989604"
  denom: uosmo
pagination:
  next_key: null
  total: "0"
```

- Transfer NAAN
```
namadac --base-dir $HOME/.local/share/namada \
    ibc-transfer \
    --amount 5 \
    --source relayer_namada \
    --receiver osmo1hnad4xdyyv24na559zt2u8svhcvndrenhcrqrj \
    --token naan \
    --channel-id channel-624 \
    --memo tpknam1qzn899fmms07r5lssg8as02udq65y6skn3e265eyjv2z04ujy0rkzyk393u
```

- Check balances after transfer
```
namadac balance --owner relayer_namada
naan: 88.5

osmosisd query bank balances osmo1hnad4xdyyv24na559zt2u8svhcvndrenhcrqrj
balances:
- amount: "5"
  denom: ibc/277ABAE39F65A09855FA71A06B84D0D386E8D10CC0CA6E23D75237D1365A2D4C
- amount: "99989604"
  denom: uosmo
pagination:
  next_key: null
  total: "0"
```

### Osmosis -> Namada
- Transfer UOSMO
```
osmosisd tx ibc-transfer transfer \
  transfer \
  channel-6133 \
  tnam1qruh4rgn2hn9v8pnkw4x067qwa6cwwfc3cgvw62h \
  1000000uosmo \
  --from relayer_osmosis \
  --gas auto \
  --gas-prices 0.035uosmo \
  --gas-adjustment 1.2 \
  --home "$HOME/.osmosisd" \
  --chain-id osmo-test-5 \
  --yes


gas estimate: 131150
code: 0
codespace: ""
data: ""
events: []
gas_used: "0"
gas_wanted: "0"
height: "0"
info: ""
logs: []
raw_log: '[]'
timestamp: ""
tx: null
txhash: 420E87C5CD6EBEBFC50AE5EB4368D2A4DD84AA5C238EC41B51F10BFA9420705D
```

- Check balances after transfer
```
namadac balance --owner relayer_namada
naan: 88.5
transfer/channel-624/uosmo: 1000000

osmosisd query bank balances osmo1hnad4xdyyv24na559zt2u8svhcvndrenhcrqrj
balances:
- amount: "5"
  denom: ibc/277ABAE39F65A09855FA71A06B84D0D386E8D10CC0CA6E23D75237D1365A2D4C
- amount: "98985013"
  denom: uosmo
pagination:
  next_key: null
  total: "0"
```
