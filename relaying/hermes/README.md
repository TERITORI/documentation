# Hermes Relayer Tutorial  

In this tutorial you will learn how to setup and IBC relayer using [Hermes](https://hermes.informal.systems/)  

Update and install few packages:  
```shell
apt update && apt upgrade -y && apt install librust-openssl-dev build-essential git -y
```  

Install [rust](https://www.rust-lang.org/tools/install)  
```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```  

## Hermes Installation

```shell
git clone https://github.com/informalsystems/ibc-rs.git hermes && cd hermes && git checkout v1.0.0-rc.0 && cargo build --release
```  


```shell
cp target/release/hermes /usr/bin
```  

## Hermes Configuration  

Make hermes config directory:  

```shell
mkdir -p $HOME/.hermes
```

We will now create default hermes configuration. Here are all the parameters you have to take in account and change before pasting it to your machine:  
- For `rpc_addr`, `websocket_addr` and `grpc_addr`:
  - __ip address__ 
  - __port__ 
- For all chains `channel-X`:
  - replace X by the channel-id associated to the chains you want to relay for [here](https://github.com/TERITORI/teritori-chain/tree/main/relaying/ibc-channels)  
  - if the ibc channel doesn't exist yet, feel free to discuss it on our [Discord](https://discord.gg/teritori) before creating it
- For `memo_prefix`:
  - change the `<Service provider>` field with your validator name

__Copy__ and __modify__ the below file before pasting it on your machine:
```shell
cat <<EOF > /$HOME/.hermes/config.toml
[global]
log_level = 'debug'

[mode]

[mode.clients]
enabled = false
refresh = false
misbehaviour = false

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
#filter = enabled
tx_confirmation = false

[rest]
enabled = true
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = false
host = '127.0.0.1'
port = 3001

[[chains]]
id = 'teritori-1'
rpc_addr = 'http://localhost:26657'
grpc_addr = 'http://localhost:9090'
websocket_addr = 'ws://localhost:26657/websocket'
rpc_timeout = '30s'
account_prefix = 'teritori'
key_name = 'relayer'
store_prefix = 'ibc'
max_tx_size = 180000
max_gas = 2000000
gas_price = { price = 0.0025, denom = 'utori' }
gas_adjustment = 0.1
clock_drift = '15s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '<Service provider> IBC service'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-X']
]

[[chains]]
id = 'osmosis-1'
rpc_addr = 'http://localhost:26657'
websocket_addr = 'ws://localhost:26657/websocket'
grpc_addr = 'http://localhost:9090'
rpc_timeout = '30s'
account_prefix = 'osmo'
key_name = 'relayer'
store_prefix = 'ibc'
max_gas = 15000000
max_msg_num = 10
gas_price = { price = 0.0001, denom = 'uosmo' }
gas_adjustment = 1
clock_drift = '15s'
trusting_period = '9days'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '<Service provider> IBC service'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-X']
]

EOF

```
You can now validate your hermes configuration file:  
```shell
hermes config validate
```  

## Add your relaying-wallets  

We recommand you to create a new key and never use it for anything else than relaying
```shell
hermes keys add teritori-1 -m "12 or 24 magic words"
hermes keys add osmosis-1 -m "12 or 24 magic words"
```  

## Create daemon service file  

```shell
tee /etc/systemd/system/hermes.service > /dev/null <<EOF
[Unit]
  Description=Hermes relayer daemon
  After=network-online.target
[Service]
  User=$USER
  ExecStart=/usr/bin/hermes start
  Restart=on-failure
  RestartSec=3
  LimitNOFILE=4096
[Install]
  WantedBy=multi-user.target
EOF
```  

## Start hermes service 

```shell
systemctl enable hermes && systemctl daemon-reload
```  
You can now start the service and look at the logs:
```shell
systemctl restart hermes && journalctl -u hermes.service -f
```  
