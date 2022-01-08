# How To Configure Your Node For Kava Testnet 14000

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
kava status --log_format json | jq '.sync_info'
```
This will give output like:
```bash
{
  "latest_block_hash": "1C2335882316936706565C20B9D65B48705AD7B4399A575E7C44CB1006C3D68D",
  "latest_app_hash": "24EC5AC12DCE697F0CBE27AEDC465FC685F1DF0E963333294D9EE00CD3C5482D",
  "latest_block_height": "1190",
  "latest_block_time": "2022-01-08T01:18:46.477424985Z",
  "earliest_block_hash": "00A76F0ADAD55C00A8CE77F3174A312E321D7238227EED3229CA42E56839A89E",
  "earliest_app_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
  "earliest_block_height": "1",
  "earliest_block_time": "2022-01-07T23:00:00Z",
  "catching_up": false
}
```
The main thing to watch is that the block height is increasing. Once you are caught up with the chain, `catching_up` will become false. At that point, you can start using your node to create a validator.

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
--pubkey=$(kava tendermint show-validator) \
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
kava status --log_format json | jq '.ValidatorInfo'
```

If you have questions, please join the active conversation in the #validators thread of the [__Kava Discord Channel__](https://discord.com/invite/kQzh3Uv).
