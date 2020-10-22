# Debezium Lab

#### Start server

```sh
# Copy config
cp application-pubsub.properties debezium-server/conf/application.properties

# Run, from the pre-packaged directory
# With cbor disabled, since the local kinesis does not support it
cd debezium-server
mkdir data
rm data/offsets.dat
touch data/offsets.dat

./run.sh
```

#### Pubsub client

First add the json printing listener.

```sh
cd nodejs-pubsub
git apply ../0001-Add-listener-which-prints-json.patch
```

```sh
cd samples

export PUBSUB_PROJECT_ID=project_id
export PUBSUB_EMULATOR_HOST=localhost:8085

# Create the topics
node createTopic.js 'tutorial.inventory.customers'
node createTopic.js 'tutorial.inventory.geom'
node createTopic.js 'tutorial.inventory.orders'
node createTopic.js 'tutorial.inventory.products'
node createTopic.js 'tutorial.inventory.products_on_hand'
node createTopic.js 'tutorial.inventory.spatial_ref_sys'
node listAllTopics.js

# Subscribe (only to customers for now)
node listSubscriptions.js
node createSubscription.js tutorial.inventory.customers inventory.customers
node listSubscriptions.js

# Try listen
node listenForMessages.js projects/project_id/subscriptions/inventory.customers # No response, quit it
node publishMessage.js 'tutorial.inventory.customers'
node listenForMessagesJSON.js projects/project_id/subscriptions/inventory.customers # as json
node publishMessage.js 'tutorial.inventory.customers'

node listenForMessagesJSON.js projects/project_id/subscriptions/inventory.customers $((60 * 10)) & # Run in background, for 10 min
jobs
kill

# Listen for "real"
node listenForMessagesJSON.js projects/project_id/subscriptions/inventory.customers | jq 'del(.data.schema)'
```
