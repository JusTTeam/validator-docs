# Configuring Your Node For Kava Testnet 14000

How to submit gentx and setup validator node you can find [here](https://github.com/polkachu/cosmos-validators/blob/main/docs/kava/testnet-14000.md).  
Original validator guide you can find [here](https://github.com/Kava-Labs/kava/blob/master/docs/validator_guide.md).

## Configuring Your Node
### Download the genesis file
```bash
wget https://raw.githubusercontent.com/Kava-Labs/kava-testnets/master/14000/genesis.json -O ~/.kava/config/genesis.json
```
### Verify genesis hash
```bash
cd ~/.kava/config/
shasum -a 256 genesis.json

# Output
ccce0d62cb57c0e3e39ea857fcd78473ec77a7587627c027288d479412518f43
```

Next, adjust some configurations. To open the config file:
```bash
vim $HOME/.kava/config/config.toml
```
At line 188, add some persistent peers, which help maintain a connection to the peer-to-peer network
```bash
c393dae6f1fb31824a8be066c03cb33434943505@107.21.78.161:26656
```
Next, chose how much historical state you want to store. To open the application config file:
```bash
vim $HOME/.kava/config/app.toml
```
In this file, choose between `default`, `nothing`, and `everything`. To reduce hard drive storage, choose `everything` or `default`. To run an archival node, chose `nothing`.
```bash
pruning = "default"
```
In the same file, you will want to set minimum gas prices — setting a minimum prevents spam transactions:
```bash
minimum-gas-prices = "0.001ukava"
```
### Syncing Your Node
To sync your node, you will use systemd, which manages the Kava daemon and automatically restarts it in case of failure. To use systemd, you will create a service file.
```bash
sudo tee /etc/systemd/system/kava.service > /dev/null <<'EOF'
[Unit]
Description=Kava daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/kava start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
To start syncing:
```bash
# Start the node
sudo systemctl enable kava
sudo systemctl start kava
```
To check on the status of syncing:
```bash
kava status --output json | jq '.sync_info'
```
To check the logs of the node:
```bash
sudo journalctl -u kava -f
```

## Creating a Validator
To see a list of wallets on your node
```bash
kava keys list
```
To see the options when creating a validator:
```bash
kava tx staking create-validator -h
```
An example of creating a validator with 50KAVA self-delegation and 10% commission:
```bash
# Replace <key_name> with the key you created previously
kava tx staking create-validator \
--amount=50000000ukava \
--pubkey=$(kvd tendermint show-validator) \
--moniker="choose moniker" \
--website="optional website for your validator"
--details="optional details for your validator"
--commission-rate="0.10" \
--commission-max-rate="0.20" \
--commission-max-change-rate="0.01" \
--min-self-delegation="1" \
--from=<key_name> \
--chain-id=kava-testnet-14000 \
--gas=auto
--gas-adjustment=1.4
```
To check on the status of your validator:
```bash
kava status --output json | jq '.validator_info'
```

If you have questions, please join the active conversation in the #validators thread of the [__Kava Discord Channel__](https://discord.com/invite/kQzh3Uv).
