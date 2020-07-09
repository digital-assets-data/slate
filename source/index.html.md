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
If you are already comfortable with GraphQL you can review additional documentation and test your connection and your queries using our <a href="/schema-explorer" target="_blank">GraphQL Explorer</a>.

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
pair | No | N/A | The standardized pair/symbol, e.g. `BTC/USD` |
startDate | No | 10 minutes ago | Can't be more than 1 hour apart from endDate
endDate | No | Now | Can't be more than 1 hour apart from startDate
startTradeId | No | N/A | Min. Trade ID (where the exchange provides a numeric ID)
endTradeId | No | N/A | Max. Trade ID (where the exchange provides a numeric ID)

The Trade ID parameters are recommended for filling in missing records, not for general use. They will not work for exchanges that use IDs formatted as strings instead of integers. Date filters must also be applied in order to use the trade ID filters.

If dates are not specified, the endpoint will return all trades within the previous 10 minutes by default.
<br><br>
Dates must be formatted per the ISO 8601 standard (e.g.: 2020-01-10T17:49:26Z)
<br><br>
<aside class="notice">
The record limit for the trades endpoint is 10,000 trades per request
</aside>

## Prices / Candles

Historical prices and OHLCV data are available through this endpoint. You can specify the exchange, pairs, and granularity of the prices.
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
pair | No | N/A | The standardized pair/symbol, e.g. `BTC/USD` |
windowType | No | `tumble_01m` | The granularity and methodology of the price
startTime | No | Now | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date

### Window Types

The window type must be specified in order to retrieve prices from the API. The window type determines the methodology, granularity, and update frequency of the prices.
<br><br>
"Tumble" indicates that the price includes all trades from a given window (e.g. 1 minute), and then the next price includes all trades from the following window with no overlap. Each price is thus completely distinct.
<br><br>
The default is `tumble_01m`, but a list of options is provided below. This list may be expanded in the future.

Window Type | Description 
--------- | -------
`tumble_01m` | Minutely prices
`tumble_05m` | 5 minutely prices
`tumble_15m` | 15 minutely prices
`tumble_01h` | Hourly prices
`tumble_01d` | Daily prices

<aside class="notice">
The record limit for the prices endpoint is 100,000
</aside>

## Order Book Metrics

Digital Assets Data calculates a variety of metrics based on order book snapshots, such as spread, depth, resiliency, and slippage. These metrics are calculated at a variety of different levels based on percentages or the number of price levels.

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    exchange(dadExchangeId: "coinbase") {
      orderBookMetrics(metricName:"10%_asks_depth",pair: "BTC/USD", first: 1) {
        edges {
          node {
            id
            timestamp
            baseAssetId
            quoteAssetId
            pair
            metricName
            metricType
            metricValue
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
        "orderBookMetrics": {
          "edges": [
            {
              "node": {
                "id": "10%_asks_depth__coinbase__Bitcoin_BTC_BTC__USDollar_USD_USD",
                "timestamp": "2020-07-08T20:35:07.421000",
                "baseAssetId": "Bitcoin_BTC_BTC",
                "quoteAssetId": "USDollar_USD_USD",
                "pair": "BTC/USD",
                "metricName": "10%_asks_depth",
                "metricType": "depth",
                "metricValue": 2866.75202654
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
dadExchangeId | Yes | N/A | The DAD exchange key, e.g. `bitmex` |
metricName | Yes | N/A | The name of the metric being requested
pair | No | N/A | The standardized symbol, e.g. `XBTUSD` |
startTime | No | 10 minutes ago | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date

### Metric List

The current and complete list of metrics can be found in the Data Explorer. Here is a sample of the metrics that are available:

- `2%_asks_depth`
- `2%_asks_depth_usd_converted`
- `2%_asks_orderbook_resiliency`
- `2%_asks_orderbook_resiliency_usd_converted`
- `5%_bids_depth`
- `5%_bids_depth_usd_converted`
- `5%_bids_orderbook_resiliency`
- `5%_bids_orderbook_resiliency_usd_converted`
- `10%_midpoint_asks_depth`
- `10%_midpoint_asks_depth_usd_converted`
- `10%_midpoint_asks_orderbook_resiliency`
- `10%_midpoint_asks_orderbook_resiliency_usd_converted`
- `10%_midpoint_bids_depth`
- `10%_midpoint_bids_depth_usd_converted`
- `10%_midpoint_bids_orderbook_resiliency`
- `10%_midpoint_bids_orderbook_resiliency_usd_converted`
- `dollar_slippage_10000_asks`
- `dollar_slippage_10000_bids`
- `dollar_slippage_bps_10000_asks`
- `dollar_slippage_bps_10000_bids`
- `metric_spread_quote`
- `microprice_quote`
- `microprice_usd_converted`
- `spread_usd`

## Derivatives: Instruments

The instruments endpoint can be used to pull relevant metadata about derivative instruments, including their IDs, types, and subtypes.

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    exchange (dadExchangeId:"deribit") {
      instruments (first:1) {
        edges {
          node {
            id
            timestamp
            marketplace
            instrumentStatus
            instrumentId
            instrumentName
            underlyingInstrument
            baseAssetId
            quoteAssetId
            expirationTimestampUtc
            settlementTimestampUtc
            strikePrice
            derivativeType
            derivativeSubtype
            contractSize
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
        "instruments": {
          "edges": [
            {
              "node": {
                "id": "deribit_BTC-25SEP20-24000-C__1585333262753",
                "timestamp": "2020-03-27T18:21:02.753000",
                "marketplace": "deribit",
                "instrumentStatus": "active",
                "instrumentId": "deribit_BTC-25SEP20-24000-C",
                "instrumentName": "BTC-25SEP20-24000-C",
                "underlyingInstrument": null,
                "baseAssetId": "BTC",
                "quoteAssetId": "USD",
                "expirationTimestampUtc": "2020-09-25T08:00:00",
                "settlementTimestampUtc": null,
                "strikePrice": 24000,
                "derivativeType": "option",
                "derivativeSubtype": "call",
                "contractSize": 1
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
dadExchangeId | Yes | N/A | The DAD exchange key, e.g. `deribit` |
instrumentId | No | N/A | The standardized symbol, e.g. `deribit_ETH-26JUN20-100-C__1585333151219` |
startTime | No | 10 minutes ago | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date
derivativeType | No | N/A | The type of derivative, such as `future`, `option`, or `perpetual_swap`

## Derivatives: Instrument Aggregates

The instrument aggregates endpoint includes significant aggregations like price, volume, implied volatility, and open interest.

These aggregations are calculated minutely for now, but may be calculated at various frequencies in the future. The frequency can be determined from the `pollingInterval`, which is displayed in milliseconds. So a pollingInterval of 60000 equates to 1 minute.

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    exchange (dadExchangeId:"deribit") {
      instrumentAggregates (first:1) {
        edges {
          node {
            id
            timestamp
            pollingInterval
            marketplace
            instrumentId
            instrumentName
            openInterestContractCount
            volume24hContractCount
            volume24hInBase
            volume24hInUsd
            liquidationVolume24hr
            volatilityImplied
            volatilityImpliedMark
            baseAssetId
            quoteAssetId
            strikePrice
            price
            priceOpen
            priceHigh
            priceLow
            priceClose
            priceHigh24h
            priceLow24h
            tradeCount
            volumeContractCount
            volumeInBase
            volumeUsd
            amount
            markPrice
            derivativeType
            derivativeSubtype
            underlyingAssetId
            underlyingPrice
            fundingRate8h
            fundingRateCurrent
            optionDelta
            optionGamma
            optionRho
            optionTheta
            optionVega
            bestBidPrice
            bestBidAmount
            bestAskPrice
            bestAskAmount
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
        "instrumentAggregates": {
          "edges": [
            {
              "node": {
                "id": "deribit_ETH-26JUN20-100-C__1585333151219",
                "timestamp": "2020-03-27T18:19:11.219000",
                "pollingInterval": 60000,
                "marketplace": "deribit",
                "instrumentId": "deribit_ETH-26JUN20-100-C",
                "instrumentName": "ETH-26JUN20-100-C",
                "openInterestContractCount": 5,
                "volume24hContractCount": null,
                "volume24hInBase": 0,
                "volume24hInUsd": null,
                "liquidationVolume24hr": null,
                "volatilityImplied": null,
                "volatilityImpliedMark": 119.57,
                "baseAssetId": "ETH",
                "quoteAssetId": "USD",
                "strikePrice": 100,
                "price": 0.337,
                "priceOpen": 0,
                "priceHigh": 0,
                "priceLow": 0,
                "priceClose": 0,
                "priceHigh24h": 0,
                "priceLow24h": 0,
                "tradeCount": null,
                "volumeContractCount": null,
                "volumeInBase": 0,
                "volumeUsd": null,
                "amount": null,
                "markPrice": 0.356971,
                "derivativeType": "option",
                "derivativeSubtype": "call",
                "underlyingAssetId": "ETH-26JUN20",
                "underlyingPrice": 134.74,
                "fundingRate8h": 0,
                "fundingRateCurrent": 0,
                "optionDelta": 0.78769,
                "optionGamma": 0.00361,
                "optionRho": 0.14401,
                "optionTheta": -0.12851,
                "optionVega": 0.19468,
                "bestBidPrice": 0.3215,
                "bestBidAmount": 58,
                "bestAskPrice": 0.3795,
                "bestAskAmount": 58
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
dadExchangeId | Yes | N/A | The DAD exchange key, e.g. `deribit` |
instrumentId | No | N/A | The standardized symbol, e.g. `deribit_ETH-26JUN20-100-C__1585333151219` |
startTime | No | 10 minutes ago | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date
derivativeType | No | N/A | The type of derivative, such as `future`, `option`, or `perpetual_swap`

## Derivatives: Trades

The derivatives trades endpoint can be used to pull individual trades from supported derivative markets.

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    exchange(dadExchangeId:"bitmex") {
      tradesDerivatives(instrumentName:"XBTUSD", first:1) {
        edges {
          node{
            id
            timestamp
            instrumentName
            instrumentId
            derivativeType
            eventId
            isSell
            priceInQuote
            size
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
        "tradesDerivatives": {
          "edges": [
            {
              "node": {
                "id": "XBTUSD__bitmex__77c7218a-4fe9-7348-78b9-16b59405e4ce",
                "timestamp": "2020-07-07T20:56:24.648000",
                "instrumentName": "XBTUSD",
                "instrumentId": "bitmex_XBTUSD",
                "derivativeType": "swap",
                "eventId": null,
                "isSell": true,
                "priceInQuote": 9236,
                "size": 3
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
dadExchangeId | Yes | N/A | The DAD exchange key, e.g. `bitmex` |
instrumentName | No | N/A | The standardized symbol, e.g. `XBTUSD` |
startTime | No | 10 minutes ago | ISO 8601 standard date
endTime | No | Now | ISO 8601 standard date

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

## Blockchain UTXO Age Bands

You can use this endpoint to pull daily UTXO age band data based on Digital Assets Data's default age bands. The `reportDate` corresponds to the date in the blockchain's history and `ageBandPercent` shows the percentage of UTXO volume within the relevant age band. 

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    blockchain(dadBlockchainId: "btc") {
      utxoDailyAgeBands (first:1) {
        edges {
          node {
            id
            projectId
            assetId
            reportDate
            ageBand
            ageBandVolume
            ageBandPercent
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
        "utxoDailyAgeBands": {
          "edges": [
            {
              "node": {
                "id": "utxo_daily_age_bands__Bitcoin_BTC_BTC__1583798400000",
                "projectId": "Bitcoin_BTC",
                "assetId": "Bitcoin_BTC_BTC",
                "reportDate": "2020-03-10T00:00:00",
                "ageBand": "18-24m",
                "ageBandVolume": 30589.00795745,
                "ageBandPercent": 0.006703168568080371
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

### Age Bands

The default age bands used by Digital Assets Data are listed below.

Age Band | Description 
--------- | -------
`<1d` | Less than 24 hours
`1d-1w` | 1 day to 1 week
`1w-1m` | 1 week to 1 month
`1-3m` | 1 to 3 months
`3-6m` | 3 to 6 months
`6-12m` | 6 to 12 months
`12-18m` | 12 to 18 months
`18-24m` | 18 to 24 months
`2-3y` | 2 to 3 years
`3-5y` | 3 to 5 years
`>5y` | More than 5 years

## Blockchain UTXO Summaries

You can use this endpoint to pull the daily UTXO summaries underlying our age band data. This data could be used to calculate custom age bands.

The `reportTime` corresponds to the date in the blockchain's history, and `utxoCreatedTime` corresponds to the date the UTXOs were created. `utxoVolume` and `utxoCount` show the number of distinct UTXOs and their volume that were created on that date.

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    blockchain(dadBlockchainId: "btc") {
      utxoDailySummary (first:1) {
        edges {
          node {
            id
            projectId
            assetId
            reportTime
            utxoCreatedTime
            utxoCreatedEpochTime
            utxoVolume
            utxoCount
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
        "utxoDailySummary": {
          "edges": [
            {
              "node": {
                "id": "utxo_daily_summary__Bitcoin_BTC_BTC__1583884800000__1583798400000",
                "projectId": "Bitcoin_BTC",
                "assetId": "Bitcoin_BTC_BTC",
                "reportTime": "2020-03-11T00:00:00",
                "utxoCreatedTime": "2020-03-10T00:00:00",
                "utxoCreatedEpochTime": 1583798400000,
                "utxoVolume": 229861.19601658,
                "utxoCount": 384951
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


## Blockchain Daily Active Addresses

You can use this endpoint to pull daily stats on the number of active addresses for UTXO-based chains.

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    blockchain(dadBlockchainId: "btc") {
      addressSummary (first:1) {
        edges {
          node {
            id
            activeAddresses
            newAddresses
            totalAddressCount
            assetId
            projectId
            timestamp
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
        "addressSummary": {
          "edges": [
            {
              "node": {
                "id": "daily_address_summary__1583798400000__Bitcoin_BTC_BTC",
                "activeAddresses": 826647,
                "newAddresses": 429371,
                "totalAddressCount": 636211263,
                "assetId": "Bitcoin_BTC_BTC",
                "projectId": "Bitcoin_BTC",
                "timestamp": "2020-03-10T00:00:00"
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

## Blockchain Miner's Rolling Inventory

Miner's Rolling Inventory is a metric that measures the extent to which miners are selling vs retaining the coins that they are generating/earning. We calculate Miner's Rolling Inventory daily over several different intervals: 1 day, 21 days, and 42 days. This endpoint also shows the total inventory still held by miners over each blockchain's history.

> Sample request:

```graphql
{
  apiViewer(publicKey: "foo", privateKey: "bar") {
    blockchain(dadBlockchainId: "btc") {
      minerInventoryDaily (first:1) {
        edges {
          node {
            id
            timestamp
            projectId
            assetId
            dailyMri
            rolling21DayMri
            rolling42DayMri
            firstSpend
            generated
            unspentInventory
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
        "minerInventoryDaily": {
          "edges": [
            {
              "node": {
                "id": "miner_inventory__Bitcoin_BTC__1585440000000",
                "timestamp": "2020-03-29T00:00:00",
                "projectId": "Bitcoin_BTC",
                "assetId": "Bitcoin_BTC_BTC",
                "dailyMri": 90.32193745820581,
                "rolling21DayMri": 104.4700141839271,
                "rolling42DayMri": 102.62442152253325,
                "firstSpend": 1654.9668572299997,
                "generated": 1832.29778258,
                "unspentInventory": 1761104.563763461
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

# Websockets

To connect to websockets, you will need to include your public and private keys as query parameters.

A sample URL is shown below.

`wss://api.digitalassetsdata.com/exchange/coinbase?publicKey={foo}&privateKey={bar}`

Websockets are currently enabled for each exchange, so you can subscribe to a different websocket by swapping out `coinbase` for the key of any other supported exchange.

## Trades

Trades can be pushed to you through the `exchange` websocket endpoint. That can be accessed from URLs formatted like this:

`wss://api.digitalassetsdata.com/exchange/coinbase?publicKey={foo}&privateKey={bar}`

We've included an example here of a message that would subscribe to the trades channel for BTC/USD.

>Sample message:

```json
{"action": "subscribe", "channels": ["trades"], "pairs": ["BTC/USD"]}
```

> Sample response:

```json
{"trade": {"volume": 0.011304879561066628, "price": 8077.43017578125, "dadExchangeId": "coinbase", "pair": "BTC/USD", "tradeId": 80911105, "timestamp": 1578687219514}}
```

The websocket will push all new trades on an ongoing basis. The websocket will not push historical trades.

### Message Parameters
Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
action | Yes | N/A | ping or subscribe
channels | Yes | trades |  
pair | Yes | BTC/USD | The standardized pair/symbol

## Derivative Trades

Derivative trades can be pushed to you through the same `exchange` websocket endpoint. That can be accessed from URLs formatted like this:

`wss://api.digitalassetsdata.com/exchange/bitmex?publicKey={foo}&privateKey={bar}`

>Sample message:

```json
{"action": "subscribe", "channels": ["trades_derivatives"], "instruments": ["XBTUSD"]}
```

> Sample response:

```json
{"trade": {"instrumentId": "bitmex_XBTUSD", "priceInQuote": 9432.5, "eventId": "6b0cd966-ea5e-9cbf-89a8-0091e13a16e5", "size": 50000.0, "isSell": 0, "derivativeType": "swap", "instrumentName": "XBTUSD", "timestamp": 1594244866243}, "exchange": "bitmex"}
```

The websocket will push all new trades on an ongoing basis. The websocket will not push historical trades.

### Message Parameters
Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
action | Yes | N/A | ping or subscribe
channels | Yes | trades_derivatives |  
instruments | Yes | N/A | A list of the standardized derivative symbols


## Prices / Candles

Prices can be pushed to you through the `prices` websocket endpoint. That can be accessed from this URL:

`wss://api.digitalassetsdata.com/prices?publicKey={foo}&privateKey={bar}`

We've included an example here of a message that would subscribe to 5 secondly prices for BTC/USD on Coinbase.

>Sample message:

```json
{"action": "subscribe", "exchanges": ["coinbase"], "pairs": ["BTC/USD"], "windows": ["tumble_05s"]}
```

> Sample response:

```json
{"volume": 9.2951021194458, "price": 9360.0, "pair": "BTC/USD", "timestamp": 1582673495000, "dadExchangeId": "UNIFIED", "windowType": "tumble_05s", "periodStart": 1582673495000, "periodEnd": 1582673500000, "open": 9359.23046875, "high": 9361.873046875, "low": 9358.7900390625, "close": 9361.07421875, "quoteVolume": 87002.1484375, "tradeCount": 118}
```

The websocket will push all newly generated prices that match your parameters on an ongoing basis, and may push a limited amount of historical prices on subscription.

### Message Parameters

Parameter | Required | Default | Description
--------- | ------- | -------- | -----------
action | Yes | N/A | ping or subscribe  
pairs | Yes | BTC/USD | List of standardized pairs/symbols
exchanges | No | UNIFIED | List of exchanges
windows | No | tumble_05s | List of desired price windows/methodologies

Window Type | Description 
--------- | -------
`tumble_01m` | Minutely prices
`tumble_05m` | 5 minutely prices
`tumble_15m` | 15 minutely prices
`tumble_01h` | Hourly prices
`tumble_01d` | Daily prices

# Change Log
The Digital Assets Data API uses the following error codes:

Date | Updates
---------- | -------
7/8/2020 | - Added Order Book Metrics and Derivative Trades endpoints to the GraphQL API<br> - Added Derivative Trades to the websocket<br> - Updated price documentation to reflect currently supported windows


# Errors
The Digital Assets Data API uses the following error codes:

Error Code | Meaning
---------- | -------
400 | Bad Request -- Your request is invalid.
404 | Not Found -- The endpoint could not be found.
500 | Internal Server Error -- We had a problem with our server. Try again later.
503 | Service Unavailable -- We're temporarily offline for maintenance. Please try again later.
