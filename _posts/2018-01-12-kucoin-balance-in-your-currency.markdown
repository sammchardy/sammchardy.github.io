---
layout: post
title: "Calculate Kucoin Balance in your Currency"
date:   2018-01-12 21:07:16 +1100
categories: kucoin
description: Keep track of your crypto portfolio balance on Kucoin
img: kucoin-balance.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [kucoin, balance]
redirect_from:
  - /kucoin/2018/01/12/kucoin-balance-in-your-currency.html
---
With so many different coins available it can be hard to keep track of your portfolio.

Why not use python to make it simple for you. In this post I'll step through downloading your balances on Kucoin
and then calculating our total balance in USD and other currencies supported by Kucoin.

At the time of writing Kucoin supports these currencies      
USD, EUR, AUD, CAD, CHF, CNY, GBP, JPY, NZD, BGN, BRL, CZK, DKK, HKD, HRK, HUF, IDR, ILS, INR, KRW, MXN, MYR, NOK, PHP, PLN, RON, RUB, SEK, SGD, THB, TRY, ZAR

This example requires an account on [Kucoin][kucoin] as it utilises private API calls, you will need an API key and secret
go to your [API settings][kucoin-api-settings] to generate one, and enable it.

You will also need to have the [python-kucoin][python-kucoin] library installed.

### Fetching our coin balances

Kucoin provides an endpoint to fetch all balances, so let's try it.

{% highlight python %}

from kucoin.client import Client

# fill in your API key and secret values
KUCOIN_API_KEY = ''
KUCOIN_API_SECRET = ''

# create the Kucoin client with your key
client = Client(KUCOIN_API_KEY, KUCOIN_API_SECRET)

# fetch your balances
balances = client.get_all_balances()

{% endhighlight %}

Now we have a list of balance objects in the format

{% highlight json %}

{
    "coinType": "BTC",
    "balance": 1233214,  
    "freezeBalance": 321321, 
    "balanceStr": "1233214"
    "freezeBalanceStr": "321321"
}

{% endhighlight %}

- `coinType` is the code of the coin
- `balance` is the total available balance on the exchange
- `freezeBalance` is the amount tied up in active trades
- `balanceStr` is a string representation of the balance
- `freezeBalanceStr` is a string representation of the frozen balance

So to get our total balance we need to add the `balance` and `freezeBalance` values for each coin.

If we inspect the list we see that it has returned some coins that we don't have any balance at all, e.g

{% highlight json %}

{
    "coinType": "SNOV",
    "balanceStr": "0.0",
    "freezeBalance": 0.0,
    "balance": 0.0,
    "freezeBalanceStr": "0.0"
}

{% endhighlight %}

We will be able to filter these out by testing for `balanceStr == '0.0' and freezeBalanceStr == '0.0'`

To get a unique list of the coins we do have we can use python list comprehension.

{% highlight python %}

balances = client.get_all_balances()
coins = [b['coinType'] for b in balances]

{% endhighlight %}

### Fetching value in Fiat

Kucoin has a helpful function to fetch Fiat rates for a list of coins which is exactly what we need.

It requires a comma separated list of coins as a parameter

{% highlight python %}

# create comma separated list of coins

coins_csl = ','.join(coins)
currency_res = client.get_currencies('BTC,RPX,ONION,GOD')
rates = currency_res['rates']

{% endhighlight %}

There are some forked coins on Kucoin such as GOD (at time of writing) that don't have a fiat value, and you
won't even find on Coin Market Cap yet.

The `get_currencies` call does not return them so we need to be aware of that.

### Calculate our total in Fiat

Now that we have the balance of each coin and the fiat value of the coin we can finally determine a total.

{% highlight python %}

CURRENCY = 'EUR'

total = 0
for b in balances:
    # ignore any coins of 0 value
    if b['balanceStr'] == '0.0' and b['freezeBalanceStr'] == '0.0':
        continue
    # ignore the coin if we don't have a rate for it
    if b['coinType'] not in rates:
        continue
    # add the value for this coin to the total
    total += (b['balance'] + b['freezeBalance']) * rates[b['coinType']][CURRENCY]

print(total)

{% endhighlight %}

And there you have it, your balance in your currency.

### Bonus

Well seeing as this turned out to just use the Kucoin endpoints I've added a `get_total_balance` function
to [python-kucoin][python-kucoin] to do this for you.

{% highlight python %}

balance = client.get_total_balance('EUR)
print(balance)

{% endhighlight %}

### What next

Why not send your daily balance breakdown to yourself via email, or maybe a quick snapshot your own Telegram bot.

Another source of prices is [Coin Market Cap][coin-market-cap] which returns some extra information in the ticker, such as percentage changes per hour, day or week
price in BTC and even coin market cap rank. I will look at an example of this for the [Binance][binance] exchange soon.

Further extensions could include
- a total balance in BTC, well it turns out we can reuse our code from the last example.
- overall percentage change per hour, day or week.
- weighted value of our coin market cap rank.


[binance]: https://www.binance.com/en/register?ref=KNNBYWG8
[kucoin]: https://www.kucoin.com/#/?r=E42cWB
[get_klines]: https://python-binance.readthedocs.io/en/latest/binance.html#binance.client.Client.get_klines
[binance-examples]: https://github.com/sammchardy/python-binance/tree/master/examples
[python-kucoin]: https://github.com/sammchardy/python-kucoin
[coin-market-cap]: https://coinmarketcap.com
[kucoin-api-settings]: <https://www.kucoin.com/#/user/setting/api>