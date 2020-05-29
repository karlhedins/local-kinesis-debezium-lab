# Requirements
https://github.com/pypa/pipenv

python 3

docker, docker-compose

https://stedolan.github.io/jq/

# Install
```
pipenv install
```

duplicate the `.env-example` file, rename to `.env` and modify to your liking.

# Run localstack
activate the pipenv shell:
```
pipenv shell
```

spin up docker-compose:
docker-compose up -d

All aws services in localstack are exposed via a single edge service endpoint:
http://localhost:4566

Visiting that URL should give you
```
{
  "status": "running"
}
```

## aws
make sure your aws cli is not connected to your default aws account by mistake:

check:
```
aws iam get-user
```

if using mock values from .env you should get something like:
```
An error occurred (InvalidClientTokenId) when calling the GetUser operation: The security token included in the request is invalid.
```
this confirms you're running the local aws client.

# Kinesis
list kinesis streams:
```
awslocal kinesis list-streams
```

should return:
```
{
    "StreamNames": []
}
```

## create kinesis stream

https://docs.aws.amazon.com/streams/latest/dev/fundamental-stream.html

```
awslocal kinesis create-stream --stream-name mystream --shard-count 1
```

list kinesis streams:
```
awslocal kinesis list-streams
```

should return:
```
{
    "StreamNames": [
        "mystream"
    ]
}
```

<!-- awslocal kinesis describe-stream --stream-name mystream -->

## put kinesis record

```
awslocal kinesis put-record --stream-name mystream --partition-key 123 --data testdata
```

## read the kinesis stream

Create a shard iterator for the kinesis stream:

```
SHARD_ITERATOR=$(awslocal kinesis get-shard-iterator --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON --stream-name mystream --query 'ShardIterator'  | tr -d '"')
```

Make sure you got a value for the iterator:
```
echo $SHARD_ITERATOR
```

```
awslocal kinesis get-records --shard-iterator $SHARD_ITERATOR | jq -r '.Records[].Data | @base64d'
```

should return:
```
testdata
```

## debezium server

### download debezium server

Download the debezium server by following the instructions on:
https://debezium.io/documentation/reference/operations/debezium-server.html

At the time of writing this, it consisted of downloading the debezium-server tar file:
https://repo1.maven.org/maven2/io/debezium/debezium-server/1.2.0.Beta2/debezium-server-1.2.0.Beta2-distribution.tar.gz

### Note on Java
Use Java any way you prefer, but for reference this is my setup:
I'm running Java with jenv (https://www.jenv.be/), version 0.5.2.
The Java version is `openjdk64-1.8.0.232`
Activating a project local jenv with:

```
jenv local openjdk64-1.8.0.232
```

### Configure debezium server
Setup the configuration values for debezium-server to match your stup, the file can be found here:
`debezium-server/conf/application.properties`

This is how I configured my `application.properties`:

```
debezium.sink.type=kinesis
debezium.sink.kinesis.region=eu-west-1
debezium.sink.kinesis.region=http://localhost:4566
debezium.sink.kinesis.creadentials.profile=local
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.offset.storage.file.filename=data/offsets.dat
debezium.source.offset.flush.interval.ms=0
debezium.source.database.hostname=localhost
debezium.source.database.port=5432
debezium.source.database.user=postgres
debezium.source.database.password=postgres
debezium.source.database.dbname=postgres
debezium.source.database.server.name=tutorial
debezium.source.schema.whitelist=inventory
aws.cborEnable=false
```

### Running debezium server

Having unpacked the file, cd into the `debezium-server` directory.
Now let's install the `debezium-server` by running:

```
./run.sh
```

