---
title: WebSocket streaming
---

# WebSocket streaming

The WebSocket API is the most efficient (speed, latency, resource consumption, and bandwidth) way to receive quotes for instruments across all exchanges during live market hours. A quote consists of fields such as open, high, low, close, last traded price, 5 levels of bid/offer market depth data etc.

In addition, the text messages, alerts, and order updates (the same as the ones available as Postbacks) are also streamed. As the name suggests, the API uses WebSocket protocol to establish a single long standing TCP connection after an HTTP handshake to receive streaming quotes. To connect to the GoPocket WebSocket API, you will need a WebSocket client library in your choice of programming language.

You can subscribe for up to 3000 instruments on a single WebSocket connection and receive live quotes for them. Single API key can have upto 3 websocket connections.

!!! Note

Implementing an asynchronous WebSocket client with a binary parser for the market data structure may be a complex task. We recommend using one of our pre-built client libraries.

## Connecting to the WebSocket endpoint

```

// Javascript example.
var ws = new WebSocket("wss://ch1.GoPocket.com/NorenWS/");

```

The WebSocket endpoint is wss://ch1.GoPocket.com/NorenWS/. To establish a connection, you have to pass two query parameters, `api_key` and `access_token`.

### Response Structure

```
// Subscribe to quotes for INFY (408065) and TATAMOTORS (884737)
var message = { a: "subscribe", v: [408065, 884737] };
ws.send(JSON.stringify(message));

```

Requests are simple JSON messages with two parameters, a (action) and v (value). Following are the available actions and possible values. Many values are arrays, for instance, array of instrument_token that can be passed to subscribe to multiple instruments at once.

**Parameters**

| a             | v                               |
| ------------- | ------------------------------- |
| `subscribe`   | [instrument_token ... ]         |
| `unsubscribe` | [instrument_token ... ]         |
| `mode`        | [mode, [instrument_token ... ]] |

```
// Set INFY (408065) to 'full' mode to
// receive market depth as well.
message = { a: "mode", v: ["full", [408065]] };
ws.send(JSON.stringify(message));

// Set TATAMOTORS (884737) to 'ltp' to only receive the LTP.
message = { a: "mode", v: ["ltp", [884737]] };
ws.send(JSON.stringify(message));

```

### Modes

There are three different modes in which quote packets are streamed.

| mode    |                                                                          |
| ------- | ------------------------------------------------------------------------ |
| `ltp`   | LTP. Packet contains only the last traded price (8 bytes).               |
| `quote` | Quote. Packet contains several fields excluding market depth (44 bytes)  |
| `full`  | Full. Packet contains several fields including market depth (184 bytes). |

!!! Note

Always check the type of an incoming WebSocket messages. Market data is always binary and Postbacks and other updates are always text.

```
If there is no data to be streamed over an open WebSocket connection, the API will send a 1 byte "heartbeat" every couple seconds to keep the connection alive. This can be safely ignored.

```

## Binary market data

WebSocket supports two types of messages, binary and text.

Quotes delivered via the API are always binary messages. These have to be read as bytes and then type-casted into appropriate quote data structures. On the other hand, all requests you send to the API are JSON messages, and the API may also respond with non-quote, non-binary JSON messages, which are described in the next section.

For quote subscriptions, instruments are identified with their corresponding numerical instrument_token obtained from the instrument list API.

## Message structure

Each binary message (array of 0 to n individual bytes)--or frame in WebSocket terminology--received via the WebSocket is a combination of one or more quote packets for one or more instruments. The message structure is as follows.

![Image](assets/images/websocket-message.png)

|     |                                                                                                                   |
| --- | ----------------------------------------------------------------------------------------------------------------- |
| `A` | The first two bytes ([0 - 2] -- SHORT or int16) represent the number of packets in the message.                   |
| `B` | The next two bytes ([2 - 4] -- SHORT or int16) represent the length (number of bytes) of the first packet.        |
| `C` | The next series of bytes ([4 - 4+B]) is the quote packet.                                                         |
| `D` | The next two bytes ([4+B - 4+B+2] -- SHORT or int16) represent the length (number of bytes) of the second packet. |
| `C` | The next series of bytes ([4+B+2 - 4+B+2+D]) is the next quote packet.                                            |

### Quote packet structure

Each individual packet extracted from the message, based on the structure shown in the previous section, can be cast into a data structure as follows. All prices are in paise. For currencies, the int32 price values should be divided by 10000000 to obtain four decimal plaes. For everything else, the price values should be divided by 100.

| Bytes    | Type   |                                                          |
| -------- | ------ | -------------------------------------------------------- | ----------- |
| 0 - 4    | int32  | instrument_token                                         |
| 4 - 8    | int32  | Last traded price (If mode is ltp, the packet ends here) |
| 8 - 12   | int32  | Last traded quantity                                     |
| 12 - 16  | int32  | Average traded price                                     |
| 16 - 20  | int32  | Volume traded for the day                                |
| 20 - 24  | int32  | Total buy quantity                                       |
| 24 - 28  | int32  | Total sell quantity                                      |
| 28 - 32  | int32  | Open price of the day                                    |
| 32 - 36  | int32  | High price of the day                                    |
| 36 - 40  | int32  | Low price of the day                                     |
| 40 - 44  | int32  | Close price (If mode is quote, the packet ends here)     |
| 44 - 48  | int32  | Last traded timestamp                                    |
| 48 - 52  | int32  | Open Interest                                            |
| 52 - 56  | int32  | Open Interest Day High                                   |
| 56 - 60  | int32  | Open Inte                                                | est Day Low |
| 60 - 64  | int32  | Exchange timestamp                                       |
| 64 - 184 | []byte | Market depth entries                                     |

### Index packet structure

The packet structure for indices such as NIFTY 50 and SENSEX differ from that of tradeable instruments. They have fewer fields.

| Bytes   | Type  |                                                       |
| ------- | ----- | ----------------------------------------------------- |
| 0 - 4   | int32 | Token                                                 |
| 4 - 8   | int32 | Last traded price                                     |
| 8 - 12  | int32 | High of the day quantity                              |
| 12 - 16 | int32 | Low of the day                                        |
| 16 - 20 | int32 | Open of the day                                       |
| 20 - 24 | int32 | Close of the day                                      |
| 24 - 28 | int32 | Price change (If mode is quote, the packet ends here) |
| 28 - 32 | int32 | Exchange timestamp                                    |

### Market depth structure

Each market depth entry is a combination of 3 fields, quantity (int32), price (int32), orders (int16) and there is a 2 byte padding at the end (which should be skipped) totalling to 12 bytes. There are ten entries in succession—five [64 - 124] bid entries and five [124 - 184] offer entries.

## Postbacks and non-binary updates

Apart from binary market data, the WebSocket stream delivers postbacks and other updates in the text mode. These messages are JSON encoded and should be parsed on receipt. For order Postbacks, the payload is contained in the data key and has the same structure described in the Postbacks section.

**Message Structure**

```
{
  "type": "order",
  "data": {}
}

```

## Message types

| Type      |                                                                                     |
| --------- | ----------------------------------------------------------------------------------- |
| `order`   | Order Postback. The data field will contain the full order Postback payload         |
| `error`   | Error responses. The data field contain the error string                            |
| `message` | Messages and alerts from the broker. The data field will contain the message string |
