# Debezium Lab

## Tutorial

https://debezium.io/
https://debezium.io/documentation/reference/1.1/tutorial.html

## Debezium Server

https://debezium.io/blog/
https://debezium.io/blog/2020/05/19/debezium-1-2-beta2-released/
https://debezium.io/documentation/reference/1.2/operations/debezium-server.html

## Source

https://github.com/debezium/debezium

## Command log

Commented history of what commands were run during the lab.

### Start

First of all, start up the local dependencies. Both a pre configured postgres server and a local kinesis server.

```sh
docker-compose up -d
```

### Local kinesis

First test the local kinesis.

```sh
# Helper function
awslocal() {
  aws --endpoint=http://localhost:4568 $@
}

# Create a stream
awslocal kinesis list-streams
# > {
# >     "StreamNames": []
# > }
awslocal kinesis create-stream --stream-name mystream --shard-count 1
awslocal kinesis list-streams
# > {
# >     "StreamNames": [
# >         "mystream"
# >     ]
# > }

# Put a record on the stream
awslocal kinesis put-record --stream-name mystream --partition-key 123 --data testdata
# > {
# >     "ShardId": "shardId-000000000000",
# >     "SequenceNumber": "49607460903051878213938622908891381945100083754734452738"
# > }

# Read the stream
## Create a shard iterator for the kinesis stream
SHARD_ITERATOR=$(
  awslocal kinesis get-shard-iterator \
    --shard-id shardId-000000000000 \
    --shard-iterator-type TRIM_HORIZON \
    --stream-name mystream \
    --query 'ShardIterator' \
    --output text
)
# Make sure you got a value for the iterator:
echo "$SHARD_ITERATOR"
# > AAAAAAAAAAGl5uxo+asOiIswDuwm+yjIqbGFkVuVw9dstmw7COJA/WEXmho5FU06cu+RdCBg1KEmM/YOhPbT4EqtNnTAsl0GVtVo0XLEDeaqTyB9yyGz3t5v53v2PGdFDggjtWVHwuMg+TIQkEA8M/VltmZJj/jBtJl3H64tuCS9IF8H+4eo4oOFvLh99TTaWY1LnjX+u10=

## Read using the shard iterator
awslocal kinesis get-records --shard-iterator "$SHARD_ITERATOR" | jq -r '.Records[].Data' | base64 --decode
# > testdata
```

If that all works we have a functioning local kinesis to use for testing.

### Local postgres

This should work out of the box since it's pre configures.

### debezeum-server

https://debezium.io/documentation/reference/1.2/operations/debezium-server.html

Follow instructions. Download to `./debezeum-server`.

#### Download dist

```sh
# Debezium server
curl -O https://repo1.maven.org/maven2/io/debezium/debezium-server-dist/1.3.0.Final/debezium-server-dist-1.3.0.Final.tar.gz
gunzip debezium-server-dist-1.3.0.Final.tar.gz
tar -xvf debezium-server-dist-1.3.0.Final.tar

# Common logging dependency
curl -O https://repo1.maven.org/maven2/commons-logging/commons-logging/1.2/commons-logging-1.2.jar
mv commons-logging-1.2.jar debezeum-server/lib
```

#### Git clone

```sh
# Clone the upstream repo
git submodule add https://github.com/debezium/debezium.git
cd debezium

# Patch to enable using a local kinesis
git apply ../0001-Add-support-for-local-kinesis-by-adding-endpoint-con.patch

# Build
mvn install -DskipITs -DskipTests

# Replace the pre-packaged jar file with the newly built version
cp debezium-server/debezium-server-kinesis/target/debezium-server-kinesis-1.4.0-SNAPSHOT.jar ../debezium-server/lib/debezium-server-kinesis-1.3.0.Final.jar
cd ..
```

#### Start

```sh
# Copy config
cp application.properties debezium-server/conf/

# Run, from the pre-packaged directory
# With cbor disabled, since the local kinesis does not support it
cd debezium-server
mkdir data
touch data/offsets.dat
JAVA_OPTS='-Daws.cborEnabled=false' ./run.sh
```

This will fail. Since there are no streams for the tables.

```sh
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

### Test it

Once all the dependencies are up and running we want to try it out.

In separate terminal windows, run a kinesis stream client and make a change in the db.

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

#### Postgres client

Use the `psql` client from within the server container.

```sh
# Run bash within the server container
docker exec -it debezium-lab_source-db_1 bash

# Connect a `psql` client to the server
psql postgres postgres
```

Some example updates.

```sql
INSERT INTO inventory.customers VALUES (default,'Emil','Pirfält','emil.pirfalt@jayway.com');
INSERT INTO inventory.customers VALUES (default,'Hans','Karlsson','hans.karlsson@jayway.com');
INSERT INTO inventory.customers VALUES (default,'Karl','Hedin Sånemyr','karl.hedin@jayway.com');
INSERT INTO inventory.customers VALUES (default,'Hugo','Hjertén','hugo.hjerten@jayway.com');
INSERT INTO inventory.customers VALUES (default,'Arvid','Huss','arvid.huss@jayway.com');
INSERT INTO inventory.customers VALUES (default,'Magnus','Kivi','magnus.kivi@jayway.com');

UPDATE inventory.customers SET email = 'emil.pirfalt@jayway.se' WHERE email = 'emil.pirfalt@jayway.com';
UPDATE inventory.customers SET email = 'hans.karlsson@jayway.se' WHERE email = 'hans.karlsson@jayway.com';
UPDATE inventory.customers SET email = 'karl.hedin@jayway.se' WHERE email = 'karl.hedin@jayway.com';
UPDATE inventory.customers SET email = 'hugo.hjerten@jayway.se' WHERE email = 'hugo.hjerten@jayway.com';
UPDATE inventory.customers SET email = 'arvid.huss@jayway.se' WHERE email = 'arvid.huss@jayway.com';
UPDATE inventory.customers SET email = 'magnus.kivi@jayway.se' WHERE email = 'magnus.kivi@jayway.com';

UPDATE inventory.customers SET email = 'emil.pirfalt@jayway.com' WHERE email = 'emil.pirfalt@jayway.se';
UPDATE inventory.customers SET email = 'hans.karlsson@jayway.com' WHERE email = 'hans.karlsson@jayway.se';
UPDATE inventory.customers SET email = 'karl.hedin@jayway.com' WHERE email = 'karl.hedin@jayway.se';
UPDATE inventory.customers SET email = 'hugo.hjerten@jayway.com' WHERE email = 'hugo.hjerten@jayway.se';
UPDATE inventory.customers SET email = 'arvid.huss@jayway.com' WHERE email = 'arvid.huss@jayway.se';
UPDATE inventory.customers SET email = 'magnus.kivi@jayway.com' WHERE email = 'magnus.kivi@jayway.se';

DELETE FROM inventory.customers WHERE email = 'emil.pirfalt@jayway.com';
DELETE FROM inventory.customers WHERE email = 'hans.karlsson@jayway.com';
DELETE FROM inventory.customers WHERE email = 'karl.hedin@jayway.com';
DELETE FROM inventory.customers WHERE email = 'hugo.hjerten@jayway.com';
DELETE FROM inventory.customers WHERE email = 'arvid.huss@jayway.com';
DELETE FROM inventory.customers WHERE email = 'magnus.kivi@jayway.com';
-- I guess delete needs to be fixed, otherwise 👍!
```
