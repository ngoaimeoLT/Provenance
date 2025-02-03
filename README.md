Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.23.1"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export PROVENANCE_CHAIN_ID="pio-mainnet-1"" >> $HOME/.bash_profile
echo "export PROVENANCE_PORT="57"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf bin
wget https://github.com/provenance-io/provenance/releases/download/v1.21.0/provenance-linux-amd64-v1.21.0.zip
unzip provenance-linux-amd64-v1.20.3.zip
chmod +x $HOME/bin/provenanced
rm provenance-linux-amd64-v1.20.3.zip
mv ~/bin/provenanced ~/go/bin/
sudo cp ~/bin/libwasmvm.x86_64.so /usr/lib/
sudo ldconfig
export PIO_HOME=~/.provenanced
```

**config and init app**
```
provenanced init $MONIKER --chain-id $PROVENANCE_CHAIN_ID
provenanced config set node tcp://localhost:${PROVENANCE_PORT}657
sed -i -e 's/namespace = "cometbft"/namespace = "provenance"/' $HOME/.provenanced/config/config.toml
```

# download genesis and addrbook
wget -O $HOME/.provenanced/config/genesis.json https://server-5.itrocket.net/mainnet/provenance/genesis.json
wget -O $HOME/.provenanced/config/addrbook.json  https://server-5.itrocket.net/mainnet/provenance/addrbook.json

# set seeds and peers
SEEDS="a280ec7a1b563cb71510723b860ed37d40494308@provenance-mainnet-seed.itrocket.net:57656"
PEERS="b75bb5d0c033b5a8ca24df607a757d09e4f99acd@provenance-mainnet-peer.itrocket.net:57656,a5a160c50ba3b8e276c6fbe2815d0be3347c2394@20.237.232.187:26656,2d4bf27ccf0a1ee146b71f4770d7df5efdc050a1@35.196.232.201:26656,7c10fcd87e07163c928f859e87b3a2fb124998fa@65.21.167.185:27056,9fa182c140f35a36f1565a580a7567903292d8a9@65.21.91.160:27100,244cf2cb869ff1883eeff674b1209a3407444898@64.226.78.86:26260,3e9b2ae79ed091e909ec30baa51b0e0cdcf97c3e@65.108.201.240:27056,7e2c9022216834e5136daae9a1b105a031977f3c@65.109.28.177:29656,f35bcf766eb67b44b05ceb928b2a87f39c3e805a@34.230.174.108:26656,e6cc4562f2db800e0cfc2f103067a09b4dc5b7e8@104.196.172.172:26656,82aa2d9d8e88618502916f11720e4bb3763c730b@46.4.23.225:29656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.provenanced/config/config.toml

# set custom ports in app.toml
sed -i.bak -e "s%:1317%:${PROVENANCE_PORT}317%g;
s%:8080%:${PROVENANCE_PORT}080%g;
s%:9090%:${PROVENANCE_PORT}090%g;
s%:9091%:${PROVENANCE_PORT}091%g;
s%:8545%:${PROVENANCE_PORT}545%g;
s%:8546%:${PROVENANCE_PORT}546%g;
s%:6065%:${PROVENANCE_PORT}065%g" $HOME/.provenanced/config/app.toml

# set custom ports in config.toml file
sed -i.bak -e "s%:26658%:${PROVENANCE_PORT}658%g;
s%:26657%:${PROVENANCE_PORT}657%g;
s%:6060%:${PROVENANCE_PORT}060%g;
s%:26656%:${PROVENANCE_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${PROVENANCE_PORT}656\"%;
s%:26660%:${PROVENANCE_PORT}660%g" $HOME/.provenanced/config/config.toml

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.provenanced/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.provenanced/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.provenanced/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "1905nhash"|g' $HOME/.provenanced/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.provenanced/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.provenanced/config/config.toml

# create service file
sudo tee /etc/systemd/system/provenanced.service > /dev/null <<EOF
[Unit]
Description=Provenance node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.provenanced
ExecStart=$(which provenanced) start --home $HOME/.provenanced
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
provenanced tendermint unsafe-reset-all --home $HOME/.provenanced
if curl -s --head curl https://server-5.itrocket.net/mainnet/provenance/provenance_2025-01-26_21679435_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/mainnet/provenance/provenance_2025-01-26_21679435_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.provenanced
    else
  echo "no snapshot found"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable provenanced
sudo systemctl restart provenanced && sudo journalctl -u provenanced -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/mainnet/provenance/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
provenanced keys add $WALLET

# to restore exexuting wallet, use the following command
provenanced keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(provenanced keys show $WALLET -a)
VALOPER_ADDRESS=$(provenanced keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
provenanced status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
provenanced query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.provenanced/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://provenance-mainnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, nhash
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(provenanced comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000nhash\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
provenanced tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id pio-mainnet-1 \
	--gas auto --gas-adjustment 1.5
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${PROVENANCE_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop provenanced
sudo systemctl disable provenanced
sudo rm -rf /etc/systemd/system/provenanced.service
sudo rm $(which provenanced)
sudo rm -rf $HOME/.provenanced
sed -i "/PROVENANCE_/d" $HOME/.bash_profile
