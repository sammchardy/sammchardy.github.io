---
layout: post
title: "Save Historical data from Binance"
date:   2018-01-08 17:07:16 +1100
categories: binance
description: In this post I'm going to step through how to download and save historical data from the Binance API over a given timeframe
img: binance-history.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [binance, klines, backtesting]
redirect_from:
  - /binance/2018/01/08/historical-data-download-binance.html
---
The basis of any trading strategy is having a good backtesting solution, and you can't backtest unless you have data.

In this post I'm going to step through how to download and save historical data from the Binance API over a given timeframe.

This example does not require an account on [Binance][binance] as it utilises public API calls.

### Working with dates

It's convenient to be able to pass human readable dates, unfortunately the Binance server only understands millisecond timestamps 
so we need to convert our date.

This can be done by using helper websites like [current millis][current-millis] but we've got the power of python so let's use it.

First install the `dateparser` package

{% highlight bash %}
pip install dateparser
{% endhighlight %}

Now we can create a function to take a readable string into milliseconds. 

{% gist fcbb2b836d1f694f39bddd569d1c16fe %}

With this function we can convert all sorts of handy date formats to milliseconds like the examples below.

{% highlight python %}

print(date_to_milliseconds("January 01, 2018"))
print(date_to_milliseconds("11 hours ago UTC"))
print(date_to_milliseconds("now UTC"))

{% endhighlight %}

### Binance Kline endpoint

Now we have that out of the way we can start to work with the Binance API. 
For our purposes we are interested in the [get_klines][get_klines] endpoint to fetch the actual data.

This takes parameters

- `symbol` - e.g ETHBTC
- `interval` - one of (1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 3d, 1w, 1M)
- `limit` - max 500
- `startTime` - milliseconds
- `endTime` - milliseconds

`symbol` and `limit` are straightforward, `startTime` and `endTime` we now have covered by the above `date_to_milliseconds` function.

We note from this that we can get a maximum of 500 results each request, so we will need to build a loop if we are fetching over a long time period.

`interval` we have a list of so these are passed easily.

The [get_klines][get_klines] endpoint returns an array of klines in this order

{% highlight python %}

  [
    1499040000000,      # Open time
    "0.01634790",       # Open
    "0.80000000",       # High
    "0.01575800",       # Low
    "0.01577100",       # Close
    "148976.11427815",  # Volume
    1499644799999,      # Close time
    "2434.19055334",    # Quote asset volume
    308,                # Number of trades
    "1756.87402397",    # Taker buy base asset volume
    "28.46694368",      # Taker buy quote asset volume
    "17928899.62484339" # Ignore
  ]
  
{% endhighlight %}

We are mostly interested in the open time, open, high, low, close and volume values. But why not save all the info while we have it.

### Binance Intervals

The `interval` is a string which the API understands, but for our purposes we will need to convert it to a millisecond representation to help with our loop.
We will ignore "M" or month in this case as it's tricky to determine how long a month is and given that we will fetch 500 results and Binance has only  been running for a year we 
can fetch all "1M" results in one request.

{% gist 3547cfab1faf78e385b3fcb83ad86395 %}

With this function we can pass any of the Binance interval values and receive the corresponding value in milliseconds.

{% highlight python %}

from binance.client import Client

print(interval_to_milliseconds(Client.KLINE_INTERVAL_1MINUTE))
print(interval_to_milliseconds(Client.KLINE_INTERVAL_30MINUTE))
print(interval_to_milliseconds(KLINE_INTERVAL_1WEEK))

{% endhighlight %}

### Fetch the Klines already

Ok enough messing around, we're ready to build our function to fetch historical data.

{% gist 0c740c40276e8f05b6390ce304476605 %}

With this function we have a really simple way of fetching a list of klines using simple to use dates and intervals.

{% highlight python %}

from binance.client import Client

# fetch 1 minute klines for the last day up until now
klines = get_historical_klines("BNBBTC", Client.KLINE_INTERVAL_1MINUTE, "1 day ago UTC")

# fetch 30 minute klines for the last month of 2017
klines = get_historical_klines("ETHBTC", Client.KLINE_INTERVAL_30MINUTE, "1 Dec, 2017", "1 Jan, 2018")

# fetch weekly klines since it listed
klines = get_historical_klines("NEOBTC", KLINE_INTERVAL_1WEEK, "1 Jan, 2017")

{% endhighlight %}

The full code can be found in the [examples folder][binance-examples] of the [python-binance][python-binance] package. 

### Save to file

Once we have fetched the list of klines, it makes sense to save them to a file for later use.

{% highlight python %}

import json
from binance.client import Client

symbol = "ETHBTC"
start = "1 Dec, 2017"
end = "1 Jan, 2018"
interval = Client.KLINE_INTERVAL_30MINUTE

klines = get_historical_klines(symbol, interval, start, end)

# open a file with filename including symbol, interval and start and end converted to milliseconds
with open(
    "Binance_{}_{}_{}-{}.json".format(
        symbol, 
        interval, 
        date_to_milliseconds(start),
        date_to_milliseconds(end)
    ),
    'w' # set file write mode
) as f:
    f.write(json.dumps(klines))
            
{% endhighlight %}

### Bonus

Well after all that I thought I may as well add these functions to the [python-binance][python-binance] library for everyone to use.

`date_to_milliseconds` and `interval_to_milliseconds` have been added to the new binance.helpers file.

`get_historical_klines` has been added to the binance.client, so now all you need to do is run


{% highlight python %}

import json
from binance.client import Client

client = Client("", "")

klines = client.get_historical_klines("ETHBTC", Client.KLINE_INTERVAL_30MINUTE, "1 Dec, 2017", "1 Jan, 2018")
            
{% endhighlight %}

### Next Steps

With these cached klines we can open them at a later date and run backtesting on them.

See the related [Kucoin post][kucoin-post] to download historical klines on [Kucoin][kucoin].


Subsequent posts will look at how to use [pandas][pandas] and [TA-Lib][ta-lib] for some simple backtesting.

[current-millis]: https://currentmillis.com/
[binance]: https://www.binance.com/en/register?ref=KNNBYWG8
[kucoin]: https://www.kucoin.com/#/?r=E42cWB
[get_klines]: https://python-binance.readthedocs.io/en/latest/binance.html#binance.client.Client.get_klines
[binance-examples]: https://github.com/sammchardy/python-binance/tree/master/examples
[python-binance]: https://github.com/sammchardy/python-binance
[pandas]: https://pandas.pydata.org/
[ta-lib]: https://github.com/mrjbq7/ta-lib