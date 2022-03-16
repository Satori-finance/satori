---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - python
  - javascript

includes:
#  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for Satori Perpetual Protocol
---

# Introduction

Welcome to the docs for the Satori Perpetual Protocol. Satori is a decentralized financial derivatives platform built on Polkadot. It features a hybrid of orderbook and AMM models to provide comprehensive exposure to a wide range of assets, and an “off-chain aggregation and on-chain settlement” design that combines the security and transparency of a decentralized exchange, with the speed and usability of a centralized exchange.

# Perpetuals

## Margin

Satori enforces margin requirements for users -- an initial margin requirement to open and size-up positions, and a maintenance margin requirement to avoid liquidations. All positions are set to isolated margin, which means margin is assigned by specific positions. Margin can be added and removed at will above the initial margin.

### Initial Margin

The maximum supported leverage is 20x, so least 5% of the position value must be used as collateral upon opening a new position.

For example, with $100 of collateral, the maximum position size that can be put on is:

| Leverage | Position Size |
| -------- | ------------- |
| 5x       | $500          |
| 10x      | $1,000        |
| 20x      | $2,000        |

### Maintenance Margin

If the value of a position falls below the maintenance margin level, it may be automatically closed out by the liquidation engine.

The maintenance margin rates are determined on a per-token-pair basis:

| Token | Maintenance Margin Rate |
| ----- | ----------------------- |
| BTC   | 0.5%                    |
| ETH   | 0.5%                    |
| DOT   | 0.5%                    |

For example, a long leveraged position in BTC-USDT may be liquidated after the mark price dips below 0.5% above the bankruptcy price.

All collateral is held as DOT, and the denominated asset for all perpetual markets is USDT.

## Liquidations

Positions where the margin rate is less than or equal to the maintenance margin rate may be liquidated.

### Closing Price

The closing price is determined by:

<aside class="formula">
  <code>
    Closing Price (long) = Average Opening Price * (1 + Maintenance Margin Rate) - (Position Margin / Number of Contracts)
    Closing Price (short) = Average Opening Price * (1 - Maintenance Margin Rate) + (Position Margin / Number of Contracts)
  </code>
</aside>

If the position is liquidated at a price better than the bankruptcy price, then the funds will be added to the insurance vault. Otherwise, if the position is liquidated at a worse price, then the insurance vault funds will be used to cover the loss due to the position. If that insurance vault is insufficient, then the [auto-deleveraging system](#automatic-deleveraging-system) will reduce the positions of investors that have positions in the opposite direction.

### Insurance Vault

When a position is liquidated, it will be taken over by the forced liquidation system. If the liquidation cannot be fulfilled by the time the perpetual reaches the bankruptcy price, the loss will be covered by the insurance vault, a reserve maintained by the platform.

The insurance vault is currently seeded and maintained by the Satori team, and will eventually be funded through trading fees.

### Automatic Deleveraging System

In the event that the insurance vault is insufficient to cover a liquidation loss, the Automatic Deleveraging System (ADL) will automatically deleverage traders holding positions in the opposite direction.

The ADL will deleverage traders in descending order of profit and leverage, i.e. the higher the profit and the more leverage used the higher the ranking.

Traders can view their priority ranking for automatic position reduction via the “ADL Ranking” indicator in the UI. Each light represents a 20% priority ranking, and when all lights are on, the position may be reduced in the event of a forced liquidation.

Positions will be reduced atomically: after the first user's position in line has been fully reduced, the second position will continue to be reduced and so on.

### ADL ranking calculation method

<aside class="formula">
    <code>
    Ranking =
            % profit * effective leverage (if profit, i.e. % profit > 0)

            OR

            % profit / effective leverage (if loss, i.e. % profit < 0)
    </code>

</aside>

Where:

- `Effective leverage` is `mark value / (mark value - bankruptcy value)`
- `% profit` is `(mark value - average open value) / average open value`
- `Mark value` is the value of the position at the mark price
- `Bankruptcy value` is the value of position at the bankruptcy price
- `Average open value` is the the value of position at average open price

## Funding Costs

We anchor the price of perpetual contracts to the spot index price through a funding fee mechanism. Long positions pay short positions when the future is trading at a premium; short positions pay long positions when the future is trading below. The platform does not receive any of the fees.

The funding fee is charged every hour, and calculated as follows:

<aside class="formula">
  Funding fee = position size * oracle price * funding fee rate
</aside>

### Funding Fee Rate Calculation

The funding fee rate is comprised of two components: the interest rate and the premium index.

The interest rate is currently set to `0.03%` per day.

The premium index is calculated for each market, per minute (at random points in that minute) using the following formula:

<aside class="formula">
  Premium Index = (max(0, B - I) - max(0, I - A)) / I
</aside>

Where:

- `B` is the `impact bid price`, or the average execution price to close out a long position of impact notional value through a market sell
- `A` is the `impact ask price`, or the average execution price to close out a short position of impact notional value through a market buy
- `I` is the spot index price
- `Impact Notional Value` is set to 800 USDT / Initial Margin Ratio of the Market

The total premium index for an hour is then the average of all premium indices calculated for that hour.

Altogether, the funding fee rate is calculated as:

<aside class="formula">
    Funding Fee Rate = Premium Index + clamp(Interest Rate/24 - Premium Index, 0.05%, -0.05%)
</aside>

In other words, the funding fee ate will equal the hourly interest rate if the premium index is within 0.05% of the hourly interest rate.

### Funding Fee Rate Cap

The absolute value of the Funding Fee Rate is capped at 0.075%.

## Index Price

The `Index Price` refers to the price of the underlying asset on the spot market. It's an aggregate price based off price data from multiple exchanges.

The exchanges we aggregate from are as follows for each market:

#### BTC-USDT

`Okex`
`Binance`
`Huobi`

#### ETH-USDT

`Okex`
`Binance`
`Huobi`

#### DOT-USDT

`Okex`
`Binance`
`Huobi`

We have implemented logic to ensure that index fluctuations are within the normal range when there is a significant deviation from the price of a single exchange.

If an exchange price deviates by more than 3% relative to the median price of all exchanges, the weight that exchange is given will be reduced.

## Oracle Price

The `Oracle Price` is an aggregate price calculated using multiple on-chain price oracles. Oracle prices are used to determine collateralization and liquidations on Satori.

Since the oracle price is also an aggregate price, it offers similar protection from flash crashes as the index price does.

For the Alpha launch, Satori runs its own oracle nodes on Layer 2.

## Matchmaking

Orders will be aggregated by price-time on a FIFO basis. A transaction happens if there is a buy order that has a price equal or greater than a sell order, or if there is a market order placed in one direction and orders exist in the opposing direction

<!-- ```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
```

```shell
# With shell, you can just pass the correct header with each request
curl "api_endpoint_here" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require("kittn");

let api = kittn.authorize("meowmeowmeow");
```

> Make sure to replace `meowmeowmeow` with your API key.

Kittn uses API keys to allow access to the API. You can register a new Kittn API key at our [developer portal](http://example.com/developers).

Kittn expects for the API key to be included in all API requests to the server in a header that looks like the following:

`Authorization: meowmeowmeow`

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

# Kittens

## Get All Kittens

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require("kittn");

let api = kittn.authorize("meowmeowmeow");
let kittens = api.kittens.get();
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

| Parameter    | Default | Description                                                                      |
| ------------ | ------- | -------------------------------------------------------------------------------- |
| include_cats | false   | If set to true, the result will also include cats.                               |
| available    | true    | If set to false, the result will include kittens that have already been adopted. |

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require("kittn");

let api = kittn.authorize("meowmeowmeow");
let max = api.kittens.get(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

| Parameter | Description                      |
| --------- | -------------------------------- |
| ID        | The ID of the kitten to retrieve |

## Delete a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.delete(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.delete(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -X DELETE \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require("kittn");

let api = kittn.authorize("meowmeowmeow");
let max = api.kittens.delete(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted": ":("
}
```

This endpoint deletes a specific kitten.

### HTTP Request

`DELETE http://example.com/kittens/<ID>`

### URL Parameters

| Parameter | Description                    |
| --------- | ------------------------------ |
| ID        | The ID of the kitten to delete | -->
