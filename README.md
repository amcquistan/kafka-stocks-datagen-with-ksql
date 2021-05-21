# Stock Streaming Demo with Confluent Datagen and ksqlDB

This demo uses ksql-datagen (or sometimes referred to as kafka-connect-datagen) Kafka Connector to create fake stock data which is then consumed by ksqlDB.

### Goals

1) Produce fake stock data for two fictitious companies (ACME and HOOLI) to a Kafka topic where stock quotes are random doubles choosen between 100-200

2) Find average stock price over a 1 minute tumbling window and filter down to those that are over 170 and tag with a SELL action or under 130 and tag with BUY action

3) Dump the filtered 1 minute stock quote windows identified in step 2 to a topic back stream so that a consumer could then react to those buy/sell actions

### Process

1) Clone Repo


```
git clone https://github.com/amcquistan/kafka-stocks-datagen-with-ksql.git
```

2) Fire Up Docker Compose Service

```
cd kafka-stocks-datagen-with-ksql
docker-compose up -d
```

3) Create topic in Kafka

```
docker exec -it broker kafka-topics \
	--bootstrap-server localhost:9092 --create --topic stocks
```

4) Add stocks-datagen Connector

Show Avro Schema and Data Generator Config.

Schema: stocks.avsc

```
{
  "type": "record",
  "name": "stockquote",
  "fields": [
    {
      "name": "symbol",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "ACME",
            "HOOLI"
          ]
        }
      }
    },
    {
      "name": "quote",
      "type": {
        "type": "double",
        "arg.properties": {
          "range": {
            "min": 100,
            "max": 200
          }
        }
      }
    }
  ]
}
```

Connector config: stocks-data-config.json

```
{
  "name": "stocks-datagen",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "stocks",
    "schema.string": "{\"type\":\"record\",\"name\":\"stockquote\",\"fields\":[{\"name\":\"symbol\",\"type\":{\"type\":\"string\",\"arg.properties\":{\"options\":[\"ACME\",\"HOOLI\"]}}},{\"name\":\"quote\",\"type\":{\"type\":\"double\",\"arg.properties\":{\"range\":{\"min\":100,\"max\":200}}}}]}",
    "schema.keyfield": "symbol",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter.schemas.enable": "false",
    "max.interval": 2000,
    "iterations": 10000
  }
}


```

I prefer [HTTPie](https://httpie.io/) HTTP CLI client over CURL so, all REST requests will be shown with HTTPie client

```
http POST http://localhost:8083/connectors @stocks-data-config.json
```

5) Verify Connector was added and Running

Is it added?

```
http http://localhost:8083/connectors
```

Should see this output.

```
HTTP/1.1 200 OK
Content-Length: 18
Content-Type: application/json
Date: Fri, 21 May 2021 19:33:53 GMT
Server: Jetty(9.4.33.v20201020)

[
    "stocks-datagen"
]
```

Is it running?

```
http http://localhost:8083/connectors/stocks-datagen/status
```

Should see this output.

```
HTTP/1.1 200 OK
Content-Length: 164
Content-Type: application/json
Date: Fri, 21 May 2021 19:34:00 GMT
Server: Jetty(9.4.33.v20201020)

{
    "connector": {
        "state": "RUNNING",
        "worker_id": "connect:8083"
    },
    "name": "stocks-datagen",
    "tasks": [
        {
            "id": 0,
            "state": "RUNNING",
            "worker_id": "connect:8083"
        }
    ],
    "type": "source"
}
```

6) Fire Up KSQL Shell

```
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

7) Create stocks\_stream on stocks Topic in KSQL

```
CREATE STREAM stocks_stream_avro (
  symbol VARCHAR,
  quote DOUBLE
) WITH (KAFKA_TOPIC='stocks', VALUE_FORMAT='avro');
```

8) Create stocks\_1min\_windows\_tbl Aggregate Table in KSQL

```
CREATE TABLE stocks_1min_windows_tbl 
WITH (KAFKA_TOPIC='stocks_1min_windows_tbl') AS
SELECT
  symbol,
  AVG(quote) avg_1min_quote,
  CASE
    WHEN AVG(quote) > 170 THEN 'SELL'
    WHEN AVG(quote) < 130 THEN 'BUY'
  END action
FROM stocks_stream_avro
WINDOW TUMBLING ( SIZE 1 MINUTE )
GROUP BY symbol
HAVING AVG(quote) > 170 OR AVG(quote) < 130
EMIT CHANGES;
```

9) Create stocks_buysell\_interim\_stream Stream in KSQL

```
CREATE STREAM stocks_buysell_interim_stream (
  symbol VARCHAR KEY,
  avg_1min_quote DOUBLE,
  action VARCHAR
) WITH (KAFKA_TOPIC='stocks_1min_windows_tbl', VALUE_FORMAT='avro');
```

10) Create final stocks\_buysell\_stream Stream in KSQL

```
CREATE STREAM stocks_buysell_stream
WITH (KAFKA_TOPIC='stocks_buysell_stream') AS
SELECT * FROM stocks_buysell_interim_stream
WHERE symbol IS NOT NULL
PARTITION BY symbol
EMIT CHANGES;
```

11) Start kafka-avro-console-consumer in KSQL

This would ideally be a consumer (in Java/Python) subscribed to the stocks\_buysell\_stream topic and initiating some response (actual loading up or off loading stocks).

```
docker exec -it broker kafka-avro-console-consumer --bootstrap-server localhost:9092 \
  --topic stocks --from-beginning \
  --property schema.registry.url=http://localhost:8081
```
