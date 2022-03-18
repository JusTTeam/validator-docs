# Validator Guide for Kava EVM Alpha (Kava-10) Testnet on Ubuntu 20.04
Devs repo you can find [here](https://github.com/Kava-Labs/kava-testnets/tree/master/15000).   
More info about Kava-10 you can find [here](https://medium.com/kava-labs/kava-10-kava-network-1-0-ba9195a029b7).  
If you have questions, please join the active conversation in the #validators thread of the [__Kava Discord Channel__](https://discord.com/invite/kQzh3Uv).
## Installing Kava

### Prerequisites
You should select an all-purpose server with at least 8GB of RAM, good connectivity, and a solid state drive with sufficient disk space. Storage requirements are discussed further in the section below. In addition, you’ll need to open **port 26656** to connect to the Kava peer-to-peer network. As the usage of the blockchain grows, the server requirements may increase as well, so you should have a plan for updating your server as well.

### Storage
The monthly storage requirements for a node are as follows. These are estimated values based on experience, but should serve as a good guide.

- An archival node (`pruning = "nothing"`) grows at a rate of ~100 GB per month
- A fully pruning node (`pruning = "everything"`) grows at a rate of ~5 GB per month
- A default pruning node (`pruning = “default”`) grows at a rate of ~25 GB per month

## Install Go
Kava is built using Go and requires Go version 1.17+. In this example, you will be installing Go on a fresh install on Ubuntu 20.04.

```bash
# Update Ubuntu
sudo apt update && sudo apt upgrade -y

# Install packages necessary to run go and jq for pretty formatting command line outputs
sudo apt install build-essential jq -y

# Install go (latest version at https://golang.org/dl/)
wget https://dl.google.com/go/go1.18.linux-amd64.tar.gz 
sudo tar -xvf go1.18.linux-amd64.tar.gz
sudo mv go /usr/local

# Updates environmental variables to include go
cat <<EOF>> ~/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF

source ~/.profile
```
To verify that Go is installed:
```bash
go version
# Should return go version go1.18 linux/amd64
```
## Install Kava
Install Kava using `git clone`. Note that version v0.17.0-alpha.1 is the current version for testnet.
```bash
# Install git
sudo apt install git

git clone https://github.com/kava-labs/kava
cd kava
git checkout v0.17.0-alpha.1
make install
```
To verify that kava is installed:
```bash
kava version --long
# name: kava
# server_name: kava
# version: 0.17.0-alpha.1
# commit: 14e9e047e213b4341c62db3233352f8d0ae5c7af
# build_tags: netgo,ledger
# go: go version go1.18 linux/amd64
```
## Configuring Your Node
Next, download the correct genesis file and sync your node with the Kava testnet. To download the genesis file:
```bash
# First, initialize kava. Replace <name> with the public name of your node
kava init --chain-id kava_2221-15000 <name>
# Download the genesis file
wget https://raw.githubusercontent.com/Kava-Labs/kava-testnets/master/15000/genesis.json -O ~/.kava/config/genesis.json
# Verify genesis hash
jq -S -c -M '' $HOME/.kava/config/genesis.json | shasum -a 256
# 66f1f7cae99692f433e37e9514ad2da3e83e01cdb72cf56eacfe7c742a271e57
```
Next,  adjust some configurations. To open the config file:
```bash
vim $HOME/.kava/config/config.toml
```
At line 215, add some persistent peers *(b57674fc9b810facdc9fbceb64065484884f8911@35.174.166.39:26656)*, which help maintain a connection to the peer-to-peer network.  
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
## Syncing Your Node
To sync your node, you will use systemd, which manages the Kava daemon and automatically restarts it in case of failure. To use systemd, you will create a service file. Be sure to replace `<your_user>` with the user on your server:
```bash
sudo tee /etc/systemd/system/kavad.service > /dev/null <<'EOF'
[Unit]
Description=Kava daemon
After=network-online.target

[Service]
User=<your_user>
ExecStart=/home/<your_user>/go/bin/kava start
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
sudo systemctl enable kavad
sudo systemctl start kavad
```
To check on the status of syncing:
```bash
kava status --log_format json | jq '.sync_info'
```
This will give output like:
```bash
{
  "latest_block_hash": "31D88B5D350840F185C28EE0AF3D92DDF123DBEF075569C65AD61BEC2893E391",
  "latest_app_hash": "8B086382576FC1833BB5F4500EB5F8D54D43EEDE2DB1960298C88D5D63DF3700",
  "latest_block_height": "206",
  "latest_block_time": "2022-03-08T00:17:03.951809275Z",
  "earliest_block_hash": "8A97AA603B2C5BB48B8BC668A9FE6D922E60E63CDEB0BDE51C3F67672F13626F",
  "earliest_app_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
  "earliest_block_height": "1",
  "earliest_block_time": "2022-03-08T00:00:00Z",
  "catching_up": true
}
```
The main thing to watch is that the block height is increasing. Once you are caught up with the chain, `catching_up` will become false. At that point, you can start using your node to create a validator.  
To check the logs of the node:
```bash
sudo journalctl -u kavad -f
```
## Creating a Validator
First, create a wallet, which will give you a private key / public key pair for your node.
```bash
# Replace <your-key-name> with a name for your key that you will remember
kava keys add <your-key-name>
# To see a list of wallets on your node
kava keys list
```
**Be sure to write down the mnemonic for your wallet and store it securely. Losing your mnemonic could result in the irrecoverable loss of KAVA tokens.**
To see the options when creating a validator:
```bash
kava tx staking create-validator -h
```
**Note: Before creating a validator, don't forget to go to #validators thread in Discord and request testnet tokens to your address.**   
An example of creating a validator with 50KAVA self-delegation and 10% commission:
```bash
# Replace <key_name> with the key you created previously
kava tx staking create-validator \
--amount=50000000ukava \
--pubkey=$(kava tendermint show-validator) \
--moniker="choose moniker" \
--website="optional website for your validator" \
--details="optional details for your validator" \
--commission-rate="0.10" \
--commission-max-rate="0.20" \
--commission-max-change-rate="0.01" \
--min-self-delegation="1" \
--from=<your-key-name> \
--chain-id=kava_2221-15000 \
--gas=auto \
--gas-adjustment=1.4
```
To check on the status of your validator:
```bash
kava status --log_format json | jq '.ValidatorInfo'
```
Checkout Kava Testnet explorer: https://testnet.mintscan.io/kava-testnet

If you have questions, please join the active conversation in the #validators thread of the [__Kava Discord Channel__](https://discord.com/invite/kQzh3Uv).
