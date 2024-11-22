Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

**Clone project repository**
```
cd && rm -rf axoned
git clone https://github.com/axone-protocol/axoned
cd axoned
git checkout v10.0.0
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.axoned/cosmovisor/genesis/bin
ln -s $HOME/.axoned/cosmovisor/genesis $HOME/.axoned/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**

cp $(which axoned) $HOME/.axoned/cosmovisor/genesis/bin

**Set node CLI configuration**
```
axoned config chain-id axone-dentrite-1
axoned config keyring-backend test
axoned config node tcp://localhost:17657
```

**Initialize the node**
```
axoned init "Your Node Name" --chain-id axone-dentrite-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots-testnet.nodejumper.io/axone/genesis.json > $HOME/.axoned/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/axone/addrbook.json > $HOME/.axoned/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "3f472746f46493309650e5a033076689996c8881@axone-testnet.rpc.kjnodes.com:13659"|' $HOME/.axoned/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.001uaxone"|' $HOME/.axoned/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.axoned/config/app.toml
```

**Enable prometheus**
```
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.axoned/config/config.toml
```

**Change ports**
```
sed -i -e "s%:1317%:17617%; s%:8080%:17680%; s%:9090%:17690%; s%:9091%:17691%; s%:8545%:17645%; s%:8546%:17646%; s%:6065%:17665%" $HOME/.axoned/config/app.toml
sed -i -e "s%:26658%:17658%; s%:26657%:17657%; s%:6060%:17660%; s%:26656%:17656%; s%:26660%:17661%" $HOME/.axoned/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots-testnet.nodejumper.io/axone/axone_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.axoned"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

**Create a service**
```
sudo tee /etc/systemd/system/axone.service > /dev/null << EOF
[Unit]
Description=Axone node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.axoned
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.axoned"
Environment="DAEMON_NAME=axoned"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable axone.service
```

**Start the service and check the logs**
```
sudo systemctl start axone.service
sudo journalctl -u axone.service -f --no-hostname -o cat
Secure Server Setup (Optional)
```

**generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE**
```
ssh-keygen -t rsa
```

**save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY**
```
cat ~/.ssh/id_rsa.pub
```

**upgrade system packages**
```
sudo apt update
sudo apt upgrade -y
```

**add new admin user**
```
sudo adduser admin --disabled-password -q
```


**upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above**
```
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**disable root login, disable password authentication, use ssh keys only**
```
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd
```

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
