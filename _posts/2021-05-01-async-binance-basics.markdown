---
layout: post
title: "Async basics for Binance"
date:   2021-05-01 17:07:16 +1100
categories: binance
description: Understand Python async basics in python for Binance cryptocurrency exchange trading
img: async.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [binance, asyncio]
redirect_from:
  - /binance/2021/05/01/async-binance-basics.html
---

With the v1.0.0 release of [python-binance][python-binance] for [Binance][binance] came async functionality option for 
the REST Client as well as migrating the websocket streams and depth cache implementations to async.

The advantage of async processing is that we don't need to block on I/O which is every action that
we make when we interact with the Binance servers.

By not blocking execution we can continue processing data while we wait for responses or new data
from websockets.

### Simple async example

Save this as a python file to run.

{% highlight python %}

import asyncio

from binance import AsyncClient


async def main():

    client = await AsyncClient.create()
    exchange_info = await client.get_exchange_info()
    tickers = await client.get_all_tickers()

if __name__ == "__main__":

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

{% endhighlight %}

asyncio runs with an event loop, we call `run_until_complete` at the start on our main function. This is a 
general pattern for asyncio programs. This will finish when the main function finishes, and we can control
this especially if we are listening to websockets.

The `async` keyword in front of the function defines it as a coroutine. If you call a coroutine directly
the function isn't executed, you just get the coroutine back.

To actually execute the coroutine we use the `await` keyword, as we have done with the `get_exchange_info`
and `get_all_tickers` functions from the Binance client.

Now if we run this we do get both the exchange info and all tickers, however we haven't actually leveraged
any advantages of asyncio here at all. Each request will actually run one after the other.

So how do we improve this? We use [asyncio.gather][asyncio-gather], let's  update our main function to the following

{% highlight python %}

async def main():

    client = await AsyncClient.create()
    res = await asyncio.gather(
        client.get_exchange_info()
        client.get_all_tickers()
    )

{% endhighlight %}

How does this help? 

What we are doing here is collecting coroutines that we want executed, and then pass them together to
asyncio to execute concurrently.

`res` will be a list of responses, ordered the same as the coroutines we pass to gather.

### Adding Websockets

Making requests is great but acting on realtime information is the foundation of any bot strategy.

So how would we listen to realtime websocket data while also making API requests?

{% highlight python %}

from binance import AsyncClient, BinanceSocketManager


async def kline_listener(client):
    bm = BinanceSocketManager(client)
    async with bm.kline_socket(symbol='BNBBTC') as stream:
        while True:
            res = await stream.recv()
            print(res)

async def main():

    client = await AsyncClient.create()
    await kline_listener(client)

{% endhighlight %}

Here we import the BinanceSocketManager and add a coroutine to listen to the 1 minute BNBBTC kline stream.

You may be familiar with `with` in python and context managers, so  here we are interacting with an
asynchronous context manager.

We update the main function to call this coroutine.

When we run this we see that it doesn't exit after the first message but continues. We accomplished that
by using `while True:` within the asynchronous context manager making sure that it didn't exit.

Now we've moved from API requests to websocket listening, so let's add in an API Request.


{% highlight python %}

from binance import AsyncClient, BinanceSocketManager


async def kline_listener(client):
    bm = BinanceSocketManager(client)
    symbol = 'BNBBTC'
    res_count = 0
    async with bm.kline_socket(symbol=symbol) as stream:
        while True:
            res = await stream.recv()
            res_count += 1
            print(res)
            if res_count == 5:
                res_count = 0
                order_book = await client.get_order_book(symbol=symbol)
                print(order_book)

{% endhighlight %}

After every 5th websocket response we fetch and display the order book. This request could be placing
an order, but for our purposes the result is the same.

Now this may look like we are done, but what is actually happening here is similar to our first example.
When we fetch the order book we are actually blocking the websocket `recv` function from being called.

So what can we do here to remove this block? We can leverage the [asyncio.call_soon][asyncio-call_soon]
which schedules a coroutine to be run at the next loop interval. Which translates to as soon as possible
and it breaks us out of this current loop to avoid blocking.

{% highlight python %}

from binance import AsyncClient, BinanceSocketManager

async def order_book(client, symbol):
    order_book = await client.get_order_book(symbol=symbol)
    print(order_book)


async def kline_listener(client):
    bm = BinanceSocketManager(client)
    symbol = 'BNBBTC'
    res_count = 0
    async with bm.kline_socket(symbol=symbol) as stream:
        while True:
            res = await stream.recv()
            res_count += 1
            print(res)
            if res_count == 5:
                res_count = 0
                loop.call_soon(asyncio.create_task, order_book(client, symbol))

{% endhighlight %}

Now we can listen to the websocket, react to events and call API requests without blocking.

### Next Steps

I would recommend reading the [python asyncio][python-asyncio] docs to learn more.

Register with [Binance][binance] and try out the new functionality of [python-binance][python-binance] 
and the asyncio techniques.


[asyncio-gather]: https://docs.python.org/3/library/asyncio-task.html#asyncio.gather
[asyncio-call_soon]: https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.call_soon
[python-binance]: https://github.com/sammchardy/python-binance
[python-asyncio]: https://docs.python.org/3/library/asyncio.html

[binance]: https://www.binance.com/en/register?ref=KNNBYWG8
