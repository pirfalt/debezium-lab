# Debezium Lab

#### Start server

```sh
# Copy config
cp application-kinesis.properties debezium-server/conf/application.properties

# Run, from the pre-packaged directory
# With cbor disabled, since the local kinesis does not support it
cd debezium-server
mkdir data
touch data/offsets.dat
JAVA_OPTS='-Daws.cborEnabled=false' ./run.sh
```

This will fail. Since there are no streams for the tables.

```sh
# Helper function
awslocal() {
  aws --endpoint=http://localhost:4568 $@
}

# Create the streams
awslocal kinesis create-stream --stream-name 'tutorial.inventory.customers' --shard-count 1
awslocal kinesis create-stream --stream-name 'tutorial.inventory.geom' --shard-count 1
awslocal kinesis create-stream --stream-name 'tutorial.inventory.orders' --shard-count 1
awslocal kinesis create-stream --stream-name 'tutorial.inventory.products' --shard-count 1
awslocal kinesis create-stream --stream-name 'tutorial.inventory.products_on_hand' --shard-count 1
awslocal kinesis create-stream --stream-name 'tutorial.inventory.spatial_ref_sys' --shard-count 1
awslocal kinesis list-streams
```

```sh
# Run again
JAVA_OPTS='-Daws.cborEnabled=false' ./run.sh
```

#### Kinesis client

A simple `aws-cli` and bash kinesis client.

```sh
# Make a function to poll the stream
poll_stream() {
  local STREAM_NAME="${1:?}"

  # Create an initial shard iterator for the kinesis stream
  local SHARD_ITERATOR=$(
    awslocal kinesis get-shard-iterator \
      --shard-id shardId-000000000000 \
      --shard-iterator-type TRIM_HORIZON \
      --stream-name "$STREAM_NAME" \
      --query 'ShardIterator' \
      --output text
  )

  # Poll until `ctrl-c`
  while true ; do
    # Get records from the shard iterator
    local RESPONSE=$(awslocal kinesis get-records --shard-iterator "$SHARD_ITERATOR")
    # Replace the shard iterator with the next one
    local SHARD_ITERATOR=$(echo "$RESPONSE" | jq -r '.NextShardIterator')

    # Extract only the .Data from the records, ignoring kinesis metadata
    echo "$RESPONSE" | jq -r '.Records[].Data' | while read LINE; do
      # Base64 decode and extract only the .payload, ignoring the schema information
      echo "$LINE" | base64 --decode | jq '.payload | {
        # Concatenate a subset of the source information
        # Print everything else as is
        source: (.source.db + "." + .source.schema + "." + .source.table),
        op,
        ts_ms,
        before,
        after,
        transaction
      }';
    done

    sleep 1
  done
}

poll_stream 'tutorial.inventory.customers'
```
