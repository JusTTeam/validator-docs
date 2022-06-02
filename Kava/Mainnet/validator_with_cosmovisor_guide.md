# Validator Guide (with Cosmovisor) for Kava-10 Mainnet on Ubuntu 20.04
This is a guide to setting up the mainnet validator in Kava-10 with Cosmovisor. Most of the code and descriptions are taken from the original guide that you can find [here](https://github.com/Kava-Labs/docusaurus-kava/blob/main/docs/participate/validator-node.mdx).   

Note that this is a minimal guide and does not cover more advanced topics like [sentry node architecture](https://github.com/stakefish/cosmos-validator-design) and [double signing protection](https://github.com/tendermint/tmkms). It is strongly recommended that any parties considering validating do additional research.  If you have questions, please join the active conversation in the #validators thread of our [__Discord Channel__](https://discord.com/invite/kQzh3Uv).
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
wget https://dl.google.com/go/go1.18.3.linux-amd64.tar.gz 
sudo tar -xvf go1.18.3.linux-amd64.tar.gz
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
# Should return go version go1.18.3 linux/amd64
```
## Install Kava
Install Kava using `git clone`. Note that version 0.17.4 is the correct version for mainnet.
```bash
# Install git
sudo apt install git

git clone https://github.com/kava-labs/kava
cd kava
git checkout v0.17.4
make install
```
To verify that kava is installed:
```bash
kava version --long
# name: kava
# server_name: kava
# version: 0.17.4
# commit: 98ceba15005eda0d3ff40a5152480335f9ab9971
# build_tags: netgo,ledger
# go: go version go1.18.3 linux/amd64
```
## Configuring Your Node
Next, download the correct genesis file and sync your node with the Kava mainnet. To download the genesis file:
```bash
# First, initialize kava. Replace <name> with the public name of your node
kava init --chain-id kava_2222-10 <name>
# Download the genesis file
wget https://kava-genesis-files.s3.us-east-1.amazonaws.com/kava_2222-10/genesis.json -O ~/.kava/config/genesis.json
# Verify genesis hash
jq -S -c -M '' $HOME/.kava/config/genesis.json | shasum -a 256
# 3bc9829faf3beae2892ff1dfb7158f41a3f0cff303b4798777a559250a4dc815
```
Next,  adjust some configurations. To open the config file:
```bash
vim $HOME/.kava/config/config.toml
```
At line 212, add [seeds](https://docs.google.com/spreadsheets/d/1s3LXLPFJazzdmwKdRv909pe_-ZV_Xbt2uCCNrhoOyIE). These are used to connect to the peer-to-peer network:
At line 215, add some [persistent peers](https://docs.google.com/spreadsheets/d/1s3LXLPFJazzdmwKdRv909pe_-ZV_Xbt2uCCNrhoOyIE), which help maintain a connection to the peer-to-peer network
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
minimum-gas-prices = "0.001ukava;1000000000akava"
```
## Install Cosmovisor
Next, install Cosmovisor version 1.0.0
```bash
git clone https://github.com/cosmos/cosmos-sdk
cd cosmos-sdk
git checkout cosmovisor/v1.0.0
make cosmovisor
cp cosmovisor/cosmovisor ~/go/bin/cosmovisor
echo $(which cosmovisor)
```
Put the node binary files in Cosmovisor directory
```bash
mkdir -p $HOME/.kava/cosmovisor/genesis/bin $HOME/.kava/cosmovisor/upgrades; \
cp `which kava` $HOME/.kava/cosmovisor/genesis/bin
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
Environment="DAEMON_NAME=kava"
Environment="DAEMON_HOME=/home/<your_user>/.kava"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
ExecStart=/home/<your_user>/go/bin/cosmovisor start
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
  "latest_block_hash": "920E4D32F02CFF8064D26DD7D34C65DC623F4026C65C192BBCD7DBF19AFB5630",
  "latest_app_hash": "442E7E55982109D9F73467EA0E374312B402AE620DEC81CB3441E949ED0D0A29",
  "latest_block_height": "2437",
  "latest_block_time": "2022-05-25T23:07:36.752766828Z",
  "earliest_block_hash": "9D2AF876309BB9174604004A813DCFEE94F4947B08C5BB4C1A042F318488851E",
  "earliest_app_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
  "earliest_block_height": "1",
  "earliest_block_time": "2022-05-25T17:00:00Z",
  "catching_up": true
}
```
The main thing to watch is that the block height is increasing. Once you are caught up with the chain, `catching_up` will become false. At that point, you can start using your node to create a validator. If you need to sync using a snapshot, please use https://kava.quicksync.io/
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
--chain-id=kava_2222-10 \
--gas=auto \
--gas-adjustment=1.4
```
To check on the status of your validator:
```bash
kava status --log_format json | jq '.ValidatorInfo'
```
After you have completed this guide, your validator should be up and ready to receive delegations. Note that only the top 100 validators by weighted stake (self-delegations + other delegations) are eligible for block rewards. To view the current validator list, checkout one of the Kava block explorers:
- https://www.mintscan.io/kava
- https://kava.bigdipper.live/
- https://kavascan.com/

If you have questions, please join the active conversation in the #validators thread of the [__Kava Discord Channel__](https://discord.com/invite/kQzh3Uv).
