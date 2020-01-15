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

You will need to include your public and private keys in the header of the request. That will look like this:

`DAD-PUBLIC-KEY=foo`<br>
`DAD-PRIVATE-KEY=bar`

```python
import requests

api = dad(DAD-PUBLIC-KEY=foo, DAD-PRIVATE-KEY=bar)
```

Make sure to replace `foo` and `bar` with your public and private key, respectively. Your public key will never change, and if you lose your private key you can generate a new one anytime.

<aside class="notice">
You will need both a public and private key to authenticate.
</aside>

# GraphQL

GraphQL is a more flexible alternative to the traditional REST framework. If you are new to GraphQL, we recommend engaging with the introduction materials on their <a href="https://graphql.org/learn/" target="_blank">website</a>.

If you are already comfortable with GraphQL you can review additional documentation and test your connection and your queries using our <a href="https://api.digitalassetsdata.com/graphql/" target="_blank">GraphQL Explorer</a>.

While GraphQL provides a flexible query language for interacting with our APIs that will enable you to pull exactly what you need (and no more), we have documented the most common use cases with code snippets and example responses below. Feel free to test these code snippets yourself in the GraphQL Explorer.

## Supported Exchanges / Pairs

You can use the supported exchanges endpoint to determine which exchanges and pairs are currently supported through the API.

> Sample request:

```graphql
{ 
	apiViewer(publicKey:"foo=" privateKey:"bar") {
    supportedExchanges{
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

Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
exchange | Yes | coinbase | The DAD exchange key |


## Trades

You can pull individual trades on a per exchange/per pair basis. Historical trades are available for up to an hour at a time using the `startDate` and `endDate` parameters.

> Sample request:

```graphql
{ 
	apiViewer(publicKey:"foo" privateKey:"bar") {
    exchange(dadExchangeId:"coinbase") {
      trades(pair:"BTC/USD") {
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
startDate | No | {10 minutes ago} | Can't be more than 1 hour apart from endDate
endDate | No | {Now} | Can't be more than 1 hour apart from startDate
startTradeId | No | N/A | Min. Trade ID (where the exchange provides a numeric ID)
endTradeId | No | N/A | Max. Trade ID (where the exchange provides a numeric ID)

The Trade ID parameters are recommended for filling in missing records, not for general use. They will not work for exchanges that use string IDs instead of numeric IDs.

If dates are not specified, the endpoint will return all trades within the previous 10 minutes by default.

Dates must be formatted like `2020-01-10T17:49:26Z`

<aside class="notice">
The max distance between startDate and endDate is 1 hour.
</aside>

## Prices / Candles

Coming soon...

## Order Book Snapshots

Coming soon...

## Order Book Streams

Coming soon...

# Websockets

To connect to websockets, you will need your public and private key.

A sample URL is shown below.

`wss://api.digitalassetsdata.com/exchange/coinbase?publicKey={foo}&privateKey={bar}`

Websockets are currently enabled for each exchange, so you can subscribe to a different websocket by swapping out `coinbase` for the key of any other supported exchange.

## Trades

After connecting to the websocket for a given exchange, you can subscribe to the channel(s) of your choice. We've included an example here of a message that would subscribe to the trades channel for BTC/USD.

>Sample message:

```json
{"action": "subscribe", "channels": ["trades"], "pairs": ["BTC/USD"]}
```

> Sample response:

```json
{"trade": {"volume": 0.011304879561066628, "price": 8077.43017578125, "dadExchangeId": "coinbase", "pair": "BTC/USD", "tradeId": 80911105, "timestamp": 1578687219514}, "exchange": "coinbase"}
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
401 | Unauthorized -- Your API key is wrong.
404 | Not Found -- The endpoint could not be found.
500 | Internal Server Error -- We had a problem with our server. Try again later.
503 | Service Unavailable -- We're temporarily offline for maintenance. Please try again later.

