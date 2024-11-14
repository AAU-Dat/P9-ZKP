# Shortened guide to setting up Prysm DevNet (Ubuntu)
## First step
- Install GoLang
- Use a linux GNU screen on the `remote server` supposed to run the `devnet` 
## Create DevNet folder
```bash
mkdir devnet && cd devnet
```
## Clone prysm master branch
```bash
git clone https://github.com/prysmaticlabs/prysm && cd prysm && git checkout master
go build -o=../beacon-chain ./cmd/beacon-chain
go build -o=../validator ./cmd/validator
go build -o=../prysmctl ./cmd/prysmctl
cd ..
```

## Clone geth release/1.14 branch (stable)
```bash
git clone https://github.com/ethereum/go-ethereum && cd go-ethereum && git checkout release/1.14
make geth
cp ./build/bin/geth ../geth
cd ..
```

## Create JWT token
```bash
touch jwt.hex && nano jwt.hex
```
Copy and insert the following:
```bash
0xfad2709d0bb03bf0e8ba3c99bea194575d3e98863133d1af638ed056d1d59345
```
Close the nano terminal 
1. `ctrl+x`
2. `shift+y`
3. `enter`

## Transfer config.yml to devnet
Download the file [config.yml](https://github.com/OffchainLabs/eth-pos-devnet/blob/master/consensus/config.yml)

In a local windows terminal:
```bash
scp -i ~/.ssh/*ssh key* *Path to file* *server path*
```
In our case, fx.:
```bash
scp -i ~/.ssh/desktop C:\Users\ander\Downloads\config.yml ubuntu@10.92.1.165:~/devnet/
```

## Transfer genesis.json to devnet
Download the file [genesis.json](https://github.com/OffchainLabs/eth-pos-devnet/blob/master/execution/genesis.json)

In a local windows terminal:
```bash
scp -i ~/.ssh/*ssh key* *Path to file* *server path*
```
In our case, fx.:
```bash
scp -i ~/.ssh/desktop C:\Users\ander\Downloads\genesis.json ubuntu@10.92.1.165:~/devnet/
```

## Set genesis time
We are now in `devnet` folder on remote server
```bash
./prysmctl testnet generate-genesis --fork capella --num-validators 64 --genesis-time-delay 600 --chain-config-file config.yml --geth-genesis-json-in genesis.json  --geth-genesis-json-out genesis.json --output-ssz genesis.ssz
```

## Create file with secret key for first node
In `devnet` folder
```bash
touch secret.txt && nano secret.txt
```
Copy and paste the following
```bash
2e0834786285daccd064ca17f1654f67b4aef298acbb82cef9ec422fb4975622
```
Close the nano terminal 
1. `ctrl+x`
2. `shift+y`
3. `enter`

Run command:
```bash
./geth --datadir=gethdata account import secret.txt
```
Choose password - WILL BE USED TO START GETH

## Run first geth node
In `devnet` folder. Initialize genesis:
```bash
./geth --datadir=gethdata init genesis.json
```
Run geth:
```bash
./geth --http --http.api eth,net,web3 --ws --ws.api eth,net,web3 --authrpc.jwtsecret jwt.hex --datadir gethdata --nodiscover --syncmode full --allow-insecure-unlock --unlock 0x123463a4b065722e99115d6c222f267d9cabb524
```
Before entering password, copy the `enode` field hidden in the logs

After entering password, wait for the log output somewhere saying
```bash
INFO [01-09|18:38:23.618] Unlocked account                         address=0x123463a4B065722E99115D6c222f267d9cABb524
WARN [01-09|18:38:58.164] Post-merge network, but no beacon client seen. Please launch one to follow the chain!
```

## Run first prysm node
The following should be done fairly quickly after each other, because the chain will start counting down to genesis, where the chain begins working.

In another `screen`, still in the `devnet` folder. 

Run the following:
```
./beacon-chain --datadir beacondata --min-sync-peers 0 --genesis-state genesis.ssz --bootstrap-node= --interop-eth1data-votes --chain-config-file config.yml --contract-deployment-block 0 --chain-id 32382 --accept-terms-of-use --jwt-secret jwt.hex --suggested-fee-recipient 0x123463a4B065722E99115D6c222f267d9cABb524 --minimum-peers-per-subnet 0 --enable-debug-rpc-endpoints --execution-endpoint gethdata/geth.ipc
```
Continue with next step
## Run first batch of validators
Shortly after, in a new `screen` run this:
```
./validator --datadir validatordata --accept-terms-of-use --interop-num-validators 64 --chain-config-file config.yml
```
## Running chain
Prysm node (beacon chain) will start counting down
```bash
[2024-01-10 14:42:18]  INFO slotutil: 6m55s until chain genesis genesisStateRoot=a97895d2be4529508cf8d61d9eeb02629a59e0bcbacbf24b14b0d9a7ceec4b3a genesisTime=2024-01-10 14:49:14 +0100 CET genesisValidators=64
```
When done, chain should be running

## Setup for 2nd node
Run the following command:
```bash
curl localhost:8080/p2p
```
Two parts need to be copied. 

1. Copy the part starting with `enr` until the first comma, like this:

- `enr:-MK4QCxV5SEkUO1chqZDSqMChX5fTbqeas4PEJqZtzmcWqZOKpZN8ABVrQFTqHI74M9TKNjE6DPAAgyv5JydsQ6NfPqGAYKxymm8h2F0dG5ldHOI-7v9vd4GdE2EZXRoMpCMkQYoIAAAkf__________gmlkgnY0gmlwhAoAAEqJc2VjcDI1NmsxoQNZcfvPnVEfnKz-mFv285nkDzgRXRVujloXQ_tjuCNEbYhzeW5jbmV0cw-DdGNwgjLIg3VkcIIu4A`

2. Copy the part starting with `/ip4`, like this:
-  `/ip4/10.0.0.74/tcp/13000/p2p/16Uiu2HAmJg9Sfy8bX4wyjZNTi8soJrdPt9E9pPzJSmewN5rLoRM6`

set an environmental variable with the `/ip4` part:
```bash
export PEER=/ip4/10.0.0.74/tcp/13000/p2p/16Uiu2HAmJg9Sfy8bX4wyjZNTi8soJrdPt9E9pPzJSmewN5rLoRM6
```

## Starting the 2nd geth node
In a new screen, still in the `devnet` folder, use the following command, but replace the bootnodes parameter with the previously copied `enode` from the 1st geth node:
```bash
./geth --datadir=gethdata2 init genesis.json
./geth --http --http.api eth,net,web3 --ws --ws.api eth,net,web3 --authrpc.jwtsecret jwt.hex --datadir gethdata2 --nodiscover --syncmode full --discovery.port 30304 --port 30304 --http.port 8547 --ws.port 8548 --authrpc.port 8552 --bootnodes *insert enode here*
```
Wait for log output to say:
```bash
WARN [01-09|18:38:58.164] Post-merge network, but no beacon client seen. Please launch one to follow the chain!
```
## Starting the 2nd prysm node
In a new `screen`, run the following command with the `enr` (from `Setup for 2nd node`) inserted as the parameter for `bootstrap-node`:
```bash
 ./beacon-chain --datadir beacondata2 --min-sync-peers 1 --genesis-state genesis.ssz --bootstrap-node *INSERT ENR HERE* --interop-eth1data-votes --chain-config-file config.yml --contract-deployment-block 0 --chain-id 32382 --accept-terms-of-use --jwt-secret jwt.hex --suggested-fee-recipient 0x123463a4B065722E99115D6c222f267d9cABb524 --minimum-peers-per-subnet 0 --enable-debug-rpc-endpoints --execution-endpoint gethdata2/geth.ipc --peer=$PEER --p2p-udp-port 12001 --p2p-tcp-port 13001 --grpc-gateway-port 3501 --rpc-port 4001
```

## Setting up 64 validators for 2nd node
In another `screen`, in the `devnet` folder, use the following command:
```bash
./validator --datadir validatordata2 --accept-terms-of-use --interop-num-validators 64 --interop-start-index 64 --chain-config-file config.yml --beacon-rest-api-provider http://127.0.0.1:3501 --beacon-rpc-provider 127.0.0.1:4001 --graffiti "validator2" --clear-db --monitoring-port 8082
```

# Making the crawler node
Make a new directory outside the `devnet` directory, and import the modified prysm into it. We have decided to call it `mod-prysm`.

## Starting the 3rd geth node
In a new screen, still in the `devnet` folder, use the following command, but replace the bootnodes parameter with the previously copied `enode` from the 1st geth node:
```bash
./geth --datadir=gethdata3 init genesis.json
./geth --http --http.api eth,net,web3 --ws --ws.api eth,net,web3 --authrpc.jwtsecret jwt.hex --datadir gethdata3 --nodiscover --syncmode full --discovery.port 30305 --port 30305 --http.port 8560 --ws.port 8561 --authrpc.port 8562 --bootnodes *insert enode here*
```
Wait for log output to say:
```bash
WARN [01-09|18:38:58.164] Post-merge network, but no beacon client seen. Please launch one to follow the chain!
```
## Setup for 3rd prysm node 
Make sure to run the following commands in the `mod-prysm/` directory
```bash
cd prysm
go build -o=../beacon-chain ./cmd/beacon-chain
go build -o=../validator ./cmd/validator
go build -o=../prysmctl ./cmd/prysmctl
cd ..
```
## Starting the 3rd prysm node
In a new `screen`, run the following command with the `enr` (from `Setup for 2nd node`) inserted as the parameter for `bootstrap-node`. We will make sure that we start the node from a checkpoint:
```bash
 ./beacon-chain --datadir beacondata3 --min-sync-peers 1 --genesis-state ../devnet/genesis.ssz --bootstrap-node *INSERT ENR HERE* --interop-eth1data-votes --chain-config-file ../devnet/config.yml --contract-deployment-block 0 --chain-id 32382 --accept-terms-of-use --jwt-secret ../devnet/jwt.hex --suggested-fee-recipient 0x123463a4B065722E99115D6c222f267d9cABb524 --minimum-peers-per-subnet 0 --enable-debug-rpc-endpoints --execution-endpoint ../devnet/gethdata3/geth.ipc --peer=$PEER --p2p-udp-port 12002 --p2p-tcp-port 13002 --grpc-gateway-port 3502 --rpc-port 4002 --checkpoint-sync-url http://localhost:3500
```