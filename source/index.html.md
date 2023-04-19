---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - java
  - json

includes:
#  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for Satori Perpetual Protocol
---

# Introduction

Welcome to the docs for the Satori Perpetual Protocol. Satori is a decentralized financial derivatives platform built on Polygon zkEVM, zkSync, and Scroll. It features an order book model in collaboration with many market makers to provide comprehensive exposure to a wide range of assets, and an “off-chain aggregation and on-chain settlement” design that combines the security and transparency of a decentralized exchange, with the speed and usability of a centralized exchange.

# User funds

## Adding Funds
Users can directly deposit USDT to trade on the Satori exchange.

## Withdrawing Funds
Users can withdraw their funds in USDT at any point.

# Perpetuals

A perpetual contract is a derivative that approximates leveraged spot trading. Investors can buy to gain from rising prices, or sell to gain from falling prices. The perpetual contract differs from traditional futures in some ways: it has no expiration time and therefore no restrictions on how long users are able to hold a position. In order to ensure that the underlying price index is tracked, perpetual contracts are guaranteed to follow the price of the underlying asset through a funding fee mechanism.

## Matchmaking

Orders will be matched by price-time on a FIFO basis. A transaction happens if there is a buy order that has a price equal or greater than a sell order, or if there is a market order placed in one direction and orders exist in the opposing direction.

## Margin

Satori enforces margin requirements for users -- an initial margin requirement to open and size-up positions, and a maintenance margin requirement to avoid liquidations. Margin can be added and removed at will above the initial margin.

Either cross-margining or isolated-margining can be used. Cross-margining means that all of the margin balance in a user's account is used as margin for any position. In this mode, a user's position is at a much lower risk of being forced to close. Isolated-margining means that the guaranteed assets allocated to a position are limited to a certain amount. In this mode, if a user's position is blown out due to price fluctuations, only the margin amount of the position in that direction will be lost, and no other funds in the account will be affected.

### Initial Margin

The maximum supported leverage is 25x, so least 4% of the position value must be used as collateral upon opening a new position.

For example, with $100 of collateral, the maximum position size that can be put on is:

| Leverage | Position Size |
| -------- | ------------- |
| 5x       | $500          |
| 10x      | $1,000        |
| 25x      | $2,500        |

### Maintenance Margin

If the value of a position falls below the maintenance margin level, it may be automatically closed out by the liquidation engine.

The maintenance margin rates are determined on a per-token-pair basis:

| Token | Maintenance Margin Rate |
| ----- | ----------------------- |
| BTC   | 0.5%                    |
| ETH   | 0.5%                    |
| MATIC | 0.5%                    |

For example, a long leveraged position in BTC-USDT may be liquidated after the mark price dips below 0.5% above the bankruptcy price.

The denominated asset for all perpetual markets is USDT.

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

In other words, the funding fee rate will equal the hourly interest rate if the premium index is within 0.05% of the hourly interest rate.

### Funding Fee Rate Cap

The absolute value of the Funding Fee Rate is capped at 0.75%.

## Trading fees
The fees are differentiated between whether or not the user was a maker or taker for the trade.

| Token | Maker Fees | Taker Fees |
| ----- | ---------- | ---------- |
| BTC   | 0.04%      | 0.06%      |
| ETH   | 0.04%      | 0.06%      |
| MATIC | 0.04%      | 0.06%      |

## Index Price

The `Index Price` refers to the price of the underlying asset on the spot market. It's an aggregate price based off price data from multiple exchanges, calculated every second.

The exchanges we aggregate from are as follows for each market:

#### BTC-USD

`Coinbase`
`Binance`
`OKX`

#### ETH-USD

`Coinbase`
`Binance`
`OKX`

#### MATIC-USD

`Coinbase`
`Binance`
`OKX`

For BTC-denominated currency pairs, the system multiplies by the OKX BTC-USD index to convert to the US dollar price.

We have implemented logic to ensure that index fluctuations are within the normal range when there is a significant deviation from the price of a single exchange. In addition, the system invalidates exchanges whose latest transaction price and trade volume have not been updated for a period of time.

If an exchange price deviates by more than 3% relative to the median price of all exchanges, the exchange price will be calculated according to the `median*0.97` or the `median*1.03`.

## Oracle Price

The `Oracle Price` is an aggregate price calculated using multiple on-chain price oracles. Oracle prices are used to determine collateralization and liquidations on Satori.

Since the oracle price is also an aggregate price, it offers similar protection from flash crashes as the index price does.

For the Alpha launch, Satori runs its own oracle nodes.

# Liquidations

Positions where the margin rate is less than or equal to the maintenance margin rate may be liquidated.

## Closing Price

The closing price is determined by:

<aside class="formula">
  <code>
    Closing Price (long) =
      Average Opening Price * (1 + Maintenance Margin Rate) - (Position Margin / Number of Contracts)

    Closing Price (short) =
      Average Opening Price * (1 - Maintenance Margin Rate) + (Position Margin / Number of Contracts)
  </code>
</aside>

If the position is liquidated at a price better than the bankruptcy price, then the funds will be added to the insurance vault. Otherwise, if the position is liquidated at a worse price, then the insurance vault funds will be used to cover the loss due to the position. If that insurance vault is insufficient, then the [auto-deleveraging system](#automatic-deleveraging-system) will reduce the positions of investors that have positions in the opposite direction.

## Insurance Vault

When a position is liquidated, it will be taken over by the forced liquidation system. If the liquidation cannot be fulfilled by the time the perpetual reaches the bankruptcy price, the loss will be covered by the insurance vault, a reserve maintained by the platform.

The insurance vault is currently seeded and maintained by the Satori team, and will eventually be funded through trading fees.

## Automatic Deleveraging System

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
- `% profit` is `(mark value - average open value) / average opening value`
- `Mark value` is the value of the position at the mark price
- `Bankruptcy value` is the value of position at the bankruptcy price
- `Average opening value` is the the value of position at average opening price

# Websocket API

> Example

```java
JSONObject params=new JSONObject();
            params.put("method","SUBSCRIBE");
            params.put("event","api_account");
            params.put("apiKey","ZPV45xgpL3ofaHLPicjA0D4PLgzPRlKd");
            params.put("symbol","USDT");
			//params.put("period","1MIN");
            long time=new Date().getTime();
            params.put("timestamp",time);
            String sha256 = HMacUtil.sha256(String.valueOf(time), "5928758659f61d099c191607b3ebdf07aabf1cf2");
            params.put("signature",sha256);
            System.out.println(params.toJSONString());
```

> HMacUtil.java

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;

public class HMacUtil {


    public static String sha256(String message,String secret) throws InvalidKeyException, NoSuchAlgorithmException {
        Mac sha256HMAC = Mac.getInstance("HmacSHA256");
        SecretKeySpec secret_key = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
        sha256HMAC.init(secret_key);
        return byteArrayToHexString(sha256HMAC.doFinal(message.getBytes()));
    }

    private static String byteArrayToHexString(byte[] b) {
        StringBuilder hs = new StringBuilder();
        String stmp;
        for (int n = 0; b!=null && n < b.length; n++) {
            stmp = Integer.toHexString(b[n] & 0XFF);
            if (stmp.length() == 1)
                hs.append('0');
            hs.append(stmp);
        }
        return hs.toString().toLowerCase();
    }
}
```


Satori offers a WebSocket API for streaming updates.

- Testing: `ws://zk-test.satori.finance`
- Production: `ws://zk.satori.finance`

## Subscriptions

### Subscribing to Account Data

url: `${wsUrl}/api/account/ws`

> Example Request

```json
{
  "symbol":"MATIC",
  "method":"SUBSCRIBE",
  "apiKey":"5WlQeUYgKx85H8ID3XygdYVMOFjlXyw9",
  "signature":"1b883450a6bb3749729d429133fd06cec28215a6abcb876b08cc531ed86b7782",
  "event":"api_account",
  "timestamp":1657157345159
}
```

> Example Response

```json
{
  "success":true,
  "msg":"subscribe success: api_account",
  "method":"SUBSCRIBE",
  "event":"api_account"
}
```

#### Request Parameters

| Field     | Type   | Required | Description                               |
| --------- | ------ | -------- | ----------------------------------------- |
| method    | String | √        | Methods - SUBSCRIBE, UNSUBSCRIBE          |
| event     | String | √        | Event, see event table (api_account）     |
| apiKey    | String | √        | API Key                                   |
| timestamp | Long   | √        | Time                                      |
| signature | String | √        | Signature                                 |
| symbol    | String | √        | Asset symbol                              |

#### Response Parameters

| Field   | Type    | Required | Description                                    |
| ------- | ------- | -------- | ---------------------------------------------- |
| success | Boolean | √        | true,false                                     |
| msg     | String  | √        | Message                                        |
| method  | String  | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event   | String  | √        | The same as the event parameter in the request. See the event table for details. |
| symbol  | String  | √        | Asset symbol                                   |

### Subscribing to Orders

url: `${wsUrl}/api/entrust/ws`

#### Request Parameters

| Field     | Type    | Required | Description                               |
| --------- | ------ | -------- | ------------------------------------------ |
| method    | String | √        | Methods - SUBSCRIBE，UNSUBSCRIBE           |
| event     | String | √        | See event table (api_entrust) for details. |
| apiKey    | String | √        | API key                                    |
| timestamp | Long   | √        | Time                                       |
| signature | String | √        | Signature                                  |
| pair      | String | √        | Trading Pair                               |

#### Response Parameters

| Field   | Type    | Required | Description                                    |
| ------- | ------- | -------- | ---------------------------------------------- |
| success | Boolean | √        | true,false                                     |
| msg     | String  | √        | Message                                        |
| method  | String  | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event   | String  | √        | The same as the event parameter in the request. See the event table for details. |
| pair    | String  | √        | Trading pair                                   |

### Subscribing to Positions

url: `${wsUrl}/api/position/ws`

#### Request Parameters

| Field     | Type   | Required | Description                                    |
| --------- | ------ | -------- | ---------------------------------------------- |
| method    | String | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event     | String | √        | See the event table (api_position) for details.|
| apiKey    | String | √        | API key                                        |
| timestamp | Long   | √        | Time                                           |
| signature | String | √        | Signature                                      |
| pair      | String | √        | Trading pair                                   |

#### Response Parameters

| Field   | Type    | Required | Description                                    |
| ------- | ------- | -------- | ---------------------------------------------- |
| success | Boolean | √        | true,false                                     |
| msg     | String  | √        | Message                                        |
| method  | String  | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event   | String  | √        | The same as the event parameter in the request. See the event table for details. |
| pair    | String  | √        | Trading pair                                   |

### Subscribing to Kline Data

url: `${wsUrl}/api/kline/ws`

#### Request Parameters

| Field     | Type   | Required | Description                                |
| --------- | ------ | -------- | ------------------------------------------ |
| method    | String | √        | Methods - SUBSCRIBE，UNSUBSCRIBE           |
| event     | String | √        | Check event table （api_kline）            |
| apiKey    | String | √        | API key                                    |
| timestamp | Long   | √        | Time                                       |
| signature | String | √        | Signature                                  |
| pair      | String | √        | Trading pair                               |
| period    | String | √        | Check the periodEnum table                 |

#### Response Parameters

| Field   | Type    | Required | Description                                    |
| ------- | ------- | -------- | ---------------------------------------------- |
| success | Boolean | √        | true,false                                     |
| msg     | String  | √        | Message                                        |
| method  | String  | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event   | String  | √        | The same as the event parameter in the request. See the event table for details. |
| pair    | String  | √        | Trading pair                                   |
| period  | String  | √        | Check the periodEnum table                     |

### Subscribing to Market Data

url: `${wsUrl}/api/depth/ws`

#### Request Parameters

| Field     | Type   | Required | Description                                |
| --------- | ------ | -------- | ------------------------------------------ |
| method    | String | √        | Methods - SUBSCRIBE，UNSUBSCRIBE           |
| event     | String | √        | See event table（api_depth）               |
| apiKey    | String | √        | API key                                    |
| timestamp | Long   | √        | Time                                       |
| signature | String | √        | Signature                                  |
| pair      | String | √        | Trading pair                               |

#### Response parameters

| Field   | Type    | Required | Description                                    |
| ------- | ------- | -------- | ---------------------------------------------- |
| success | Boolean | √        | true,false                                     |
| msg     | String  | √        | msg                                            |
| method  | String  | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event   | String  | √        | The same as the event parameter in the request. See the event table for details. |

### Subscribing to Trading Data

 url: `${wsUrl}/api/trans/ws`

#### Request Parameters

| Field     | Type   | Required | Description                                 |
| --------- | ------ | -------- | ------------------------------------------- |
| method    | String | √        | Methods - SUBSCRIBE，UNSUBSCRIBE            |
| event     | String | √        | See event table （api_trade）             |
| apiKey    | String | √        | API key                                     |
| timestamp | Long   | √        | Time                                        |
| signature | String | √        | Signature                                   |
| pair      | String | √        | Trading pair                                |
| period    | String | √        | Check the periodEnum table                  |

#### Response parameters

| Field   | Type    | Required | Description                                    |
| ------- | ------- | -------- | ---------------------------------------------- |
| success | Boolean | √        | true,false                                     |
| msg     | String  | √        | msg                                            |
| method  | String  | √        | Methods - SUBSCRIBE，UNSUBSCRIBE               |
| event   | String  | √        | The same as the event parameter in the request. See the event table for details. |
| pair    | String  | √        | Trading pair                                   |

## Updates
### Account Update

> Example response

```json
{
  "data":
  {
    "availableAmount":"8000",
    "coinId":5,
    "frozenAmount":"0",
    "logId":8,
    "operateAmount":"0.00",
    "symbol":"MATIC",
    "coinId":0
  },
  "event":"api_account",
  "success":true
}
```

| Field   | Type    | Required | Description                     |
| ------- | ------- | -------- | ------------------------------- |
| data    | Object  |          | Account information             |
| event   | String  |          | See event table (api_account)   |
| success | Boolean |          | true,false                      |

#### Data

| Field           | Type   | Required | Description                              |
| --------------- | ------ | -------- | -----------------------------------------|
| availableAmount | String |          | Available amount                         |
| frozenAmount    | String |          | Frozen Amount                            |
| logId           | Long   |          | Log ID                                   |
| operateAmount   | String |          | Operating amount                         |
| symbol          | String |          | Asset symbol                             |
| coinId          | Long   |          | Coin ID                                  |

### Order Update

| Field   | Type    | Required | Description                     |
| ------- | ------- | -------- | ------------------------------- |
| data    | Object  |          | Account information             |
| event   | String  |          | See event table(api_entrust)    |
| success | Boolean |          | true,false                      |

#### Data

| Field            | Type    | Required | Description                            |
| ---------------- | ------- | -------- | -------------------------------------- |
| currentEntrustId | Long    |          | Order ID                               |
| changeType       | String  |          | Type：ENTRUST；NEW；TRADE；CANCELED     |
| dealQuantity     | String  |          | Quantity already traded                |
| direction        | String  |          | LONG or SHORT                          |
| lever            | Integer |          | Leverage                               |
| price            | String  |          | Price                                  |
| quantity         | String  |          | Quantity                               |
| symbol           | String  |          | Asset symbol                           |
| nowDealQuantity  | String  |          | Currently traded quantity              |
| isClose          | Boolean |          | true (closing position);false (opening position); |
| contractPairId   | Long    |          | Trading pair ID                        |
| dealAmount       | String  |          | Amount already traded                  |
| matchType        | Integer |          | 1 GTC；2 IOC；3 FOK；4 POST_ONLY ；    |
| clientOrderId    | String  |          | Client Order ID                        |

### Position Update

| Field   | Type    | Required | Description                      |
| ------- | ------- | -------- | -------------------------------- |
| data    | Object  |          | Account information              |
| event   | String  |          | See event table (api_position)   |
| success | Boolean |          | true,false                       |

#### Data

| Field             | Type    | Required | Description                           |
| ----------------- | ------- | -------- | ------------------------------------- |
| changeType        | String  |          | Type：NEW; INCREASE; CLOSE; LIQUIDATE; REDUCE; |
| currentQuantity   | String  |          | Updated quantity                      |
| direction         | String  |          | LONG or SHORT                         |
| operateQuantity   | String  |          | Changed quantity                      |
| positionId        | Long    |          | Position record ID                    |
| symbol            | String  |          | Trading symbol                        |
| canClosedQuantity | String  |          | Available quantity to close           |
| contractPairId    | Long    |          | Trading pair ID                       |
| positionType      | Integer |          | Position Type                         |

### Kline Update

| Field   | Type    | Required | Description                      |
| ------- | ------- | -------- | -------------------------------- |
| data    | Object  |          | Account information              |
| event   | String  |          | See event table (api_kline)      |
| success | Boolean |          | true,false                       |

#### Data

| Field          | Type       | Required | Description                          |
| -------------- | ---------- | -------- | ------------------------------------ |
| amount         | BigDecimal |          | Amount                               |
| close          | BigDecimal |          | Closing price                        |
| count          | Integer    |          | Number of transactions               |
| high           | BigDecimal |          | Highest price                        |
| low            | BigDecimal |          | Lowest price                         |
| open           | BigDecimal |          | Opening price                        |
| period         | String     |          | Time interval                        |
| quantity       | BigDecimal |          | Trading quantity                     |
| time           | Long       |          | Kline start time                     |
| contractPairId | Long       |          | Contract trading pair IDs            |

### Market Depth Update

> Example

```json
{
  "data":
  {
    "asks":
    [
      {"price":"8.291","quantity":"20.163055"},
      {"price":"8.302","quantity":"27.074094"},
      {"price":"8.33","quantity":"16.12535"},
      {"price":"8.333","quantity":"7.617261"},
      {"price":"8.347","quantity":"7.274368"},
      {"price":"8.36","quantity":"29.826174"},
      {"price":"8.385","quantity":"6.084978"},
      {"price":"8.402","quantity":"13.370699"},
      {"price":"9.16","quantity":"16.669687"},
      {"price":"9.182","quantity":"115.09525"}
    ],
    "bids":
    [
      {"price":"7.794","quantity":"7.979346"},
      {"price":"7.783","quantity":"24.131828"},
      {"price":"7.765","quantity":"16.167428"},
      {"price":"7.76","quantity":"21.880696"},
      {"price":"7.755","quantity":"14.01524"},
      {"price":"7.754","quantity":"4.673284"},
      {"price":"7.73","quantity":"16.060928"},
      {"price":"7.707","quantity":"25.653437"},
      {"price":"7.705","quantity":"20.066533"},
      {"price":"7.686","quantity":"19.35628"}
    ]
  },
  "event":"api_depth",
  "pair":"DOT-USDT",
  "success":true
}
```

| Field   | Type    | Required | Description                    |
| ------- | ------- | -------- | ------------------------------ |
| data    | Object  |          | Account information            |
| event   | String  |          | See event table(api_depth)     |
| success | Boolean |          | true,false                     |

#### data

| Field | Type | Required | Description  |
| ----- | ---- | -------- | ------------ |
| asks  | List |          | List of asks |
| bids  | List |          | List of bids |

#### asks

| Field    | Type   | Required | Description  |
| -------- | ------ | -------- | ------------ |
| price    | String |          | Price        |
| quantity | String |          | Quantity     |

#### bids

| Field    | Type   | Required | Description  |
| -------- | ------ | -------- | ------------ |
| price    | String |          | Price        |
| quantity | String |          | Quantity     |

### Trade Data Update

| Field   | Type    | Required | Description                    |
| ------- | ------- | -------- | ------------------------------ |
| data    | Object  |          | Account information            |
| event   | String  |          | See event table(api_trade)     |
| success | Boolean |          | true,false                     |

### Data

| Field          | Type       | Required | Description                          |
| ------------------- | ---------- | -------- | ------------------------------- |
| contractMatchPairId | long       |          | Match record ID                 |
| pair                | String     |          | Trading pair                    |
| price               | BigDecimal |          | Price                           |
| quantity            | BigDecimal |          | Quantity                        |
| amount              | BigDecimal |          | Trading amount                  |
| isLong              | Boolean    |          | true or false                   |
| time                | String     |          | Creation time                   |
| timestamp           | Long       |          | Time                            |
| contractPairId      | Long       |          | Trading pair ID                 |

### Fields

#### periodEnum

periodEnum represents the time periods for data used in the API

`1SECOND`
`1MIN`
`3MIN`
`5MIN`
`15MIN`
`30MIN`
`1HOUR`
`4HOUR`
`8HOUR`
`12HOUR`
`1DAY`
`1WEEK`
`1MONTH`
`1YEAR`

#### events

`api_entrust`
`api_position`
`api_kline`
`api_depth`
`api_trade`
`api_account`

`api_entrust_res`
`api_position_res`
`api_kline_res`
`api_depth_res`
`api_trade_res`
`api_account_res`

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
