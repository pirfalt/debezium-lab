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

### Local pubsub

First test the local pubsub.

```sh
git submodule add https://github.com/googleapis/nodejs-pubsub.git
# If you are following along, this will fail since it's already in the repo.
# Use this instead:
#  git submodule update --init

cd nodejs-pubsub/samples
npm install

export PUBSUB_PROJECT_ID=project_id
export PUBSUB_EMULATOR_HOST=localhost:8085

# Create the topics
node listAllTopics.js
node createTopic.js
node listAllTopics.js
node deleteTopic.js
```

Should be working.

### Local postgres

This should work out of the box since it's pre configures.

But lets try it out.

```sh
# Run bash within the server container
docker exec -it debezium-lab_source-db_1 bash

# Connect a `psql` client to the server
psql postgres postgres
```

```sql
\dt inventory.
select * from inventory.customers;
select * from inventory.orders;
\q
```

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

Sadly the distribution does not support local stream endpoints.
Therefor we clone and patch the debezium project.

```sh
# Clone the upstream repo
git submodule add https://github.com/debezium/debezium.git
# If you are following along, this will fail since it's already in the repo.
# Use this instead:
#  git submodule update --init

cd debezium

# Patch to enable using a local stream endpoints
git apply ../0001-Add-support-for-local-kinesis-by-adding-endpoint-con.patch
git apply ../0001-Add-support-for-pubsub-emulator-by-adding-endpoint-c.patch

# Build
mvn install -DskipITs -DskipTests

# Replace the pre-packaged jar file with the newly built version
cp debezium-server/debezium-server-kinesis/target/debezium-server-kinesis-1.4.0-SNAPSHOT.jar ../debezium-server/lib/debezium-server-kinesis-1.3.0.Final.jar
cp debezium-server/debezium-server-pubsub/target/debezium-server-pubsub-1.4.0-SNAPSHOT.jar ../debezium-server/lib/debezium-server-pubsub-1.3.0.Final.jar
cd ..
```

### Test it

Once all the dependencies are verified to work we want to try it out.

In separate terminal windows, run the debezium-server and a stream client. Then make a change in the db.

### Kinesis

[Kinesis](./run_kinesis.md)

### Pubsub

[Pubsub](./run_pubsub.md)

### SQL

[SQL](./run_sql.md)
