# To listen on blocks from the orderer (Deliver)

## Method 1 (using the peer cli):
```
peer channel fetch <newest|oldest|config|<block_number>> -o <orderer_ip:orderer_port> -c <channel_name>
```
Please run `peer channel fetch --help` for additional info on the command.

## Method 2 (using the sample deliver client):
In `orderer/sample_clients/deliver_stdout`:
```
go run client.go -channelID mychannel
```
Please refer the file to figure out the other flags.

# To send transactions to the orderer (Deliver)

## Config transactions
```
peer channel update -f <config_update_envelope.pb> -o <orderer_ip:orderer_port> -c <channel_name>
```
Please run `peer channel update --help` for additional info on the commands.

## Regular transactions
In `orderer/sample_clients/broadcast_msg`:
```
go run client.go -channelID mychannel
```
Please refer the file to figure out the other flags.

# Process for creating a channel

# Process for updating a configuration (not channel creation)

## BatchSize updation
We demonstrate the process for BatchSize updation. Many of the steps will actually be similar for all types of configuration updates.

#### Params:
mychannel 	- assumed name of the channel for which configuration has to be updated.
127.0.0.1:7050 	- orderer endpoint 

#### Steps:
0. Export FABRIC_CFG_PATH
```
export FABRIC_CFG_PATH=~/go/src/github.com/hyperledger/fabric/sampleconfig/
```

1. Fetch the latest config for the channel. 
```
peer channel fetch config -o 127.0.0.1:7050 -c mychannel config_block.pb
```

2. Convert the fetched block from protobuf to json using `configtxlator`.
```
configtxlator proto_decode --input config_block.pb --type common.Block --output config_block.json 
```

3. Extract just the config section from the fetched config block for convenience.
```
jq .data.data[0].payload.data.config config_block.json > config.json
```

4. Export the BatchSize path in the above json for convenience.
```
export BASTCHSIZEPATH=".channel_group.groups.Orderer.values.BatchSize.value.max_message_count"
```

5. Update the json using `jq` to set the `max_message_count` to be `20` from `10` to obtain the modified json.
```
jq "$MAXBATCHSIZEPATH = 20" config.json > modified_config.json
```

6. Convert both the jsons back to protobuf so that the `configtxlator` can compute the update. 
```
configtxlator proto_encode --input config.json --type common.Config --output config.pb
configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
```

7. Compute the update using `configtxlator`. 
```
configtxlator compute_update --channel_id mychannel --original config.pb --updated modified_config.pb --output batchsize_update.pb
```

8. Convert the above update protobuf back to json so that we can easily wrap it into an envelope.
```
configtxlator proto_decode --input batchsize_update.pb --type common.ConfigUpdate --output batchsize_update.json
```

9. Wrap the update json in a CONFIG_UPDATE transaction, which will be in the json form.
```
echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat batchsize_update.json)'}}}' | jq . > batchsize_update_in_envelope.json
```
NOTE: `.payload.header.channel_header.type == 2` signifies that it is a CONFIG_UPDATE.

10. Convert the above CONFIG_UPDATE transaction in json to protobuf using `configtxlar`.
```
configtxlator proto_encode --input batchsize_update_in_envelope.json --type common.Envelope --output batchsize_update_in_envelope.pb
```

11. Submit the config update to the orderer using the `peer` cli. 
```
peer channel update -f batchsize_update_in_envelope.pb -o 127.0.0.1:7050 -c mychannel
```
