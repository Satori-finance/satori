---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - python
  - javascript

includes:
  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for the Kittn API
---

# Introduction

Welcome to the docs for the Satori Perpetual Protocol. Satori is a decentralized financial derivatives platform built on Polkadot. It features a hybrid of orderbook and AMM models to provide comprehensive exposure to a wide range of assets, and an “off-chain aggregation and on-chain settlement” design that combines the security and transparency of a decentralized exchange, with the speed and usability of a centralized exchange.

# Perpetual Contract Concepts

## Margin

### Early warning margin rate

It’s an early warning line for the margin rate of a user's position.
Setting: pre-set early warning margin rate = underlying maintenance margin rate \* 200%.

### Restrictions on opening positions

If the user margin rate is sufficient to open a position, available margin >= new opening cost\*105% is required to open a successful order, subject to the actual transaction price; (subject to existing positions).
When the market volatility >= 2%/minute, the user cannot open a position for 20 seconds; text message: the current market volatility is high, the risk is high, temporarily do not support opening a position.

### Transfer out restrictions

Users are not allowed to transfer out their account balance when the margin rate of their position <= the alert margin rate.
Transferable amount = available balance / 105%.

## Liquidations

Accounts where the margin rate is less than or equal to the maintenance margin rate may be liquidated.

When a position burst is about to be triggered, the platform will cancel all current open orders to release margin and maintain the position. A message will be displayed that the order has been successfully withdrawn and the open order has been cancelled to release margin.

The platform will calculate the risk factor of position burst, when the risk value of burst reaches 70%, it will send a strong closing warning notice to the user by email or SMS. The early warning notice will be judged every hour and sent again after the mark is reached. SMS notifications will also be made when a burst position is actually triggered to be initiated.

<aside class="formula">
  <code>
    risk of blow-out factor = maintenance margin / (available balance + position margin) \* 100%.
  </code>
</aside>

If the maintenance margin requirement is not met after the cancellation of an outstanding order, this remaining position is taken over by the strong closing engine at the bankruptcy price (i.e., the price at which the equity in the user's account equals zero) and this bankruptcy price is not displayed on the K-line.

The user's forced liquidation record can be seen directly in the history of commissions and transactions, where the transaction price is the bankruptcy price, not the forced liquidation price.

After the system takes over the position, if the market price is better than the bankruptcy price, the order will be delegated to the market to be filled in a matchmaking manner as soon as possible.

If a strongly closed position is not able to be closed in the market and when the price reaches the insolvency price and the premium is not sufficient to cover it, the ADL system will reduce the positions of investors holding positions in the opposite direction according to the ranking and notify the user by SMS or email.

User's loss on forced liquidation: all positions in all contracts will be forced to close. The loss of a user subject to forced liquidation is close to or equal to all the assets in his virtual contract account.

### Split orders for large position burst

When a large position burst event occurs in the trading market, the large position order is split into a number of small sub-orders for separate pending sell orders. In practice, burst position splitting involves complex calculation and sequencing issues.

## Automatic Deleverage System

### ADL trigger description

When an investor is forced to close a position, their remaining positions will be taken over by the forced liquidation system. If the force close position is not able to be closed out in the market and when the price reaches the bust price and the insurance is not sufficient to cover it, the ADL system will reduce the position for traders holding positions in the opposite direction. It essentially allows the profitable to share the risk of those who have blown their positions.

The automatic position reduction strategy is triggered when 50% of the risk reserve is still insufficient to close out a position based on the bankruptcy price of the forced liquidation position. The ADL ranking is based on the profit/loss of the position and the effective leverage used, i.e., the higher the profit and the more leverage used the higher the ranking. The highest ranked trader in the system will be selected first by the Auto-Reduction system.

Traders can view their priority ranking for automatic position reduction via the “ADL Ranking” indicator, like the chart below. The Auto-Reduction will close and reduce positions based on the bankruptcy price of the forced liquidation position. If a user's position is automatically reduced, they will receive an SMS or email notification and the floating profit of the position will be converted to a real profit. The rule for the number of positions to be reduced: after the first user's position in line has been fully reduced, the second position will continue to be reduced and so on.

### ADL ranking calculation method

<aside class="formula">
  <code>
    Ranking = % profit \* effective leverage (if profit, i.e. % profit > 0) = % profit / effective leverage (if loss, i.e. % profit < 0)
  </code>
</aside>

Where:

- `Effective leverage` is `abs(marker value) / (marker value - bankruptcy value)`

- `Profit percentage` is `(mark value - average open value) / |average open value|`

- `Marker value` is `value of position at marker price`
- `Insolvency value` is `value of position at insolvency price`
- `Average open value` is `value of position at average open price`

## Funding Costs

We anchor the market price of perpetual contracts to the spot price through a funding fee mechanism.

Long positions pay short positions when the market is bullish; short positions pay long positions when the market is bearish.

The funding fee is charged every 8 hours, at 4:00, 12:00 and 20:00 each day.

The funding fee is calculated as follows:

<aside class="formula">
  <code>
  Funding fee = value of position held * funding fee rate.
  </code>
</aside>

### Funding Fee Rate Calculation

The funding fee rate is comprised of two components: the interest rate and the premium.

The premium is calculated as:

<aside class="formula">
<code>
Premium = [Max(0, Impact Bid Price - Index Price) - Max(0, Index Price - Impact Ask Price)] / Price Index
</code>
</aside>

where the impact bid and impact ask prices refer to the average execution price for a market sell (bid) or market buy (ask) of the impact notational value.

Altogether, the funding rate is calculated as:

<aside class="formula">
  <code>
    Funding rate = Premium + Clamp(Interest Rate - Premium, a, b).
  </code>
</aside>

where for BTC, ETH, EOS, XRP, BCH, BSV, ETC, LTC, TRX,

<aside class="formula">
  <code>
    a = -0.3%,b = 0.3%
  </code>
</aside>

The `Clamp` function `Clamp(x, min, max)`, returns `min` when `x < = min` and `max` when `x>= max`.

## Price Indices

To ensure that the spot index price reasonably reflects the fair spot market price of each coin, we calcualte the index price as a weighted average of prices from 3 or more mainstream exchanges.

#### BTC/USD

`Okex`
`Binance`
`Coinbase`
`Bitstamp`

#### ETH/USD

`Okex`
`Binance`
`Coinbase`
`Bitstamp`

#### LTC/USD

`Okex`
`Coinbase`
`Bitstamp`

#### XRP/USD

`Okex`
`Coinbase`
`Bitstamp`

We has also implemented logic to ensure that index fluctuations are within the normal range when there is a significant deviation from the price of a single exchange.

If an exchange price deviates by more than 3% relative to the median price of all exchanges, that exchange price is calculated as median*0.97 or median*1.03.).

## Price Oracles

The Oracle Price is equal to the median of the reported prices of the 15 independent Chainlink master nodes. It's used to calculate:

- unrealised gains and losses.
- estimated strong parity.
- capital costs.

## Insurance Vault

The insurance vault is a reserve used by the platform to protect against the risk of position penetration, the main sources of which include a percentage of the commission generated by the transaction, and the surplus of the burst position.

In the event of a breakdown loss arising from a burst position, priority will be given to using the insurance pool to cover the loss.

## Price and Time Limits

### Price limits

When a user chooses to buy, there is a maximum buy price and no buy order can be submitted above that price. The calculation formula is:

Maximum Bid Price = Latest Price \* 101%

When a user selects to sell, there is a minimum sell price and no sell order can be submitted below that price. The formula is:

Minimum Sell Price = Latest Price / 101%

### Time limits

A user may not place a closing order within 2 minutes of placing an opening order.

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
