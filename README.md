# Nibiro
v0.16.3 / nibiru-testnet-2

#### Setup validator name (replace the field "YOUR_MONIKER_GOES_HERE" with your own)
```
MONIKER="YOUR_MONIKER_GOES_HERE"
```
### Install dependencies
#### Update system and install build tools
```
sudo apt update
sudo apt install curl git jq lz4 build-essential -y
```
#### Install GO
```
sudo rm -rf /usr/local/go
sudo curl -Ls https://golang.org/dl/go1.19.4.linux-amd64.tar.gz | sudo tar -C /usr/local -xz
tee -a $HOME/.profile > /dev/null << EOF
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
```
#### Download and build binaries
```
cd $HOME
rm -rf nibiru
git clone https://github.com/NibiruChain/nibiru.git
cd nibiru
git checkout v0.16.3
make build
mkdir -p $HOME/.nibid/cosmovisor/genesis/bin
mv build/nibid $HOME/.nibid/cosmovisor/genesis/bin/
rm -rf build
```
#### Install Cosmovisor and create a service
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
sudo tee /etc/systemd/system/nibid.service > /dev/null << EOF
[Unit]
Description=nibiru-testnet node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.nibid"
Environment="DAEMON_NAME=nibid"
Environment="UNSAFE_SKIP_BACKUP=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable nibid
ln -s $HOME/.nibid/cosmovisor/genesis $HOME/.nibid/cosmovisor/current
sudo ln -s $HOME/.nibid/cosmovisor/current/bin/nibid /usr/local/bin/nibid
```
#### Initialize the node
```
nibid config chain-id nibiru-testnet-2
nibid config keyring-backend test
nibid config node tcp://localhost:39657
nibid init $MONIKER --chain-id nibiru-testnet-2
curl -Ls https://snapshots.kjnodes.com/nibiru-testnet/genesis.json > $HOME/.nibid/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/nibiru-testnet/addrbook.json > $HOME/.nibid/config/addrbook.json
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@nibiru-testnet.rpc.kjnodes.com:39659\"|" $HOME/.nibid/config/config.toml
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025unibi\"|" $HOME/.nibid/config/app.toml
sed -i -e "s|^pruning *=.*|pruning = \"custom\"|" $HOME/.nibid/config/app.toml
sed -i -e "s|^pruning-keep-recent *=.*|pruning-keep-recent = \"100\"|" $HOME/.nibid/config/app.toml
sed -i -e "s|^pruning-keep-every *=.*|pruning-keep-every = \"0\"|" $HOME/.nibid/config/app.toml
sed -i -e "s|^pruning-interval *=.*|pruning-interval = \"19\"|" $HOME/.nibid/config/app.toml
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:39658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:39657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:39060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:39656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":39660\"%" $HOME/.nibid/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:39317\"%; s%^address = \":8080\"%address = \":39080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:39090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:39091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:39545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:39546\"%" $HOME/.nibid/config/app.toml
```
#### Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/nibiru-testnet/snapshot_latest.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.nibid
```
#### Start service and check the logs
```
sudo systemctl start nibid && journalctl -u nibid -f --no-hostname -o cat
```
## Below enter one of the commands to create a new wallet or restore the old one depending on what you need. 
#### Add new key 
```
nibid keys add wallet
```
#### Recover existing key
```
nibid keys add wallet --recover
```
#### Wait 1-2 hours until the node is fully synchronized and create a validator 
## Please make sure you have adjusted moniker, identity, details and website to match your values. (You can leave just "YOUR_MONIKER_NAME" and other values such as "YOUR_KEYBASE_ID", "YOUR_DETAILS", "YOUR_WEBSITE_URL" just erase if you do not have them.)  
```
nibid tx staking create-validator \
--amount=1000000unibi \
--pubkey=$(nibid tendermint show-validator) \
--moniker="YOUR_MONIKER_NAME" \
--identity="YOUR_KEYBASE_ID" \
--details="YOUR_DETAILS" \
--website="YOUR_WEBSITE_URL"
--chain-id=nibiru-testnet-2 \
--commission-rate=0.05 \
--commission-max-rate=0.20 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=wallet \
--gas-adjustment=1.4 \
--gas=auto \
--gas-prices=0.025unibi \
-y
```
## Useful commands
#### Query wallet balance
```
nibid q bank balances $(nibid keys show wallet -a)
```
#### Unjail validator
```
nibid tx slashing unjail --from wallet --chain-id nibiru-testnet-2 --gas-adjustment 1.4 --gas auto --gas-prices 0.025unibi -y
```
#### Check service logs
```
sudo journalctl -u nibid -f --no-hostname -o cat
```
