Hardware Requirements

Minimum

4CPU 8RAM 100GB
Recommended

8CPU 32RAM 200GB
Rent On Hetzner | Rent On OVH
Dependencies Installation

# Install dependencies for building from source
sudo apt update
sudo apt install -y curl git jq lz4 build-essential

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation

Node Name

Your Node Name
Port prefix

184
# Clone project repository
cd && rm -rf humans
git clone https://github.com/humansdotai/humans
cd humans
git checkout v0.2.2

# Build binary
make install

# Prepare cosmovisor directories
mkdir -p $HOME/.humansd/cosmovisor/genesis/bin
ln -s $HOME/.humansd/cosmovisor/genesis $HOME/.humansd/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which humansd) $HOME/.humansd/cosmovisor/genesis/bin

# Set node CLI configuration
humansd config chain-id humans_3000-31
humansd config keyring-backend test
humansd config node tcp://localhost:18457

# Initialize the node
humansd init "Your Node Name" --chain-id humans_3000-31

# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/humans/genesis.json > $HOME/.humansd/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/humans/addrbook.json > $HOME/.humansd/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "6ce9a9acc23594ec75516617647286fe546f83ca@humans-testnet-seed.itrocket.net:17656,b99df5397a6104fac055f21195f1fb25b77f5704@humans-testnet-peer.itrocket.net:17656,e7c7d7868fa4d9de303b1a7a028a314b7bb55ff9@65.109.92.235:33656,5e1f23a66fafd1a73871f055a6dd2165c01fe1c2@65.109.25.62:26656"|' $HOME/.humansd/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "10aheart"|' $HOME/.humansd/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.humansd/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.humansd/config/config.toml

# Change ports
sed -i -e "s%:1317%:18417%; s%:8080%:18480%; s%:9090%:18490%; s%:9091%:18491%; s%:8545%:18445%; s%:8546%:18446%; s%:6065%:18465%" $HOME/.humansd/config/app.toml
sed -i -e "s%:26658%:18458%; s%:26657%:18457%; s%:6060%:18460%; s%:26656%:18456%; s%:26660%:18461%" $HOME/.humansd/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/humans/humans_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.humansd"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/humans.service > /dev/null << EOF
[Unit]
Description=Humans AI node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.humansd
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.humansd"
Environment="DAEMON_NAME=humansd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable humans.service

# Start the service and check the logs
sudo systemctl start humans.service
sudo journalctl -u humans.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

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
