# Defund
DeFund node


# install dependecies

sudo apt-get update && sudo apt-get upgrade -y
sudo apt install make clang pkg-config libssl-dev build-essential git jq ncdu bsdmainutils htop net-tools lsof fail2ban -y

cd $HOME
wget -O go1.17.1.linux-amd64.tar.gz https://golang.org/dl/go1.17.linux-amd64.tar.gz
rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.17.1.linux-amd64.tar.gz && rm go1.17.1.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
echo 'export GO111MODULE=on' >> $HOME/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile && . $HOME/.bash_profile
go version

git clone https://github.com/defund-labs/defund
cd defund
make install

# треугольные скобки убрать!
defundd init <ИмяВалидатора> --chain-id=defund-private-1

wget -O $HOME/.defund/config/genesis.json https://raw.githubusercontent.com/schnetzlerjoe/defund/main/testnet/private/genesis.json
defundd tendermint unsafe-reset-all

wget -O $HOME/.defund/config/addrbook.json https://raw.githubusercontent.com/sowell-owen/defund-addrbook/main/addrbook.json

# pruning (необязательно)
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"

sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.defund/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.defund/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.defund/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.defund/config/app.toml

# вставлять одной строкой
sudo tee /etc/systemd/system/defund.service > /dev/null <<EOF
Description=DeFund Node
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which defundd) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
###

sudo systemctl daemon-reload
sudo systemctl enable defund
sudo systemctl restart defund
journalctl -u defund -f -o cat

# ждём синхронизации (cathcing_up: false)
defundd 2<&1 status | jq

# создаем кошелёк и сохраняем мнемоник (скобки убрать)
defundd keys add <ИмяКошелька> 

# посмотреть адрес кошелька
defundd keys show <ИмяКошелька> 

# запрашиваем монеты в кране (ссылка в конце статьи)
# проверяем баланс
defundd q bank balances <АдресКошелька>

# создаем валидатора (подставить свои значения)
defundd tx staking create-validator \
  --amount=1900000ufetf \
  --pubkey=$(defundd tendermint show-validator) \
  --moniker=<ИмяВалидатора> \
  --chain-id=defund-private-1 \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1000000" \
  --gas="auto" \
  --from=<ИмяКошелька>
