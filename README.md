# 5ire full node systemd
start 5ire full node as normal service systemd

The fight against docker crafts continues. 

This time, the 5ire node fell under the knife.

Official documentation here https://docs.5ire.org/docs/system-admin/How-to-setup-a-5ireChain-node , only 2 commands in console. BUT, with each restart - a new container, and, accordingly, full synchronization. You can't really see the logs. Well, other delights of docker

We will run it as a normal service.
Further commands, without much text, and so everything is clear


## set env
```bash
NAME=YOUR_NODE_NAME
IMAGE=5irechain/5ire-thunder-node:0.12
DOCKER_NAME=5ire-thunder-node
BIN=5irechain 
BIN_DIR=$HOME/bin
DATA_DIR=$HOME/.5irechain
BACKUP_DIR=$HOME/backup
DAEMON=firefulld                
DESCRIPTION="5irechain full node"
```

## update system // 
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl jq
```


## docker 
```bash
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

## get-docker-image 
```bash
docker pull $IMAGE
docker create --name $DOCKER_NAME $IMAGE
```

## copy-files-from-docker-image
```bash
mkdir $BIN_DIR $DATA_DIR
docker cp $DOCKER_NAME:5ire/5irechain $BIN_DIR/5irechain
docker cp $DOCKER_NAME:5ire/thunder-chain-spec.json $DATA_DIR/thunder-chain-spec.json
echo "export PATH=$PATH:$BIN_DIR" >> $HOME/.bash_profile
```

## test-run
```bash
$BIN_DIR/$BIN \
    --no-telemetry \
    --name $NAME \
    --base-path $DATA_DIR \
    --keystore-path $DATA_DIR \
    --chain $DATA_DIR/thunder-chain-spec.json \
    --node-key-file $DATA_DIR/secrets/node.key \
    --bootnodes /ip4/3.128.99.18/tcp/30333/p2p/12D3KooWSTawLxMkCoRMyzALFegVwp7YsNVJqh8D2p7pVJDqQLhm \
    --pruning archive \
    --ws-external \
    --rpc-external \
    --rpc-cors all \
    --in-peers 5 --out-peers 5 --in-peers-light 0 --log info \
    --prometheus-port 9615 --ws-port 9944 --rpc-port 9933 --port 30333
```

## make-service
```bash
echo "
[Unit]
Description=$DESCRIPTION
After=network.target
[Service]
Type=simple
User=$USER
ExecStart= $(which $BIN) \\
    --no-telemetry \\
    --name $NAME \\
    --base-path $DATA_DIR \\
    --keystore-path $DATA_DIR \\
    --chain $DATA_DIR/thunder-chain-spec.json \\
    --node-key-file $DATA_DIR/secrets/node.key \\
    --bootnodes /ip4/3.128.99.18/tcp/30333/p2p/12D3KooWSTawLxMkCoRMyzALFegVwp7YsNVJqh8D2p7pVJDqQLhm \\
    --pruning archive \\
    --ws-external \\
    --rpc-external \\
    --rpc-cors all \\
    --in-peers 5 --out-peers 5 --in-peers-light 0 --log info \\
    --prometheus-port 9615 --ws-port 9944 --rpc-port 9933 --port 30333
Restart=always
RestartSec=10
KillSignal=SIGINT
TimeoutStopSec=45
KillMode=mixed 
LimitNOFILE=4096
[Install]
WantedBy=multi-user.target
" > $DAEMON.service
```

## en-start-service
```bash
sudo cp $DAEMON.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable $DAEMON
sudo systemctl restart $DAEMON
journalctl -f -u $DAEMON
```

## backup-node-key
```bash
mkdir $BACKUP_DIR
cp $DATA_DIR/secrets/node.key $BACKUP_DIR
```

## open ports
```bash
sudo ufw allow 30333/tcp
sudo ufw status | grep 30333
```

##  useful commands
see logs
```bash
journalctl -n 100 -f -u $DAEMON | grep -v \\
    -e "✨ Imported" \\
    -e "💤 Idle" \\
    -e "♻ ️  Reorg"
```

see node id 
```bash
curl --silent -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "method": "system_localPeerId" }' \
http://localhost:9933 |jq ."result" 
```

## update node !TBL! 
```bash
update docker image
copy new binary && chain
```

