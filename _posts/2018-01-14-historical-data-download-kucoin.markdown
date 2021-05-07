---
layout: post
title: "Save Historical data from Kucoin"
date:   2018-01-14 17:03:16 +1100
categories: kucoin
description: Download and save historical data from the Kucoin API over a given timeframe
img: kucoin-history.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [kucoin, klines, backtesting]
redirect_from:
  - /kucoin/2018/01/14/historical-data-download-kucoin.html
---
The basis of any trading strategy is having a good backtesting solution, and you can't backtest unless you have data.

Similar to the previous post on binance I'm going to step through how to download and save historical data from the Kucoin API over a given timeframe.

This example does not require an account on [Kucoin][kucoin] as it utilises public API calls.

### Working with dates

It's convenient to be able to pass human readable dates, unfortunately the Kucoin server only understands unix timestamps in seconds
so we need to convert our date.

This can be done by using helper websites like [current millis][current-millis] but we've got the power of python so let's use it.

We'll use the helper function we created in a previous blog for Binance timestamps and modify to return seconds.

{% highlight python %}

def date_to_seconds(date_str):
    """Convert UTC date to seconds

    If using offset strings add "UTC" to date string e.g. "now UTC", "11 hours ago UTC"

    See dateparse docs for formats http://dateparser.readthedocs.io/en/latest/

    :param date_str: date in readable format, i.e. "January 01, 2018", "11 hours ago UTC", "now UTC"
    :type date_str: str
    """
    # get epoch value in UTC
    epoch = datetime.utcfromtimestamp(0).replace(tzinfo=pytz.utc)
    # parse our date string
    d = dateparser.parse(date_str)
    # if the date is not timezone aware apply UTC timezone
    if d.tzinfo is None or d.tzinfo.utcoffset(d) is None:
        d = d.replace(tzinfo=pytz.utc)

    # return the difference in time
    return int((d - epoch).total_seconds())

{% endhighlight %}

With this function we can convert all sorts of handy date formats to seconds like the examples below.

{% highlight python %}

print(date_to_seconds("January 01, 2018"))
print(date_to_seconds("11 hours ago UTC"))
print(date_to_seconds("now UTC"))

{% endhighlight %}

### Kucoin intervals

Kucoin has 2 kline functions, their own and a Trading View version each having different output formats and parameters.

For now I'm just going to focus on the Trading View version call. As this endpoint returns every kline you ask for by default
we don't have to create a loop or worry about convert our intervals to seconds which removes a lot of extra work.

The Trading View kline functions has the following parameters

- `symbol` - e.g ETHBTC
- `interval` - one of (1, 5, 15, 30, 60, 480, D, W)
- `from` - timestamp in seconds
- `to` - timestamp in seconds

The `to` parameter is required, so we need to make sure that we pass a value for the time now if we want everything.

The [get_kline_data_tv][get_kline_data_tv] endpoint returns each individual value we are after in a list of its own,
so we get all the opens in a list, all closes in a list and so on for high, low and timestamp.

I don't find this very helpful to work with so let's return it in a more standard list in the OHLCV format.

### Fetch the Klines already

Ok let's get down to it, we're ready to build our function to fetch historical data and massage it into a more useful format.

{% highlight python %}

def get_historical_klines_tv(symbol, interval, start_str, end_str=None):
    """Get Historical Klines from Kucoin (Trading View)

    See dateparse docs for valid start and end string formats http://dateparser.readthedocs.io/en/latest/

    If using offset strings for dates add "UTC" to date string e.g. "now UTC", "11 hours ago UTC"

    :param symbol: Name of symbol pair e.g BNBBTC
    :type symbol: str
    :param interval: Trading View Kline interval
    :type interval: str
    :param start_str: Start date string in UTC format
    :type start_str: str
    :param end_str: optional - end date string in UTC format
    :type end_str: str

    :return: list of OHLCV values

    """

    # init our array for klines
    klines = []
    client = Client("", "")

    # convert our date strings to seconds
    start_ts = date_to_seconds(start_str)

    # if an end time was not passed we need to use now
    if end_str is None:
        end_str = 'now UTC'
    end_ts = date_to_seconds(end_str)

    kline_res = client.get_kline_data_tv(symbol, interval, start_ts, end_ts)

    print(kline_res)

    # check if we got a result
    if 't' in kline_res and len(kline_res['t']):
        # now convert this array to OHLCV format and add to the array
        for i in range(1, len(kline_res['t'])):
            klines.append((
                kline_res['t'][i],
                kline_res['o'][i],
                kline_res['h'][i],
                kline_res['l'][i],
                kline_res['c'][i],
                kline_res['v'][i]
            ))

    # finally return our converted klines
    return klines

{% endhighlight %}

With this function we have a really simple way of fetching a list of klines using simple to use dates and intervals.

{% highlight python %}

# fetch 1 minute klines for the last day up until now
klines = get_historical_klines_tv("KCS-BTC", "1", "1 day ago UTC")

# fetch 30 minute klines for the last month of 2017
klines = get_historical_klines_tv("NEO-BTC", "30", "1 Dec, 2017", "1 Jan, 2018")

# fetch weekly klines since it listed
klines = get_historical_klines_tv("XRP-BTC", "W", "1 Jan, 2017")

{% endhighlight %}

### Save to file

Once we have fetched the list of klines, it makes sense to save them to a file for later use.

{% highlight python %}

import json

symbol = "KCS-BTC"
start = "1 Dec, 2017"
end = "1 Jan, 2018"
interval = "30"

klines = get_historical_klines_tv(symbol, interval, start, end)

# open a file with filename including symbol, interval and start and end converted to seconds
with open(
    "Kucoin_{}_{}_{}-{}.json".format(
        symbol, 
        interval, 
        date_to_seconds(start),
        date_to_seconds(end)
    ),
    'w' # set file write mode
) as f:
    f.write(json.dumps(klines))
            
{% endhighlight %}

### Bonus

Well as with the Binance example I may as well add these functions to the [python-kucoin][python-kucoin] library for everyone to use.

`date_to_seconds` has been added to the new kucoin.helpers file.

`get_historical_klines_tv` has been added to the kucoin.client, so now all you need to do is run

{% highlight python %}

import json
from kucoin.client import Client

client = Client("", "")

klines = client.get_historical_klines_tv("KCS-BTC", Client.RESOLUTION_30MINUTES, "1 Dec, 2017", "1 Jan, 2018")
            
{% endhighlight %}

### Next Steps

With these cached klines we can open them at a later date and run backtesting on them.

See the related [Binance post][binance-post] to download historical klines on [Binance][binance].

Subsequent posts will look at how to use [pandas][pandas] and [TA-Lib][ta-lib] for some simple backtesting.

[current-millis]: https://currentmillis.com/
[kucoin]: https://www.kucoin.com/#/?r=E42cWB
[binance]: https://www.binance.com/en/register?ref=KNNBYWG8
[binance-post]: https://sammchardy.github.io/binance/2018/01/08/historical-data-download-binance.html
[get_kline_data_tv]: https://python-kucoin.readthedocs.io/en/latest/market.html
[kucoin-examples]: https://github.com/sammchardy/python-kucoin/tree/master/examples
[python-kucoin]: https://github.com/sammchardy/python-kucoin
[pandas]: https://pandas.pydata.org/
[ta-lib]: https://github.com/mrjbq7/ta-lib
