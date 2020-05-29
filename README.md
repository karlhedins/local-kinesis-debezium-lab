# Requirements
https://github.com/pypa/pipenv

python 3

docker, docker-compose

https://stedolan.github.io/jq/

# Install
pipenv install

duplicate the `.env-example` file, rename to `.env` and modify to your liking.

# Run localstack
activate the pipenv shell:
pipenv shell

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
make sure your aws cli is not connected to your prod account, so you dont deploy stuff by mistake:

check:
aws iam get-user

if using mock values from .env you should get:
```
An error occurred (InvalidClientTokenId) when calling the GetUser operation: The security token included in the request is invalid.
```
this confirms you're running the local aws client.


awslocal iam get-user

Connection was closed before we received a valid response from endpoint URL: "http://localhost:4593/".

# Kinesis
list kinesis streams:
awslocal kinesis list-streams

should return:
```
{
    "StreamNames": []
}
```

## create kinesis stream

https://docs.aws.amazon.com/streams/latest/dev/fundamental-stream.html

awslocal kinesis create-stream --stream-name mystream --shard-count 1

list kinesis streams:
awslocal kinesis list-streams

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

awslocal kinesis put-record --stream-name mystream --partition-key 123 --data testdata

## read the kinesis stream

Create a shard iterator for the kinesis stream:

SHARD_ITERATOR=$(awslocal kinesis get-shard-iterator --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON --stream-name mystream --query 'ShardIterator'  | tr -d '"')

Make sure you got a value for the iterator:
echo $SHARD_ITERATOR

awslocal kinesis get-records --shard-iterator $SHARD_ITERATOR | jq -r '.Records[].Data | @base64d'

should return:
```
testdata
```
