

# Install Node 

    
  # Update system and install build tools
    sudo apt -q update
    sudo apt -qy install curl git jq lz4 build-essential
    sudo apt -qy upgrade

  # Install Go
    sudo rm -rf /usr/local/go
    curl -Ls https://go.dev/dl/go1.23.2.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
    echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh
    echo 'export PATH=$PATH:$HOME/go/bin' >> $HOME/.profile

# Download project binaries
    mkdir -p $HOME/.nillionapp/cosmovisor/genesis/bin
    wget -O $HOME/.nillionapp/cosmovisor/genesis/bin/nilchaind https://snapshots.kjnodes.com/nillion-testnet/nilchaind-v0.2.2-linux-amd64
    chmod +x $HOME/.nillionapp/cosmovisor/genesis/bin/nilchaind

# Create application symlinks
    sudo ln -s $HOME/.nillionapp/cosmovisor/genesis $HOME/.nillionapp/cosmovisor/current -f
    sudo ln -s $HOME/.nillionapp/cosmovisor/current/bin/nilchaind /usr/local/bin/nilchaind -f

# Set node configuration
    nilchaind config set client chain-id nillion-chain-testnet-1
    nilchaind config set client keyring-backend test

# Initialize the node
    nilchaind init <Your_Moniker> --chain-id nillion-chain-testnet-1 --home=$HOME/.nillionapp

# Download genesis and addrbook
    wget https://raw.githubusercontent.com/Validator247/NillionApp/main/addrbook.json
    wget https://raw.githubusercontent.com/Validator247/NillionApp/main/genesis.json

# Add seeds
    sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@nillion-testnet.rpc.kjnodes.com:18059\"|" $HOME/.nillionapp/config/config.toml

# Set minimum gas price
    sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0unil\"|" $HOME/.nillionapp/config/app.toml

# Set pruning
    sed -i \
    -e 's|^pruning *=.*|pruning = "custom"|' \
    -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
    -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
    -e 's|^pruning-interval *=.*|pruning-interval = "50"|' \
    $HOME/.nillionapp/config/app.toml


# Create systemd service
    sudo tee /etc/systemd/system/nillion-testnet.service > /dev/null << EOF
    [Unit]
    Description=nillion node service
    After=network-online.target

    [Service]
    User=$USER
    ExecStart=/usr/local/bin/nilchaind start --home=$HOME/.nillionapp
    Restart=on-failure
    RestartSec=10
    LimitNOFILE=65535
    Environment="DAEMON_HOME=$HOME/.nillionapp"
    Environment="DAEMON_NAME=nilchaind"
    Environment="UNSAFE_SKIP_BACKUP=true"

    [Install]
    WantedBy=multi-user.target
    EOF

# Reload systemd and enable the service
    sudo systemctl daemon-reload
    sudo systemctl enable nillion-testnet.service

# Start service and check the logs
    sudo systemctl start nillion-testnet.service && sudo journalctl -u nillion-testnet.service -f --no-hostname -o cat
