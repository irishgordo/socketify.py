## The App.ws route
WebSocket "routes" are registered similarly, but not identically.

Every websocket route has the same pattern and pattern matching as for Http, but instead of one single callback you have a whole set of them, here's an example:

```python
app = App()
app.ws(
    "/*",
    {
        "compression": CompressOptions.SHARED_COMPRESSOR,
        "max_payload_length": 16 * 1024 * 1024,
        "idle_timeout": 12,
        "open": ws_open,
        "message": ws_message,
        'drain': lambda ws: print(f'WebSocket backpressure: {ws.get_buffered_amount()}'),
        "close": lambda ws, code, message: print("WebSocket closed"),
        "subscription": lambda ws, topic, subscriptions, subscriptions_before: print(f'subscription/unsubscription on topic {topic} {subscriptions} {subscriptions_before}'),
    },
)
```
## Use the WebSocket.get_user_data() feature
You should use the provided user data feature to store and attach any per-socket user data. Going from user data to WebSocket is possible if you make your user data hold the WebSocket, and hook things up in the WebSocket open handler. Your user data memory is valid while your WebSocket is.

If you want to create something more elaborate you could have the user data hold a pointer to some dynamically allocated memory block that keeps a boolean whether the WebSocket is still valid or not. Sky is the limit here.

## WebSockets are valid from open to close
All given WebSocket are guaranteed to live from open event (where you got your WebSocket) until close event is called. 
Message events will never emit outside of open/close. Calling ws.close or ws.end will immediately call the close handler.

## Backpressure in Websockets
Similarly to for Http, methods such as ws.send(...) can cause backpressure. Make sure to check ws.get_buffered_amount() before sending, and check the return value of ws.send before sending any more data. WebSockets do not have .onWritable, but instead make use of the .drain handler of the websocket route handler.

Inside of .drain event you should check ws.get_buffered_amount(), it might have drained, or even increased. Most likely drained but don't assume that it has, .drain event is only a hint that it has changed.

## Backpressure
Sending on a WebSocket can build backpressure. ws.send returns an enum of BACKPRESSURE, SUCCESS or DROPPED. When send returns BACKPRESSURE it means you should stop sending data until the drain event fires and ws.get_buffered_amount() returns a reasonable amount of bytes. But in case you specified a max_backpressure when creating the WebSocketContext, this limit will automatically be enforced. That means an attempt at sending a message which would result in too much backpressure will be canceled and send will return DROPPED. This means the message was dropped and will not be put in the queue. max_backpressure is an essential setting when using pub/sub as a slow receiver otherwise could build up a lot of backpressure. By setting max_backpressure the library will automatically manage an enforce a maximum allowed backpressure per socket for you.

## Ping/pongs "heartbeats"
The library will automatically send pings to clients according to the idle_timeout specified. If you set idle_timeout = 120 seconds a ping will go out a few seconds before this timeout unless the client has sent something to the server recently. If the client responds to the ping, the socket will stay open. When client fails to respond in time, the socket will be forcefully closed and the close event will trigger. On disconnect all resources are freed, including subscriptions to topics and any backpressure. You can easily let the browser reconnect using 3-lines-or-so of JavaScript if you want to.


## Settings
Compression (permessage-deflate) has three main modes; CompressOptions.DISABLED, CompressOptions.SHARED_COMPRESSOR and any of the CompressOptions.DEDICATED_COMPRESSOR_xKB. Disabled and shared options require no memory, while dedicated compressor requires the amount of memory you selected. For instance, CompressOptions.DEDICATED_COMPRESSOR_4KB adds an overhead of 4KB per WebSocket while uCompressOptions.DEDICATED_COMPRESSOR_256KB adds - you guessed it - 256KB!

Compressing using shared means that every WebSocket message is an isolated compression stream, it does not have a sliding compression window, kept between multiple send calls like the dedicated variants do.

You probably want shared compressor if dealing with larger JSON messages, or 4kb dedicated compressor if dealing with smaller JSON messages and if doing binary messaging you probably want to disable it completely.

idle_timeout is roughly the amount of seconds that may pass between messages. Being idle for more than this, and the connection is severed. This means you should make your clients send small ping messages every now and then, to keep the connection alive. The server will automatically send pings in case it needs to.

### Next [Plugins / Extensions](extensions.md)