# confluent

## Table of Contents

* [JDBC Sink](#jdbc-sink)
  * [Topology](#topology)
  * [Usage](#usage)
    * [New record](#new-record)
    * [Record update](#record-update)
    * [Record delete](#record-delete)

## JDBC Sink

### Topology

```
                   +-------------+
                   |             |
                   |    MySQL    |
                   |     CDC     |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |  (Debezium, JDBC connectors)     |
          |                                  |
          +---------------+------------------+
                          |
                          |
                          |
                          |
                  +-------v--------+
                  |                |
                  |   PostgreSQL   |
                  |                |
                  +----------------+


```
We are using Docker Compose to deploy following components
* MySQL
* Kafka
  * ZooKeeper
  * Kafka Broker
  * Kafka Connect with [Debezium](https://debezium.io/) and  [JDBC](https://github.com/confluentinc/kafka-connect-jdbc) Connectors
* PostgreSQL

### Usage

How to run:

```shell
docker-compose up -d

# see control-center on http://localhost:9021/


# Start PostgreSQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @jdbc-sink.json

# Start MySQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source.json
```

Check contents of the MySQL database:

```shell
docker-compose -f docker-compose.yaml exec mysql bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory -e "select * from customers"'
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne       | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
```

Verify that the PostgreSQL database has the same content:

```shell
docker-compose exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
 Thomas    | 1001 | Sally      | sally.thomas@acme.com
 Bailey    | 1002 | George     | gbailey@foobar.com
 Walker    | 1003 | Edward     | ed@walker.com
 Kretchmar | 1004 | Anne       | annek@noanswer.org
(4 rows)
```

#### New record

Insert a new record into MySQL;
```shell
docker-compose  exec mysql bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory'
mysql> insert into customers values(default, 'John', 'Doe', 'john.doe@example.com');
Query OK, 1 row affected (0.02 sec)
```

Verify that PostgreSQL contains the new record:

```shell
docker-compose exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
...
Doe        | 1005 | John       | john.doe@example.com
(5 rows)
```

#### Record update

Update a record in MySQL:

```shell
mysql> update customers set first_name='Jane', last_name='Roe' where last_name='Doe';
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

Verify that record in PostgreSQL is updated:

```shell
docker-compose exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
...
Roe        | 1005 | Jane       | john.doe@example.com
(5 rows)
```

#### Record delete

Delete a record in MySQL:

```shell
mysql> delete from customers where email='john.doe@example.com';
Query OK, 1 row affected (0.01 sec)
```

Verify that record in PostgreSQL is deleted:

```shell
docker-compose  exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
...
(4 rows)
```

As you can see there is no longer a 'Jane Doe' as a customer.


End application:

```shell
# Shut down the cluster
docker-compose down
