---
layout: post
title: "Binance Order Filters"
date:   2021-05-03 17:07:16 +1100
categories: binance
description: Understand Binance order filters to trade without issue
img: order-filter.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [binance, spot, filters]
redirect_from:
  - /binance/2021/05/03/binance-order-filters.html
---
One of the issues that many traders come across is the limits on exchanges especially [Binance][binance].

If you have run across a Filter failure whether it be PRICE_FILTER, LOT_SIZE, MIN_NOTIONAL, MARKET_LOT_SIZE
you know what I mean.

If you haven't seen any of these before then you are bound to sooner or later! When you start running strategies and
calculating percentages, or adding fees you will end up with some prices or quantities with greater precision
than is allowed.

Another complication is that these filters can interact with each other, while you may satisfy the PRICE_FILTER
you may not satisfy the MIN_NOTIONAL etc.

There are a number of other filters, for this post I'm going to focus on the filters related to the Spot market.

### What do these terms mean

Lot - refers to the amount of the order, otherwise known as quantity

Notional - the value in the base asset calculated by `price` * `quantity`

Base Asset - refers to the asset that is the quantity of a symbol. For the symbol BTCUSDT, BTC would be the base asset.

Quote Asset - refers to the asset that is the price of a symbol. For the symbol BTCUSDT, USDT would be the quote asset.

Symbol - a pair of assets that are traded on Binance

### How to find filters

Let's say we want to trade on BTCUSDT we will need to know the rules, but where are they?

They are found in the exchange info endpoint `get_exchange_info`, this returns a list of symbols, the
python-binance client has a helper function `get_symbol_info` if we are just interested in one symbol.

{% highlight python %}

import asyncio
import json

from binance import AsyncClient

    async def main():
    
        client = await AsyncClient.create()
        symbol_info = await client.get_symbol_info('BTCUSDT')
    
        print(json.dumps(symbol_info, indent=2))

        await client.close_connection()
    
    if __name__ == "__main__":
    
        loop = asyncio.get_event_loop()
        loop.run_until_complete(main())

{% endhighlight %}

This returns

{% highlight json %}

    {
      "symbol": "BTCUSDT",
      "status": "TRADING",
      "baseAsset": "BTC",
      "baseAssetPrecision": 8,
      "quoteAsset": "USDT",
      "quotePrecision": 8,
      "quoteAssetPrecision": 8,
      "baseCommissionPrecision": 8,
      "quoteCommissionPrecision": 8,
      "orderTypes": [
        "LIMIT",
        "LIMIT_MAKER",
        "MARKET",
        "STOP_LOSS_LIMIT",
        "TAKE_PROFIT_LIMIT"
      ],
      "icebergAllowed": true,
      "ocoAllowed": true,
      "quoteOrderQtyMarketAllowed": true,
      "isSpotTradingAllowed": true,
      "isMarginTradingAllowed": true,
      "filters": [
        {
          "filterType": "PRICE_FILTER",
          "minPrice": "0.01000000",
          "maxPrice": "1000000.00000000",
          "tickSize": "0.01000000"
        },
        {
          "filterType": "PERCENT_PRICE",
          "multiplierUp": "5",
          "multiplierDown": "0.2",
          "avgPriceMins": 5
        },
        {
          "filterType": "LOT_SIZE",
          "minQty": "0.00000100",
          "maxQty": "9000.00000000",
          "stepSize": "0.00000100"
        },
        {
          "filterType": "MIN_NOTIONAL",
          "minNotional": "10.00000000",
          "applyToMarket": true,
          "avgPriceMins": 5
        },
        {
          "filterType": "ICEBERG_PARTS",
          "limit": 10
        },
        {
          "filterType": "MARKET_LOT_SIZE",
          "minQty": "0.00000000",
          "maxQty": "104.64146873",
          "stepSize": "0.00000000"
        },
        {
          "filterType": "MAX_NUM_ORDERS",
          "maxNumOrders": 200
        },
        {
          "filterType": "MAX_NUM_ALGO_ORDERS",
          "maxNumAlgoOrders": 5
        }
      ],
      "permissions": [
        "SPOT",
        "MARGIN"
      ]
    }

{% endhighlight %}

There's a lot of information here, so let's break it down.

`baseAssetPrecision` and `quoteAssetPrecision` are interesting as they describe the precision your balance
can be for the base and the quote asset. 

This is generally a larger precision than what you can trade, you might have 1034.134894 USDT, but as you'll
see below you may end up with some small balance that are called dust.

Don't despair though as [Binance][binance] does have an option to convert dust to BNB.


### PRICE_FILTER

Remember that price relates to the quote asset, and for BTCUSDT this is USDT. The price tell us how much we 
can buy or sell BTC for in USDT, well depending on the market.

{% highlight json %}

    {
      "filterType": "PRICE_FILTER",
      "minPrice": "0.01000000",
      "maxPrice": "1000000.00000000",
      "tickSize": "0.01000000"
    }

{% endhighlight %}

I've pulled out the relevant information. So what does this tell us.

`"minPrice": "0.01000000"` - we know the min price can be 0.01, if only!

`"maxPrice": "1000000.00000000"` - we know the max price usable on the system is 1,000,000.00.

`"tickSize": "0.01000000"` - tick size is the increment we can increase or decrease our price by.

{% highlight python %}

    price = 65000.01  # valid
    price = 65000.001 # invalid fails tick size
    price = 0.001     # invalid - fails minimum price
    price = 2000000   # invalid - fails maximum price, but maybe one day...

{% endhighlight %}

### MARKET_LOT_SIZE

The Market Lot Size filter defines rules around Market orders for a symbol.

Remember `lot` relates to quantity and that's the base asset, for BTCUSDT, BTC is the base asset.

{% highlight json %}

    {
      "filterType": "MARKET_LOT_SIZE",
      "minQty": "0.00000000",
      "maxQty": "104.64146873",
      "stepSize": "0.00000000"
    }

{% endhighlight %}

This filter has a couple of values that are 0, which means this particular part of the filter does not apply.

`"maxQty": "104.64146873"` - indicates the maximum quantity allowed for a market order

If `minQty` was set then that would define the minimum quantity allowed.

`stepSize` works similarly to the `tickSize` of the `PRICE_FILTER` filter and defines the intervals we can
increase or decrease the quantity by.

### LOT_SIZE

The Lot Size filter defines the limits on the quantity for both Limit and Market orders for a symbol.

{% highlight json %}

    {
      "filterType": "LOT_SIZE",
      "minQty": "0.00000100",
      "maxQty": "9000.00000000",
      "stepSize": "0.00000100"
    }

{% endhighlight %}

`"minQty": "0.00000100"` - The minimum quantity of BTC we can place an order for

`"maxQty": "9000.00000000"` - The maximum quantity of BTC we can place an order for

`"stepSize": "0.00000100"` - The interval we can increase or decrease the quantity by

### MIN_NOTIONAL

The Min Notional filter defines the minimum value calculated in the quote asset for a symbol.

For our symbol BTCUSDT the quote symbol is USDT, so the min notional values is in USDT.

From above, we know we can calculate the notional value as following.

{% highlight python %}

    notional_value = price * quantity

{% endhighlight %}

{% highlight json %}

    {
      "filterType": "MIN_NOTIONAL",
      "minNotional": "10.00000000",
      "applyToMarket": true,
      "avgPriceMins": 5
    }

{% endhighlight %}


How about Market orders you might ask, that doesn't have a price, does this filter not apply?

Sorry to say, Market orders don't always allow you a free pass for this filter.

`"applyToMarket": true` if this is false then it doesn't apply. As of writing if you do loop through all
the symbols and check the MIN_NOTIONAL filter then none of them have `applyToMarket` as false. Even if they
did you are probably safer to just follow it.

Also note the `"avgPriceMins": 5` part, what this is hinting to is that Binance calculates an average price
for each symbol and that is used to check this MIN_NOTIONAL filter for Market orders.

To find the average price for BTCUSDT, let's modify the `main` function of our code from above and run again

{% highlight python %}

    async def main():
    
        client = await AsyncClient.create()
    
        print(json.dumps(await client.get_avg_price(symbol='BTCUSDT'), indent=2))
    
        await client.close_connection()

{% endhighlight %}

{% highlight json %}

    {
      "mins": 5,
      "price": "58425.57064095"
    }

{% endhighlight %}

We see that the `mins` value matches the symbol info response, and we now also have a price to use for 
Market orders in our notional value calculation.

### Ready to trade?

Now you know what they all mean you're ready to trade? Or more confused about how to determine a price
that will pass the filters?

Let's say that you have been watching the BTCUSDT, all the squiggles align, the price is going to go 
up and you're destined to make 3% profit.

We place a market order to get into the market fast, although let's not use our own money just yet. 

Luckily Binance offers a testnet to test out Spot trading. Head to [testnet.binance.vision][testnet-binance]
login with Github credentials and click on Generate HMAC_SHA256 Key, copy the API Key and API secret into the
values below.

We don't have a lot of USDT and we know the minimum quantity is 0.000001, so we start small.

{% highlight python %}

    from binance.exceptions import BinanceAPIException

    api_key = '<testnet api_key>'
    api_secret = '<testnet api_secret>'
    
    async def main():
    
        quantity = '0.000001'
        client = await AsyncClient.create(api_key=api_key, api_secret=api_secret, testnet=True)
    
        try:
            market_res = await client.order_market_sell(symbol='BTCUSDT', quantity=quantity)
        except BinanceAPIException as e:
            print(e)
        else:
            print(json.dumps(market_res, indent=2))
    
        await client.close_connection()

{% endhighlight %}

{% highlight python %}

    APIError(code=-1013): Filter failure: MIN_NOTIONAL

{% endhighlight %}

Oh, we forgot about the notional value, it was only `0.000001 * 58425.57064095` which is `0.058`.

The min notional value is `10.0`, so our minimum is `10.0 / 58425.57064095 = 0.00017115`, lets be a bit more confident 
now and set `quantity = 0.001` and run our script.

{% highlight json %}

    {
      "symbol": "BTCUSDT",
      "orderId": 526480,
      "orderListId": -1,
      "clientOrderId": "T6AAjnSLxvAA24CDiWGUFC",
      "transactTime": 1620041560274,
      "price": "0.00000000",
      "origQty": "0.00100000",
      "executedQty": "0.00100000",
      "cummulativeQuoteQty": "60.54000143",
      "status": "FILLED",
      "timeInForce": "GTC",
      "type": "MARKET",
      "side": "BUY",
      "fills": [
        {
          "price": "58801.75000000",
          "qty": "0.00017100",
          "commission": "0.00000000",
          "commissionAsset": "BTC",
          "tradeId": 128006
        },
        {
          "price": "60125.42000000",
          "qty": "0.00067900",
          "commission": "0.00000000",
          "commissionAsset": "BTC",
          "tradeId": 128007
        },
        {
          "price": "64398.28000000",
          "qty": "0.00015000",
          "commission": "0.00000000",
          "commissionAsset": "BTC",
          "tradeId": 128008
        }
      ]
    }

{% endhighlight %}

Perfect now our order has been filled, and we can work out the price it filled at.

{% highlight python %}

    async def main():

        fills = [
            {
                "price": "58801.75000000",
                "qty": "0.00017100",
                "commission": "0.00000000",
                "commissionAsset": "BTC",
                "tradeId": 128006
            },
            {
                "price": "60125.42000000",
                "qty": "0.00067900",
                "commission": "0.00000000",
                "commissionAsset": "BTC",
                "tradeId": 128007
            },
            {
                "price": "64398.28000000",
                "qty": "0.00015000",
                "commission": "0.00000000",
                "commissionAsset": "BTC",
                "tradeId": 128008
            }
        ]
        ave_price = sum([float(f['price']) * (float(f['qty']) / quantity) for f in fills])
        print(ave_price)

{% endhighlight %}

This gives us `58802.609`, great now we can work out where to place our Limit sell order.

{% highlight python %}

    async def main():
    
        quantity = 0.001
        profit_pct = 1 + (3 / 100)
    
        purchase_price = 58802.609
        target_price = purchase_price * profit_pct
        print(f'target_price: {target_price}')
    
        client = await AsyncClient.create(api_key=api_key, api_secret=api_secret, testnet=True)
    
        try:
            limit_res = await client.order_limit_sell(symbol='BTCUSDT', price=target_price, quantity=quantity)
        except BinanceAPIException as e:
            print(e)
        else:
            print(json.dumps(limit_res, indent=2))
    
        await client.close_connection()

{% endhighlight %}

{% highlight python %}

    target_price: 60566.687269999995
    APIError(code=-1111): Precision is over the maximum defined for this asset.

{% endhighlight %}

Well that's annoying, now we need to do a little bit more work.

We can see that is greater than our asset precision of 8 for a start, so let's adjust the target_price.

{% highlight python %}

        target_price = f'{purchase_price * profit_pct:0.8f}'

{% endhighlight %}


{% highlight python  %}

    target_price: 60566.68727000
    APIError(code=-1013): Filter failure: PRICE_FILTER

{% endhighlight %}

Ok, we know what this means, we need to satisfy 2 decimal places.

{% highlight python %}

        target_price = f'{purchase_price * profit_pct:0.2f}'

{% endhighlight %}

{% highlight python  %}

    target_price: 60566.69
    {
      "symbol": "BTCUSDT",
      "orderId": 528597,
      "orderListId": -1,
      "clientOrderId": "nVYtSgd8gk7k0uUJhNe2Ms",
      "transactTime": 1620042603709,
      "price": "60566.69000000",
      "origQty": "0.00100000",
      "executedQty": "0.00000000",
      "cummulativeQuoteQty": "0.00000000",
      "status": "NEW",
      "timeInForce": "GTC",
      "type": "LIMIT",
      "side": "SELL",
      "fills": []
    }

{% endhighlight %}

Great now our limit order has finally been placed, and we can reap our profits.

### Using helpers

The above code needs to know that our `tickSize` of `0.01` means that we should round to 2 decimal places.

We have a helper function that can do this work for us called `round_step_size`, this can be applied
to both price and quantity values.

{% highlight python %}

    # add this import
    from binance.helpers import round_step_size
    
    # update main
    async def main():
    
        quantity = 0.001
        profit_pct = 1 + (3 / 100)
    
        purchase_price = 58802.609
        target_price = round_step_size(purchase_price * profit_pct, 0.01)
        print(f'target_price: {target_price}')
    
        client = await AsyncClient.create(api_key=api_key, api_secret=api_secret, testnet=True)
    
        try:
            limit_res = await client.order_limit_sell(symbol='BTCUSDT', price=target_price, quantity=quantity)
        except BinanceAPIException as e:
            print(e)
        else:
            print(json.dumps(limit_res, indent=2))
    
        await client.close_connection()

{% endhighlight %}

{% highlight python  %}

    target_price: 60566.69
    {
      "symbol": "BTCUSDT",
      "orderId": 528597,
      "orderListId": -1,
      "clientOrderId": "nVYtSgd8gk7k0uUJhNe2Ms",
      "transactTime": 1620042603709,
      "price": "60566.69000000",
      "origQty": "0.00100000",
      "executedQty": "0.00000000",
      "cummulativeQuoteQty": "0.00000000",
      "status": "NEW",
      "timeInForce": "GTC",
      "type": "LIMIT",
      "side": "SELL",
      "fills": []
    }

{% endhighlight %}

The target_price is the same as above, and we have placed the limit order successfully.

### Next Steps

Hopefully this has helped demystify some of the terminology that you will see when interacting with the 
exchange, and you now have some tools to handle price and quantity parameters when trading.

Register with [Binance][binance] and switch over from the testnet and try out your strategies.


[testnet-binance]: https://testnet.binance.vision/
[python-binance]: https://github.com/sammchardy/python-binance
[binance]: https://www.binance.com/en/register?ref=KNNBYWG8
