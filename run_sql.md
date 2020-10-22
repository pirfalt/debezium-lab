# Debezium Lab

#### Postgres client

Use the `psql` client from within the server container to make some changes in the database, which should be published.

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
-- [x] Only an issue with Kinesis, pubsub is fine
```

#### "Manual" sync state introspection

```sql
SELECT * FROM pg_replication_slots;

-- Create a replication slot of our own
SELECT * FROM pg_create_logical_replication_slot('slot_repl', 'test_decoding');
SELECT * FROM pg_logical_slot_peek_changes('slot_repl', NULL, NULL);
SELECT * FROM pg_logical_slot_get_changes('slot_repl', NULL, NULL);

-- Check on the debezium replication slot
SELECT * FROM pg_logical_slot_peek_changes('debezium', NULL, NULL);
SELECT * FROM pg_logical_slot_peek_binary_changes('debezium', NULL, NULL);
```
