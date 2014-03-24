# WebSocket over HTTP/2.0

**This is a spec draft of the WebSocket over HTTP/2.0.
Unlike [another one](https://github.com/yutakahirano/ws-over-http2/blob/master/draft-hirano-websocket-over-http2.txt) this protocol doesn't respect [RFC6455](https://tools.ietf.org/html/rfc6455) framing.**

##Introduction
The WebSocket protocol was standardized to enable efficient bidirectional messaging mainly for browsers.
However, the core spec in RFC 6455 left one problem about scalability unaddressed.
That is that one WebSocket connection uses one TCP connection.
Use of multiple WebSocket connections provides flexibility for web apps, while using more TCP connections leads to more load to the end hosts and also to network intermediaries.

For the HTTP/1.1, there has been effort to multiplex HTTP traffic into one TCP connection called HTTP/2.0.
The HTTP/2.0 defines a general multiplexed transport on which not only HTTP but other messaging application protocol may be layered onto.
We can address the scalability issue of WebSocket by using HTTP/2.0 framing's multiplexing functionality.

In this document, we describe how to layer WebSocket semantics onto HTTP/2.0 semantics.

##Opening Handshake
Same as https://github.com/yutakahirano/ws-over-http2/blob/master/draft-hirano-websocket-over-http2.txt.

##WebSocket message
There are three kinds of WebSocket message: Text, Binary and Close.

A Text message represents a text.
A Binary message represents an arbitrary octet sequence.
A Close message consists of code and reason defined in [the WebSocket API](http://www.w3.org/TR/websockets/).

### Message representation
A WebSocket message consists of a HEADERS frame and subsequent possibly multiple DATA frames.
END_SEGMENT flag MUST be set at the last frame of a WebSocket message.
That is, WebSocket over HTTP/2.0 defines _segment_ in [HTTP/2.0] as _WebSocket message_.
END_STREAM flag MUST be set on the last frame of a Close message and MUST NOT be set on any other frames.
Note that it is possible to create a message having no payload data.
In such a case, END_SEGMENT flag MUST be set on the HEADERS frame.

The HEADERS frame stores message meta information.

The HEADERS frame MUST include the following headers.

 - :opcode This header specifies the kind of the message.
  - 1: Text
  - 2: Binary
  - 8: Close

An extension MAY insert other headers.

#### Text
A Text WebSocket message represents a text.
Without an extension, the utf-8 encoded text is stored in DATA frames in order.

An extension MAY modify the payload arbitrarily.

#### Binary
A Binary WebSocket message represents an octet sequence.
Without an extension, the sequence is stored in DATA frames in order.

An extension MAY modify the payload arbitrarily.

#### Close
A Close WebSocket message is used for the WebSocket closing handshake.

The HEADERS frame MUST include the following headers.

 - :code This header specifies the code in WebSocket closing handshake.

A Close WebSocket message MAY have the reason text as the payload data.
The reason text MUST be a valid utf-8 text.
The reason text MUST be of length at most 126 bytes.

END_STREAM flag MUST be set on the last frame of the message.

## Closing the Connection

### Definitions
#### Close the WebSocket Connection
To _Close the WebSocket Connection_, an endpoint sends an RST_STREAM frame to the peer if it is not yet closed.

#### Start the WebSocket Closing Handshake
To _Start the WebSocket Closing Handshake_ with a status code /code/ and an optional close reason /reason/, an endpoint MUST send a Close message, as described above, whose status code is set to /code/ and whose close reason is set to /reason/.

#### The WebSocket Closing Handshake is Started
Upon either sending or receiving a Close message, it is said that _The WebSocket Closing Handshake is Started_.
An endpoint MUST discard received Text or Binary messages once the endpoint sent a Close message.

Note: When an endpoint receives a Close message during sending a Text or Binary message, it MAY finish the message (i.e. send a DATA frame with no payload and END_SEGMENT set) immediately because the peer MUST discard it.
Then the endpoint can send a Close message.

#### The WebSocket Connection is Closed
When the underlying HTTP/2.0 stream is closed, it is said that _The WebSocket Connection is Closed_.
If the stream was closed by a HTTP/2.0 frame with END_STREAM set (i.e. by a WebSocket Close message), the WebSocket connection is said to have been closed _cleanly_.

If the WebSocket connection could not be established, it is also said that _The WebSocket Connection is Closed_, but not _cleanly_.
