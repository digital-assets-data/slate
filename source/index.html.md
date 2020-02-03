---
title: DAD API Reference
toc_footers:
  - <a href='https://app.digitalassetsdata.com/access-keys' target="_blank">Manage API Keys</a>
search: true
---
# Introduction
Welcome to the Digital Assets Data API documentation. This documentation provides detailed instructions for leveraging the DAD API. If you are looking to consume our market data outside of the platform, you have come to the right place.

If you need assistance consuming data from our APIs after reviewing this documentation, please reach out to us at <a href="mailto:support@digitalassetsdata.com" target="_blank">support@digitalassetsdata.com</a>.

We currently support two methods for retrieving data:

1. Batch requests (pull) through GraphQL
2. Continuous feeds (push) via websockets

Websockets are ideal for ingesting large volumes of data continuously on an ongoing basis. If you're looking to pull a specific subset of data, especially historical data, then you should use the GraphQL endpoints.

# Authentication

You will need to authenticate in order to access either the GraphQL endpoints or the websockets using public and private keys. You can retrieve your public key and generate a private key <a href="https://app.digitalassetsdata.com/access-keys" target="_blank">here</a>.

For websockets, you will need to include your public and private keys as query parameters. That will look like this:

`wss://api.digitalassetsdata.com/exchange/coinbase?publicKey={foo}&privateKey={bar}`

For GraphQL, the public and private keys will need to be included in the request headers. A sample code snippet for Python is shown to the right.

```python
import requests

headers = {
  'DAD-PUBLIC-KEY': 'foo',
  'DAD-PRIVATE-KEY': 'bar'
}
query = '''
  {
    apiViewer {
      username
    }
  }
'''
body = {
  'query': query
}

response = requests.post(
  'https://api.digitalassetsdata.com/graphql/',
  json=body,
  headers=headers
).json()
```

Make sure to replace `foo` and `bar` with your public and private key, respectively. Your public key will never change, and if you lose your private key you can generate a new one anytime.

<aside class="notice">
You will need both a public and private key to authenticate.
</aside>

# GraphQL
GraphQL is a more flexible alternative to the traditional REST framework. If you are new to GraphQL, we recommend engaging with the introduction materials on their <a href="https://graphql.org/learn/" target="_blank">website</a>.
If you are already comfortable with GraphQL you can review additional documentation and test your connection and your queries using our <a href="/graphql-explorer" target="_blank">GraphQL Explorer</a>.

> Sample connection test:

```graphql
{ 
	apiViewer(publicKey:"foo" privateKey:"bar") {
  	username
  }
}
```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "username": "example@digitalassetsdata.com"
    }
  }
}
```

You can test your connection by opening the GraphQL explorer and copying the code snippet to the right into the query editor in the left panel, swapping in your public and private keys, and then pressing the `play` button in the header. If the connection was successful, your username will be returned. If the connection was unsuccessful, you will get an error message. If that's the case, double check that you've entered your API keys correctly.

While GraphQL provides a flexible query language for interacting with our APIs that will enable you to pull exactly what you need (and no more), we have documented the most common use cases with code snippets and example responses below. Feel free to test these code snippets yourself in the GraphQL Explorer.

## Supported Exchanges / Pairs

You can use the supported exchanges endpoint to determine which exchanges and pairs are currently supported through the API.


Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
forPair | No | N/A | The standardized pair/symbol
dadExchangeId | No | N/A | The DAD exchange key

You can use the query parameters to filter by exchange or pair. This is useful when trying to determine which exchanges support a given pair, or you're interested in finding the supported symbols for a particular exchange.

In that case, you'd swap out the supportedExchanges line in the sample request to the right with this:

`supportedExchanges (dadExchangeId: "Coinbase") {`

The same concept applies when filtering by pair, like this:

`supportedExchanges (forPair: "BTC/USD") {`

<aside class="notice">
When filtering by pair, every supported pair for each exchange will be returned, however only exchanges that support the specific pair you entered will be returned.
</aside>

> Sample request:

```graphql
{ 
	apiViewer(publicKey:"foo=" privateKey:"bar") {
    supportedExchanges (dadExchangeId: "Coinbase") {
      dadExchangeId
      pairs
    }
  }
}

```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "supportedExchanges": [
        {
          "dadExchangeId": "kraken",
          "pairs": [
            "ADA/BTC",
            "BTC/USD",
          ]
        },
        {
          "dadExchangeId": "coinbase",
          "pairs": [
            "ALGO/USD",
            "BTC/USD"
          ]
        }
      ]
    }
  }
}

```

## Trades

You can pull individual trades on a per exchange/per pair basis. Historical trades are available for up to an hour at a time using the `startDate` and `endDate` parameters. You can use the parameters to cycle through historical trades and pull as much history as Digital Assets Data supports.

> Sample request:

```graphql
{ 
	apiViewer(publicKey:"foo" privateKey:"bar") {
    exchange(dadExchangeId:"coinbase") {
      trades(pair:"BTC/USD", first: 1) {
        edges {
          node {
            id
            dadExchangeId
            timestamp
            pair
            price
            volume
          }
        }
      }
    }
  }
}

```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "exchange": {
        "trades": {
          "edges": [
            {
              "node": {
                "id": "80910148",
                "dadExchangeId": "coinbase",
                "timestamp": "2020-01-10T19:54:28.237000",
                "pair": "BTC/USD",
                "price": 8068.8798828125,
                "volume": 0.014999999664723873
              }
            }
          ]
        }
      }
    }
  }
}

```
### Query Parameters

Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
exchange | Yes | N/A | The DAD exchange key, e.g. `coinbase` |
pair | Yes | N/A | The standardized pair/symbol, e.g. `BTC/USD` |
startDate | No | 10 minutes ago | Can't be more than 1 hour apart from endDate
endDate | No | Now | Can't be more than 1 hour apart from startDate
startTradeId | No | N/A | Min. Trade ID (where the exchange provides a numeric ID)
endTradeId | No | N/A | Max. Trade ID (where the exchange provides a numeric ID)

The Trade ID parameters are recommended for filling in missing records, not for general use. They will not work for exchanges that use IDs formatted as strings instead of integers.

If dates are not specified, the endpoint will return all trades within the previous 10 minutes by default.
<br><br>
Dates must be formatted per the ISO 8601 standard (e.g.: 2020-01-10T17:49:26Z)
<br><br>
<aside class="notice">
The record limit for the trades endpoint is 100,000
</aside>

## Prices / Candles

You can use this endpoint to pull individual blocks for a given blockchain back to the genesis block.

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    prices(dadExchangeId: "coinbase", pair: "BTC/USD", windowType: "tumble_01h", first: 1) {
      edges {
        node {
          periodStart
          periodEnd
          marketplace
          pair
          open
          high
          low
          close
          volume
          quoteVolume
          tradeCount
        }
      }
    }
  }
}
```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "prices": {
        "edges": [
          {
            "node": {
              "periodStart": "2020-01-31T17:00:00",
              "periodEnd": "2020-01-31T18:00:00+00:00",
              "marketplace": "coinbase",
              "pair": "BTC/USD",
              "open": 9271.650390625,
              "high": 9297.08984375,
              "low": 9260,
              "close": 9262,
              "volume": 448.23468017578125,
              "quoteVolume": 4157213.5,
              "tradeCount": 2497
            }
          }
        ]
      }
    }
  }
}

```

### Query Parameters

Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
dadExchangeId | No | N/A | The DAD exchange key, e.g. `coinbase` |
pair | Yes | N/A | The standardized pair/symbol, e.g. `BTC/USD` |
windowType | Yes | `tumble_01m` | The granularity and methodology of the price
startTime | No | Now | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date

### Window Types

The window type must be specified in order to retrieve prices from the API. The window type determines the methodology, granularity, and update frequency of the prices.
<br><br>
"Tumble" indicates that the price includes all trades from a given window (e.g. 1 minute), and then the next price includes all trades from the following window with no overlap. Each price is thus completely distinct.
<br><br>
"Sliding" means that there's overlap in the prices. For example, `sliding_05m_01m` indicates that the prices include trades over the previous 5 minutes, but the price is updated/calculated every minute. Thus, there's a smoothing effect on the prices as trades are included in multiple windows.
<br><br>
The default is `tumble_01m`, but a list of options is provided below. This list may be expanded in the future.

Window Type | Description 
--------- | -------
`tumble_01s` | Secondly prices
`tumble_05s` | 5 secondly prices
`tumble_01m` | Minutely prices
`tumble_05m` | 5 minutely prices
`tumble_15m` | 15 minutely prices
`tumble_01h` | Hourly prices
`tumble_04h` | 4 hourly prices
`sliding_05m_01m` | 5 minutely prices updated every minute
`sliding_15m_01m` | 15 minutely prices updated every minute
`sliding_01h_01m` | Hourly prices updated every minute
`sliding_04h_01h` | 4 hourly prices updated every hour

<aside class="notice">
The record limit for the blocks endpoint is 10,000
</aside>

## Order Book Snapshots

Coming soon...

## Order Book Streams

Coming soon...

## Supported Blockchains

You can use the supported blockchains endpoint to discover which blockchains are currently supported through this API. In the future, this endpoint will be expanded to also include the datasets for each blockchain. This will be particularly relevant for blockchains that weren't forked from BTC that have a different data schema or different datasets available (e.g. event logs for Ethereum).

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    supportedBlockchains {
      dadBlockchainId
    }
  }
}
```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "supportedBlockchains": [
        {
          "dadBlockchainId": "btc"
        },
        {
          "dadBlockchainId": "ltc"
        },
        {
          "dadBlockchainId": "dash"
        },
        {
          "dadBlockchainId": "zec"
        },
        {
          "dadBlockchainId": "zen"
        }
      ]
    }
  }
}

```


## Blockchain Blocks

You can use this endpoint to pull individual blocks for a given blockchain back to the genesis block.

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    blockchain(dadBlockchainId: "btc") {
      blocks(first: 1) {
        edges {
          node {
            timestamp
            blockHeight
            blockHash
            blockSize
            blockReward
            difficulty
            transactionCount
          }
        }
      }
    }
  }
}
```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "blockchain": {
        "blocks": {
          "edges": [
            {
              "node": {
                "timestamp": "2020-01-30T17:48:54",
                "blockHeight": 615247,
                "blockHash": "00000000000000000001725096df09c4dee4970e4cf0f59c574572209c967b90",
                "blockSize": 941855,
                "blockReward": 12.5,
                "difficulty": 15466098589696,
                "transactionCount": 2635
              }
            }
          ]
        }
      }
    }
  }
}

```


### Query Parameters

Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
blockHeight | No | N/A | The block height, aka block number |
blockHash | No | N/A | The block ID |
startTime | No | 24 hours ago | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date

<aside class="notice">
The record limit for the blocks endpoint is 10,000
</aside>

## Blockchain Transactions

You can use this endpoint to pull individual transactions for a given blockchain back to the genesis block.

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    blockchain(dadBlockchainId: "btc") {
      transactions(first: 1) {
        edges {
          node {
            timestamp
            transactionHash
            blockHeight
            transactionIndex
            outputVolume
            feeVolume
            isCoinbase
          }
        }
      }
    }
  }
}
```

> Sample response:

```json
{
  "data": {
    "apiViewer": {
      "blockchain": {
        "transactions": {
          "edges": [
            {
              "node": {
                "timestamp": "2020-01-31T19:44:39",
                "transactionHash": "697dd0932db6bc470db8bedf578ca962afa977cf1607b3f0965acd5b0a4258e3",
                "blockHeight": 615396,
                "transactionIndex": 379,
                "outputVolume": 0.2200581431388855,
                "feeVolume": 0.00010469999688211828,
                "isCoinbase": false
              }
            }
          ]
        }
      }
    }
  }
}

```


### Query Parameters

Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
startTime | No | N/A | ISO 8601 standard date |
endTime | No | N/A | ISO 8601 standard date |
blockHeight | No | 24 hours ago | The block number
transactionHash | No | Now | The transaction ID
blockHash | No | N/A | The block ID |
isCoinbase | No | 24 hours ago | Boolean for coinbase transactions
outputVolumeMin | No | Now | Minimum transaction size
outputVolumeMax | No | Now | Maximum transaction size

<aside class="notice">
The record limit for the transactions endpoint is 100,000
</aside>

# Websockets

To connect to websockets, you will need to include your public and private keys as query parameters.

A sample URL is shown below.

`wss://api.digitalassetsdata.com/exchange/coinbase?publicKey={foo}&privateKey={bar}`

Websockets are currently enabled for each exchange, so you can subscribe to a different websocket by swapping out `coinbase` for the key of any other supported exchange.

## Trades

After connecting to the websocket for a given exchange, you can subscribe to the channel(s) of your choice. We've included an example here of a message that would subscribe to the trades channel for BTC/USD.

The websocket will push all new trades on an ongoing basis. The websocket will not push historical trades.

>Sample message:

```json
{"action": "subscribe", "channels": ["trades"], "pairs": ["BTC/USD"]}
```

> Sample response:

```json
{"trade": {"volume": 0.011304879561066628, "price": 8077.43017578125, "dadExchangeId": "coinbase", "pair": "BTC/USD", "tradeId": 80911105, "timestamp": 1578687219514}}
```

### Message Parameters
Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
action | Yes | N/A | ping or subscribe
channels | Yes | trades |  
pair | Yes | BTC/USD | The standardized pair/symbol

## Prices / Candles
Coming soon...

## Order Book Snapshots
Coming soon...

## Order Book Streams
Coming soon...

# Errors
The Digital Assets Data API uses the following error codes:

Error Code | Meaning
---------- | -------
400 | Bad Request -- Your request is invalid.
404 | Not Found -- The endpoint could not be found.
500 | Internal Server Error -- We had a problem with our server. Try again later.
503 | Service Unavailable -- We're temporarily offline for maintenance. Please try again later.
