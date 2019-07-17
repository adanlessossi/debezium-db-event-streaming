# MySQL Database Event Streaming with Debezium using multi-container deployment

Last time, I found out a very efficient way to stream database changes to a Kafka Broker.
The solution is using Debezium ([Debezium]: <http://debezium.io/>), a quite efficient way to stream events on database.

Our goals here are to:

- Turn your existing DB into event streams
- Record the history of data changes in kafka logs
- Crash resistent
- No missed records

## Getting Started

Follow the instructions in this README file to get you up and running on your local machine for development and testing purposes.

### Prerequisites

- Docker version 18.09.02 (Windows 10 Pro)
- docker-compose version 1.23.2, build 1110ad01
- docker-py version: 3.6.0
- CPython version: 3.6.6
- OpenSSL version: OpenSSL 1.0.2o 27 Mar 2018

## Running the Multi-container images

### 1. Starting Zookeeper container

First, you need to start Zookeeper using the debezium/zookeeper image and assign the name zookeeper to the container running on ports 2181, 2888 and 3888.

`docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:0.9`

Don't be intimidated if you see a bunch of logs in your console.

### 2. Starting Kafka container

Start Kafka using the debezium/kafka image and assign the name kafka to the container running on port 9092.
Finally link the zookeeper, so that it can find the zookeeper in the container named zookeeper on the same host.

`docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:0.9`

### 3. Starting MySQL Database container

Start a MySQL Database using the debezium/example-mysql image and assign the name mysql, passing environment variables to the container.

`docker run -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:0.9`

### 4. Starting MySQL command line client container

Start a MySQL command line client container, assign the name mysqlterm and connect it to the MySQL Server container.

`docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'`

---

After connection to the server,
a) switch to the inventory database.
`mysql> use inventory;`

    b) List the database tables
    `mysql> show tables;`

    c) Explore the data in the database
    `mysql> SELECT * FROM customers;`

---

### 5. Starting Kafka Connect container

Start Kafka Connect using the debezium/connect image and assign the name connect to the container running on port 8083.
The connect container is linked to zookeeper, kafka and the MySQL containers. Also, checkout the envionment variables required
by the debezium image.

`docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:0.9`

### 6. Using the Kafka Connect REST API

`curl -H "Accept:application/json" localhost:8083/`

You will see a response like this: `{"version":"2.2.0","commit":"cb8625948210849f"}`

You will also want to try out the connectors:
`curl -H "Accept:application/json" localhost:8083/connectors/`

### 7. MySQL Database Monitoring

You need to register a connector that will begin monitoring the database server binlog and generate change events for each row that has
been changed.
This command uses the Kafka Connect service’s RESTful API to submit a POST request against /connectors resource
with a JSON document that describes our new connector.

`curl -i -X POST -H "Accept:application/json" H "Content-Type:application/json" localhost:8083/connectors/ d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'`

Here, we want that:

    a) Exactly 1 Task should operate
    b) The database host is mysql, the name of the docker container.
    c) The MySQL port is specified
    d) The database has a debezium user set up.
    e) An unique server ID and name are given (logical identifier for the MySQL Server or cluster of servers)
    f) The connector should store the history of the database schema in kafka using the named broker and topic name.

---

Try verify that the connector is included in the list of connectors.
`curl -H "Accept:application/json" localhost:8083/connectors/`

To see the Task we need to query the state of the connector.

`curl -i -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector`

---

### 8. Starting a new Kafka container to watch the topic

Use the debezium/kafka image to start a new container that runs a kafka utility to watch the topic from the beginning.
The -k flag specifies that the output should include the event’s key, which in our case contains the row’s primary key

`docker run -it --name watcher --rm --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:0.9 watch-topic -a -k dbserver1.inventory.customers`

---

Now that we’re monitoring changes, lets see what happens when we manipulate the database.
In our terminal running watch-topic, we see our new events:

`mysql> SELECT * FROM customers;`
`UPDATE customers SET first_name='Anne Marie' WHERE id=1004;`
`mysql> DELETE FROM customers WHERE id=1004;`

---

### 9. Restart the Kafka Connect service

Just to fail over a little bit... and see the behavior after restarting.

`docker stop connect`

In the MySQL console of your choice, try changing/inserting some data.

`mysql> INSERT INTO customers VALUES (default, "Sarah", "Thompson", "kitt@acme.com");`
`mysql> INSERT INTO customers VALUES (default, "Kenneth", "Anderson", "kander@acme.com");`

Then, lets restart the Connect container.

`docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:0.9`

### 10. Exploration and clean up

You may need to run a separate watch-topic command for each topic.

But for cleaning up, please issue the command:
`docker stop mysqlterm watcher connect mysql kafka zookeeper`

## Built With

- [Docker](https://www.docker.com/) - The Docker container
- [Debezium](https://debezium.io/) - A yet effective database event streaming framework
- [Kafka](https://kafka.apache.org/) - An Open Source Stream Processing software platform
- [MySQL](https://www.mysql.com/) - An Open Source relational database management system (RDBMS)

## Authors

- **Bernard Adanlessossi**

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
