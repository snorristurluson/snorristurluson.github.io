In a [previous blog](https://ccpsnorlax.blogspot.is/2017/09/working-with-xmpp-in-python.html) I started discussing Xmpp
and showed how to set up an Xmpp server and connecting to it via Python. In
this blog I will dig deeper and show how to implement a simple echo bot.

The code for this lives on Github: [https://github.com/snorristurluson/xmpp-chatbot]()

## Connecting

First, let's wrap the network layer. I've picked the Python 3 [asyncio](https://docs.python.org/3/library/asyncio.html)
for this task.

Let's start by looking at *firstconnection.py*. I've created a class called 
*FirstConnection* that inherits from *asyncio.Protocol*.

```python
class FirstConnection(asyncio.Protocol):
    def __init__(self, host):
        self.host = host
        self.transport = None

    def connect(self):
        loop = asyncio.get_event_loop()
        handler = loop.create_connection(lambda: self, self.host, 5222)
        loop.create_task(handler)

    def connection_made(self, transport):
        logger.debug("Connection made")
        self.transport = transport

        cmd = "<?xml version='1.0'?><stream:stream to='localhost' " \
              "version='1.0' xmlns='jabber:client' " \
              "xmlns:stream='http://etherx.jabber.org/streams'>"
        self.write(cmd)

        cmd = "<junk/>"
        self.write(cmd)

    def connection_lost(self, exc):
        logger.debug("Connection lost")

    def write(self, data):
        logger.debug("Send: %s", data)
        self.transport.write(data.encode())

    def data_received(self, data):
        logger.debug("Received %d bytes\n%s", len(data), data.decode())
```

Let's instantiate this class and connect:
```python
    loop = asyncio.get_event_loop()

    bot = FirstConnection("localhost")
    bot.connect()
    loop.run_forever()
```

If the connect call fails, make sure you have Prosody running. If it is, you
should see output similar to this:

```
DEBUG:__main__:FirstConnection is starting
DEBUG:asyncio:Using selector: KqueueSelector
DEBUG:__main__:Connection made
DEBUG:__main__:Send: <?xml version='1.0'?><stream:stream to='localhost' version='1.0' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams'>
DEBUG:__main__:Send: <junk/>
DEBUG:__main__:Received 605 bytes
<?xml version='1.0'?><stream:stream xmlns:stream='http://etherx.jabber.org/streams' version='1.0' from='localhost' id='743f47de-6f34-4458-afcc-558f872b949a' xml:lang='en' xmlns='jabber:client'><stream:features><starttls xmlns='urn:ietf:params:xml:ns:xmpp-tls'/><mechanisms xmlns='urn:ietf:params:xml:ns:xmpp-sasl'><mechanism>PLAIN</mechanism><mechanism>SCRAM-SHA-1</mechanism><mechanism>DIGEST-MD5</mechanism></mechanisms><auth xmlns='http://jabber.org/features/iq-auth'/></stream:features><stream:error><unsupported-stanza-type xmlns='urn:ietf:params:xml:ns:xmpp-streams'/></stream:error></stream:stream>
DEBUG:__main__:Connection lost
```

This is a small step, but an important one. We're talking to the server
and have initiated the handshake. The server sent us back its part of the
handshake - then we sent it junk and it slammed the door on us. The important
bit is that we're doing this in a more managed fashion than before and this
allows us to build something useful on top.

## Handling XMPP
Before we can actually log in we need to think about how to process the
incoming XMPP commands. XMPP is XML based so it makes sense to use an XML
parser, rather than working with the raw text.

Python offers several [different options](https://docs.python.org/3/library/xml.html)
for parsing XML, but keeping in mind that we are not dealing with a full XML
document, the [xml.sax](https://docs.python.org/3/library/xml.sax.html#module-xml.sax)
module seems appropriate.

The connection to the XMPP server is stateful - the server expects an initial
handshake at the beginning with the right commands being issued in the right
order. To manage this, I use a class named *XmppHandler*. It holds an instance
of an xml.sax parser, on which I set the
[content handler](https://docs.python.org/3/library/xml.sax.handler.html#xml.sax.handler.ContentHandler)
to an *XmppContentHandler*.

The *XmppContentHandler* responds to the incoming parser events and builds
a queue of *XmppElement* objects. The *XmppHandler* pulls from this queue,
allowing it to work on a somewhat higher level, with whole XMPP stanzas.

Data received from the server is sent to the *handle_raw_response* method of
the *XmppHandler*, which passes it on to the *feed* method of the parser.
Once the parser has been fed, I pull the elements out of the queue,
processing them one by one until the queue is empty.

## Logging in
The processing of elements depends on the state of the handler. The state
machine is not implemented in any formal way - it's simply a state attribute
and a series of *if* statements - crude but effective.

The *ConnectBot* object in *connectbot.py* takes us a step further. If you run
it, you should see it connecting. Note that you need to add an *echobot*
user on the server before it lets you log in. The password should be *echobot*
as well.

```
prosodyctl adduser echobot@localhost
```

Once you have added the user, you should see the bot successfully logging
in. The log output from *connectbot.py* shows you all the XMPP commands
that flow between the bot an the server, and the Prosody logs show you
the progress as well.

## Echo, echo
Finally we can start doing something useful (by some definition of useful).
The *EchoBot* in *echobot.py* adds the *handle_xmpp_message* method.
I monkey-patch the handler so that it calls this method whenever a
*message* stanza is received. This method simply pulls the body text
from the XMPP element and sends that back to the sender as a message.

Try this out with some XMPP chat client - I've been using
[Swift](https://swift.im/swift.html). Start a chat with *echobot@localhost*
and you should see everything you say echoed back.

## What next?
The hard part is done now - building on this framework we can start
focusing on the bots themselves. More on that later!