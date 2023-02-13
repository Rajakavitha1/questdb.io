---
title: MindsDB
description: Guide for enabling machine learning in QuestDB with MindsDB
---

[MindsDB](https://mindsdb.com/) enables Machine Learning capabilities to answer
predictive questions about users' data. With MindsDB:

- Developers can quickly add AI capabilities to their applications.
- Data scientists can streamline MLOps by deploying ML models as AI Tables.
- Data analysts can easily make forecasts on complex data, such as multivariate
  time-series with high cardinality, and visualize these in BI tools like
  Grafana, and Tableau.

Combining both MindsDB and QuestDB provides unbound prediction ability with SQL.

This guide describe how to pre-process data in QuestDB and then access these
data from MindsDB to produce powerful ML models.

## Prerequsites

- [docker-compose](https://docs.docker.com/compose/install/): To define and run
  our multi-container Docker application (it is usually installed implicitly
  when Docker is installed).
- [MySQL](https://dev.mysql.com/doc/refman/8.0/en/mysql.html): The client we
  will use to interact with MindsDB (mysql -h 127.0.0.1 --port 47335 -u mindsdb
  -p).
- [Curl](https://curl.se/download.html): To upload data to QuestDB from a local
  CSV file.

## Instructions

The following are the overall steps to connect the two databases:

1. Spawn two Docker containers to run MindsDB and QuestDB.
1. Add QuestDB as a datasource to MindsDB using a SQL Statement.
1. Create a table and add data for a simple ML use case using QuestDB's web
   console.
1. Connect to MindsDB using mysql client and write some SQL.

### Running the multi-container Docker application

Use the following
[docker-compose file](https://github.com/questdb/mindsdb-tutorial/blob/main/docker-compose.yaml)
to start the database instances:

```yaml
version: "3.8"

services:
  questdb:
    image: questdb/questdb:latest
    container_name: questdb
    pull_policy: "always"
    restart: "always"
    ports:
      - "8812:8812"
      - "9000:9000"
      - "9009:9009"
    volumes:
      - ./qdb_root:/root/.questdb

  mindsdb:
    image: mindsdb/mindsdb:latest
    container_name: mindsdb
    restart: "always"
    ports:
      - "47334:47334"
      - "47335:47335"
    volumes:
      - .:/root
    depends_on:
      - questdb

networks:
  default:
    name: mindsdb-network
    driver: bridge
```

The above file creates a local folder `qdb_root` to store table data/metadata
withthe default server configuration at `localhost:9000`. For MindsDB, it
creates two local folders, `mindsdb_store` and `nltk_data` with a configuration
file `mindsdb_config.json`.

Run docker-compose with the following command:

```shell
docker-compose up -d
```

MindsDB takes about 60-90 seconds to become available, logs can be followed in
the terminal:

```shell
docker logs -f mindsdb
...
Version 22.3.1.0
Configuration file:
   /root/mindsdb_config.json
Storage path:
   /root/mindsdb_store
http API: starting...
mysql API: starting...
mongodb API: starting...
 ✓ telemetry enabled
 ✓ telemetry enabled
 ✓ telemetry enabled
mongodb API: started on 47336
mysql API: started on 47335
http API: started on 47334
```

### Adding data to QuestDB

There are different ways to
[insert data to QuestDB](https://questdb.io/docs/develop/insert-data/):

### SQL

We can access QuestDB's web console at [localhost:9000](http://localhost:9000/):

![Screenshot showing the QuestDB web console](/img/docs/guide/questdb-web-console.png)

Run the following SQL query to create a simple table:

```questdb-sql
CREATE TABLE IF NOT EXISTS house_rentals_data (
    number_of_rooms INT,
    number_of_bathrooms INT,
    sqft INT,
    location SYMBOL,
    days_on_market INT,
    initial_price FLOAT,
    neighborhood SYMBOL,
    rental_price FLOAT,
    ts TIMESTAMP
) TIMESTAMP(ts) PARTITION BY YEAR;
```

We could populate table house_rentals_data with random data:

```questdb-sql
INSERT INTO house_rentals_data SELECT * FROM (
    SELECT
        rnd_int(1,6,0),
        rnd_int(1,3,0),
        rnd_int(180,2000,0),
        rnd_symbol('great', 'good', 'poor'),
        rnd_int(1,20,0),
        rnd_float(0) * 1000,
        rnd_symbol('alcatraz_ave', 'berkeley_hills', 'downtown', 'south_side', 'thowsand_oaks', 'westbrae'),
        rnd_float(0) * 1000 + 500,
        timestamp_sequence(
            to_timestamp('2021-01-01', 'yyyy-MM-dd'),
            14400000000L
        )
    FROM long_sequence(100)
);
```

#### CURL command

We can upload data from a local
[CSV file](https://github.com/questdb/mindsdb-tutorial/blob/main/sample_house_rentals_data.csv)
to QuestDB:

```shell
curl -F data=@sample_house_rentals_data.csv "http://localhost:9000/imp?forceHeader=true&name=house_rentals_data"
```

Either way, this gives us 100 data points, one every 4 hours, from
`2021-01-16T12:00:00.000000Z` (QuestDB's timestamps are UTC with microsecond
precision).

### Connect to MindsDB

We can connect to MindsDB with a standard mysql-wire-protocol compliant client
(no password, hit ENTER):

```shell
mysql -h 127.0.0.1 --port 47335 -u mindsdb -p
```

Only two databases are relevant to us, questdb and mindsdb:

```sql

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mindsdb            |
| files              |
| views              |
| questdb            |
+--------------------+
5 rows in set (0.34 sec)
```

To see questdb as a database we need to add it:

```sql

mysql> USE mindsdb;
Database changed

mysql>
CREATE DATASOURCE questdb
    WITH ENGINE = "questdb",
    PARAMETERS = {
        "user": "admin",
        "password": "quest",
        "host": "questdb",
        "port": "8812",
        "database": "questdb",
        "public": true
    };
```

#### `questdb`

This is a read-only view on our QuestDB instance. We can query it leveraging the
full power of QuestDB's unique SQL syntax because statements are sent from
MindsDB to QuestDB without interpreting them. It only works for SELECT
statements (it requires activation by means of `USE questdb;`):

```sql
mysql> USE questdb;
```

```shell
Database changed
```

```sql
mysql>
SELECT
    ts,
    neighborhood,
    sum(days_on_market) DaysLive,
    min(rental_price) MinRent,
    max(rental_price) MaxRent,
    avg(rental_price) AvgRent
FROM house_rentals_data
WHERE ts BETWEEN '2021-01-08' AND '2021-01-10'
SAMPLE BY 1d FILL (0, 0, 0, 0);
```

```shell
+--------------+----------------+----------+----------+----------+--------------------+
| ts           | neighborhood   | DaysLive | MinRent  | MaxRent  | AvgRent            |
+--------------+----------------+----------+----------+----------+--------------------+
| 1610064000.0 | south_side     | 19       | 1285.338 | 1285.338 | 1285.338134765625  |
| 1610064000.0 | downtown       | 7        | 1047.14  | 1047.14  | 1047.1396484375    |
| 1610064000.0 | berkeley_hills | 17       | 727.52   | 727.52   | 727.5198974609375  |
| 1610064000.0 | westbrae       | 36       | 1038.358 | 1047.342 | 1042.85009765625   |
| 1610064000.0 | thowsand_oaks  | 5        | 1067.319 | 1067.319 | 1067.318603515625  |
| 1610064000.0 | alcatraz_ave   | 0        | 0.0      | 0.0      | 0.0                |
| 1610150400.0 | south_side     | 10       | 694.403  | 694.403  | 694.4031982421875  |
| 1610150400.0 | downtown       | 16       | 546.798  | 643.204  | 595.0011291503906  |
| 1610150400.0 | berkeley_hills | 4        | 1256.49  | 1256.49  | 1256.4903564453125 |
| 1610150400.0 | westbrae       | 0        | 0.0      | 0.0      | 0.0                |
| 1610150400.0 | thowsand_oaks  | 0        | 0.0      | 0.0      | 0.0                |
| 1610150400.0 | alcatraz_ave   | 14       | 653.924  | 1250.477 | 952.2005004882812  |
| 1610236800.0 | south_side     | 0        | 0.0      | 0.0      | 0.0                |
| 1610236800.0 | downtown       | 9        | 1357.916 | 1357.916 | 1357.9158935546875 |
| 1610236800.0 | berkeley_hills | 0        | 0.0      | 0.0      | 0.0                |
| 1610236800.0 | westbrae       | 0        | 0.0      | 0.0      | 0.0                |
| 1610236800.0 | thowsand_oaks  | 0        | 0.0      | 0.0      | 0.0                |
| 1610236800.0 | alcatraz_ave   | 0        | 0.0      | 0.0      | 0.0                |
+--------------+----------------+----------+----------+----------+--------------------+
18 rows in set (0.18 sec)

```

Beyond SELECT statements, for instance when we need to save the results of a
query into a new table, we need to use QuestDB's web console available at
[localhost:9000](http://localhost:9000/):

```questdb-sql

CREATE TABLE sample_query_results AS (
    SELECT
        ts,
        neighborhood,
        sum(days_on_market) DaysLive,
        min(rental_price) MinRent,
        max(rental_price) MaxRent,
        avg(rental_price) AvgRent
    FROM house_rentals_data
    WHERE ts BETWEEN '2021-01-08' AND '2021-01-10'
    SAMPLE BY 1d FILL (0, 0, 0, 0)
) TIMESTAMP(ts) PARTITION BY MONTH;
```

#### `mindsdb`

Contains the metadata tables necessary to create ML models and add new data
sources:

```sql
mysql> USE mindsdb;
```

```shell
Database changed
```

```sql
mysql> SHOW TABLES;
```

```shell
+-------------------+
| Tables_in_mindsdb |
+-------------------+
| predictors        |
| commands          |
| datasources       |
+-------------------+
3 rows in set (0.17 sec)
```

```sql
mysql> SELECT * FROM datasources;
```

```shell
+---------+---------------+---------+------+-------+
| name    | database_type | host    | port | user  |
+---------+---------------+---------+------+-------+
| questdb | questdb       | questdb | 8812 | admin |
+---------+---------------+---------+------+-------+
1 row in set (0.19 sec)
```

### Create a predictor

We can create a predictor model `mindsdb.home_rentals_model_ts` to predict the
`rental_price` for a neighborhood considering the past 20 days, and no
additional features:

```sql

mysql> USE mindsdb;
Database changed

mysql>
CREATE PREDICTOR mindsdb.home_rentals_model_ts FROM questdb (
    SELECT
        neighborhood,
        rental_price,
        ts
    FROM house_rentals_data
)
PREDICT rental_price ORDER BY ts GROUP BY neighborhood
WINDOW 20 HORIZON 1;
```

This triggers MindsDB to create/train the model based on the full data available
from QuestDB's table `house_rentals_data` (100 rows) as a time series on the
column `ts`.

When status is complete, the model is ready for use; otherwise, we simply wait
while we observe MindsDB's logs, and repeat the query periodically.
Creating/training a model will take time proportional to the number of features,
i.e. cardinality of the source table as defined in the inner SELECT of the
CREATE PREDICTOR statement, and the size of the corpus, i.e. number of rows. The
model is a table in MindsDB:

```sql
mysql> SHOW TABLES;
```

```shell
+-----------------------+
| Tables_in_mindsdb     |
+-----------------------+
| home_rentals_model_ts |
| predictors            |
| commands              |
| datasources           |
+-----------------------+
4 rows in set (0.21 sec)
```

### Describe the predictor

We can get more information about the trained model, how was the accuracy
calculated or which columns are important for the model by executing the
`DESCRIBE` statement:

```sql
mysql> DESCRIBE home_rentals_model_ts;
```

```
*************************** 1. row ***************************
        accuracies: {'evaluate_num_array_accuracy': 1.429527527262832}
column_importances: {}
           outputs: ['rental_price']
            inputs: ['neighborhood', 'ts', '__mdb_ts_previous_rental_price']
        datasource: home_rentals_model_ts
             model: encoders --> dtype_dict --> dependency_dict --> model --> problem_definition --> identifiers --> accuracy_functions
1 row in set (0.119 sec)
```

Or, to see how the model encoded the data prior to training we can execute:

```sql
mysql> DESCRIBE home_rentals_model_ts.features;
```

```shell
+--------------+-------------+------------------+---------+
| column       | type        | encoder          | role    |
+--------------+-------------+------------------+---------+
| neighborhood | categorical | OneHotEncoder    | feature |
| rental_price | float       | TsNumericEncoder | target  |
| ts           | datetime    | ArrayEncoder     | feature |
+--------------+-------------+------------------+---------+
3 rows in set (0.077 sec)
```

Additional information about the models and how they can be customized can be
found on the [Lightwood docs](https://lightwood.io/).

### Query MindsDB for predictions

The latest `rental_price` value per neighborhood in table
`questdb.house_rentals_data` can be obtained directly from QuestDB executing
query:

```sql
mysql> USE questdb;
Database changed

mysql>
SELECT
    neighborhood,
    rental_price,
    ts
FROM house_rentals_data
LATEST BY neighborhood;
```

```shell
+----------------+--------------+--------------+
| neighborhood   | rental_price | ts           |
+----------------+--------------+--------------+
| thowsand_oaks  | 1150.427     | 1610712000.0 |   (2021-01-15 12:00:00.0)
| south_side     | 726.953      | 1610784000.0 |   (2021-01-16 08:00:00.0)
| downtown       | 568.73       | 1610798400.0 |   (2021-01-16 12:00:00.0)
| westbrae       | 543.83       | 1610841600.0 |   (2021-01-17 00:00:00.0)
| berkeley_hills | 559.928      | 1610870400.0 |   (2021-01-17 08:00:00.0)
| alcatraz_ave   | 1268.529     | 1610884800.0 |   (2021-01-17 12:00:00.0)
+----------------+--------------+--------------+
6 rows in set (0.13 sec)
```

To predict the next value:

```sql

mysql> USE mindsdb;
Database changed

mysql>
SELECT
    tb.ts,
    tb.neighborhood,
    tb.rental_price as predicted_rental_price,
    tb.rental_price_explain as explanation
FROM questdb.house_rentals_data AS ta
JOIN mindsdb.home_rentals_model_ts AS tb
WHERE ta.ts > LATEST;
```

```shwll
+---------------------+----------------+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ts                  | neighborhood   | predicted_rental_price | explanation                                                                                                                                                                              |
+---------------------+----------------+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 2021-01-17 00:00:00 | downtown       |      877.3007391233444 | {"predicted_value": 877.3007391233444, "confidence": 0.9991, "anomaly": null, "truth": null, "confidence_lower_bound": 379.43294697022424, "confidence_upper_bound": 1375.1685312764646} |
| 2021-01-19 08:00:00 | westbrae       |      923.1387395936794 | {"predicted_value": 923.1387395936794, "confidence": 0.9991, "anomaly": null, "truth": null, "confidence_lower_bound": 385.8327438509463, "confidence_upper_bound": 1460.4447353364124}  |
| 2021-01-15 16:00:00 | thowsand_oaks  |      1418.678199780345 | {"predicted_value": 1418.678199780345, "confidence": 0.9991, "anomaly": null, "truth": null, "confidence_lower_bound": 1335.4600013965369, "confidence_upper_bound": 1501.8963981641532} |
| 2021-01-17 12:00:00 | berkeley_hills |      646.5979284300436 | {"predicted_value": 646.5979284300436, "confidence": 0.9991, "anomaly": null, "truth": null, "confidence_lower_bound": 303.253838410034, "confidence_upper_bound": 989.9420184500532}    |
| 2021-01-18 12:00:00 | south_side     |       1422.69481363723 | {"predicted_value": 1422.69481363723, "confidence": 0.9991, "anomaly": null, "truth": null, "confidence_lower_bound": 129.97617491441304, "confidence_upper_bound": 2715.413452360047}   |
| 2021-01-18 04:00:00 | alcatraz_ave   |      1305.009073065412 | {"predicted_value": 1305.009073065412, "confidence": 0.9991, "anomaly": null, "truth": null, "confidence_lower_bound": 879.0232742685288, "confidence_upper_bound": 1730.994871862295}   |
+---------------------+----------------+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

### Stop the containers and remove the persisted data

To terminate the containers, run:

```shell
docker-compose down
```

To remove all persisted data and configuration executing, run:

```
./remove_persisted_data.sh
```

:::note

Doing this means that the next time you start the containers you will need to
add QuestDB as a datasource again, as well as recreate the table, add data, and
recreate your ML models.

:::

## See also

- [MindsDB GitHub](https://github.com/mindsdb/mindsdb)
- [MindsDB Documentation](https://docs.mindsdb.com/)