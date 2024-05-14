# Airchain Guide Of Setup Node & Validator
## Full Node Installation
### Install Dependencies
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
### Install Go
```
cd $HOME
VER="1.21.6"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```
### Set Variable (Change "MONIKER_NAME" With Yourself )
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="YOUR_NODE_NAME"" >> $HOME/.bash_profile
echo "export AIRCHAIN_CHAIN_ID="junction"" >> $HOME/.bash_profile
echo "export AIRCHAIN_PORT="19"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
### Install Node
```
cd $HOME
wget -O junctiond https://github.com/airchains-network/junction/releases/download/v0.1.0/junctiond
chmod +x junctiond
mv junctiond $HOME/go/bin/
```
### config and init app
```
junctiond init $MONIKER --chain-id $AIRCHAIN_CHAIN_ID 
```
### Download Genesis AddrBook
```
wget -O $HOME/.junction/config/genesis.json https://testnet-files.itrocket.net/airchains/genesis.json
wget -O $HOME/.junction/config/addrbook.json https://testnet-files.itrocket.net/airchains/addrbook.json
```
### Set Seed and Peers
```
SEEDS="04e2fdd6ec8f23729f24245171eaceae5219aa91@airchains-testnet-seed.itrocket.net:19656"
PEERS="47f61921b54a652ca5241e2a7fc4ed8663091e89@airchains-testnet-peer.itrocket.net:19656,aeaf101d54d47f6c99b4755983b64e8504f6132d@65.21.202.124:28656,df2a56a208821492bd3d04dd2e91672657c79325@158.220.126.137:27656,82af620ee9eeb2d2902ae66188eb4aa163ca8562@135.181.35.159:63656,f315edae9ff8543c50e764627a6495dfdaceb3bb@37.60.224.165:63656,c2e70f94ed3f7fa027ffad7f051c4d4688be1ee6@195.26.255.211:10056,264493e01774cccdb9baabee4af7146acbec67f2@65.21.193.80:63656,ed9e33a22fc8ee8254bb616c5ba71645345af9f0@207.180.241.242:63656,e78a440c57576f3743e6aa9db00438462980927e@5.161.199.115:26656,e3f6f7701541bd2ea183a34b061e33bfaf69ae3d@144.91.69.202:63656,613a65fe67918a5912f0cc22ef535ed1a8f0e824@65.109.112.148:4476"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.junction/config/config.toml
```
### Configuration Running
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.junction/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.junction/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.junction/config/app.toml
```
### set minimum gas price, enable prometheus and disable indexing
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.001amf"|g' $HOME/.junction/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.junction/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.junction/config/config.toml
```
### create service file
```
sudo tee /etc/systemd/system/junctiond.service > /dev/null <<EOF
[Unit]
Description=Airchains node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.junction
ExecStart=$(which junctiond) start --home $HOME/.junction
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
### reset and download snapshot
```
junctiond tendermint unsafe-reset-all --home $HOME/.junction
if curl -s --head curl https://testnet-files.itrocket.net/airchains/snap_airchains.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://testnet-files.itrocket.net/airchains/snap_airchains.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.junction
    else
  echo no have snap
fi
```
### enable and start service
```
sudo systemctl daemon-reload
sudo systemctl enable junctiond
sudo systemctl restart junctiond && sudo journalctl -u junctiond -f
```
## Wallet Create And Configuration
### to create a new wallet, use the following command. don’t forget to save the mnemonic
```
junctiond keys add $WALLET
```
### to restore exexuting wallet, use the following command
```
junctiond keys add $WALLET --recover
```
### save wallet and validator address
```
WALLET_ADDRESS=$(junctiond keys show $WALLET -a)
VALOPER_ADDRESS=$(junctiond keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```
#### check sync status, once your node is fully synced, the output from above will print "false"
```
junctiond status 2>&1 | jq 
```
### before creating a validator, you need to fund your wallet and check balance
```
junctiond query bank balances $WALLET_ADDRESS 
```
## Set FullNode to Validator
#### Create validator.json file
```
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(junctiond comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000amf\",
    \"moniker\": \"YOUR_MONIKER\",
    \"identity\": \"YOUR_KEYBASE_IDENTITY\",
    \"website\": \"YOUR_WEBSITE\",
    \"security\": \"YOUR_EMAIL\",
    \"details\": \"YOUR_DETAILED\",
    \"commission-rate\": \"0.05\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
junctiond tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id junction \
    --fees 200amf \
-y
```
