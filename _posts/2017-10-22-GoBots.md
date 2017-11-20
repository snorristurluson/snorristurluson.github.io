Earlier this year I started experimenting with the 
[Xmpp](https://en.wikipedia.org/wiki/XMPP) 
[protocol](https://xmpp.org/),
and implemented bots in Python to communicate with an Xmpp server. I've now
revisited those bots and reimplemented them in [Go](https://golang.org/). 
I've been meaning to learn Go for quite a while, and this seemed like a reasonable 
first project to tackle.

The source code lives on GitHub: [https://github.com/snorristurluson/xmpp-chatbot]()

If you are an experienced Go developer, I would appreciate any feedback and suggestions
on how to improve the code - if you are just starting out with Go like myself, I hope
this blog and the code is useful to you.

As [before](https://ccpsnorlax.blogspot.is/2017/09/working-with-xmpp-in-python.html), 
I'm using [Prosody](https://prosody.im/) to communicate with. Just like I did in
Python, my first experiment is to simply open up a socket and poke the server to
see where that takes us:

```go
func main() {
    conn, err := net.Dial("tcp", "localhost:5222")
    if err != nil {
        fmt.Errorf("Couldn't connect")
    }
    message := "<?xml version='1.0'?>" +
          "<stream:stream to='localhost' version='1.0' " +
          "xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams'>"
    conn.Write([]byte(message))

    recvBuf := make([]byte, 4096)
    conn.Read(recvBuf)
    fmt.Print("Message Received: ", string(recvBuf))
}
```

The output should look something like this (minus the formatting):
```
Message Received: <?xml version='1.0'?>
<stream:stream xmlns:stream='http://etherx.jabber.org/streams' 
    version='1.0' 
    from='localhost' 
    id='6f6e50cd-2bde-4424-8418-ccc935835ee3' 
    xml:lang='en' 
    xmlns='jabber:client'>
<stream:features>
    <mechanisms xmlns='urn:ietf:params:xml:ns:xmpp-sasl'>
        <mechanism>PLAIN</mechanism>
        <mechanism>SCRAM-SHA-1</mechanism>
        <mechanism>DIGEST-MD5</mechanism>
    </mechanisms>
    <auth xmlns='http://jabber.org/features/iq-auth'/>
    <starttls xmlns='urn:ietf:params:xml:ns:xmpp-tls'/>
</stream:features>
```
This is the start of the handshake of Xmpp - I won't go into the details here
as I've [already covered it](https://ccpsnorlax.blogspot.is/2017/09/working-with-xmpp-in-python.html) 
in the context of Python, but this is a good start.

In my blog post on the [Echobot in Python](https://ccpsnorlax.blogspot.is/2017/10/echobot.html)
I discussed how to handle the incoming Xmpp stanzas. I'm basically using
the same approach in Go, although I have to resort to some tricks to
handle the incomplete XML we're bound to get when feeding the incoming
stream.

Just like in my Python implementation the *XmppContentHandler* responds to 
the incoming parser events and builds a queue of *XmppElement* objects. 
The *XmppHandler* pulls from this queue, allowing it to work on a somewhat 
higher level, with whole XMPP stanzas.

An *XmppElement* looks like this:
```go
type XmppElement struct {
    Tag        string
    Namespace  string
    Attributes map[string]string
    Children   []*XmppElement
    Text       string
}
```
The Parse method of *XmppContentHandler* uses the 
[xml Decoder](https://golang.org/pkg/encoding/xml/#Decoder). It pulls
tokens from the parser until an error is encountered. The handler
keeps track of any leftovers, combining them with incoming data so
eventually it becomes a whole stanza that then gets added to the queue.
```go
func (handler *XmppContentHandler) Parse(input string) (err error) {
    buffer := new(bytes.Buffer)
    parser := xml.NewDecoder(buffer)
    var current_input = handler.leftovers + input
    buffer.WriteString(current_input)
    for {
        var token xml.Token
        token, err = parser.Token()
        if err == io.EOF {
            err = nil
            break
        }
        if err != nil && strings.HasSuffix(err.Error(), "unexpected EOF") {
            // Restore State to last know good
            handler.current_element = handler.prev_current_element
            handler.queue = handler.prev_queue
            handler.element_stack = handler.prev_element_stack
            handler.leftovers = current_input

            err = nil
            break
        }
        if err != nil {
            break
        }

        if handler.element_stack == nil {
            handler.element_stack = list.New()
        }

        switch token.(type)  {
        case xml.StartElement:
            start := token.(xml.StartElement)
            element := XmppElement{
                Tag:        start.Name.Local,
                Namespace:  start.Name.Space,
                Attributes: map[string]string{},
            }
            for _, attribute := range start.Attr  {
                element.Attributes[attribute.Name.Local] = attribute.Value
            }
            if element.Tag == "stream" {
                // This element won't close until stream is closed
                handler.queue = append(handler.queue, &element)
                continue
            }
            handler.element_stack.PushFront(handler.current_element)
            handler.current_element = &element

        case xml.EndElement:
            if handler.current_element == nil {
                err = errors.New("No current element")
                return
            }
            parentListNode := handler.element_stack.Front()
            handler.element_stack.Remove(parentListNode)
            parent := parentListNode.Value.(*XmppElement)
            current_element := handler.current_element
            handler.current_element = parent
            if parent != nil {
                parent.Children = append(parent.Children, current_element)
            } else {
                handler.queue = append(handler.queue, current_element)

                // We've just completed a stanza - store the current State
                // so we can handle incomplete xml snippets
                handler.prev_current_element = handler.current_element
                handler.prev_queue = handler.queue
                handler.prev_element_stack = handler.element_stack

                current_input, _ = buffer.ReadString(0)
                buffer.WriteString(current_input)
            }

        case xml.CharData:
            if handler.current_element != nil {
                handler.current_element.Text = string(token.(xml.CharData))
            }
        }
    }

    return err
}
```
Incidentally, this was one of those times were a TDD approach worked
beautifully. Go has excellent support for unit tests, and working
in [PyCharm](https://www.jetbrains.com/pycharm/) with their Go
plugin is a joy. 

## XmppHandler
The *XmppHandler* manages the connection to the Xmpp server and
processes the incoming stanzas. It manages the initial handshake,
then issues callbacks to handle messages once it's in the ready state.

The *Connect* method sets up the connection, sends the first part
of the handshake and starts a Go routine to handle the incoming
stream.

```go
func (self *XmppHandler) Connect(server string, host string, username string, password string) {
    full_address := server + ":5222"
    fmt.Printf("Connecting to %s\n", full_address)
    conn, err := net.Dial("tcp", full_address)
    if err != nil {
        fmt.Errorf("Couldn't connect")
        return
    }

    fmt.Printf("Connection made\n")
    self.connection = conn
    self.host = host
    self.username = username
    self.password = password

    self.State = "waiting_for_stream"
    self.request_id = 0
    self.id = "bot"
    go self.receive()
    self.startStream()
}
```

The *receive* method, called as a Go routine, is basically an
infinite loop, reading data from the socket, parsing it and
handling each stanza in turn:

```go
func (self * XmppHandler) receive() {
    for {
        fmt.Print("Waiting to receive\n")
        recvBuf := make([]byte, 4096)
        n, err := self.connection.Read(recvBuf)
        if err != nil {
            fmt.Print(err)
        }
        msgReceived := string(recvBuf[:n])
        fmt.Printf("Message Received: %s\n", msgReceived)
        parseErr := self.parser.Parse(msgReceived)
        if parseErr != nil {
            fmt.Print(parseErr)
        }
        fmt.Printf("%d elements in queue\n", len(self.parser.queue))

        for {
            if len(self.parser.queue) == 0 {
                break
            }
            elem := self.parser.queue[0]
            self.parser.queue = self.parser.queue[1:]
            self.handleResponse(elem)
        }
    }
}
```
*iq* stanzas get a response back from the server - these are handled
by passing a callback to the *XmppHandler IssueRequest* method:
```go
func (self *XmppHandler) IssueRequest(request string, request_type string, to string, callback func(*XmppElement)) {
    request_id := self.getRequestId()
    to_clause := ""
    if len(to) > 0 {
        to_clause = fmt.Sprintf("to='%s'", to)
    }
    template := "<iq id='%s' type='%s' from='%s' %s>%s</iq>"
    packet := fmt.Sprintf(template, request_id, request_type, self.username, to_clause, request)
    fmt.Println(packet)
    self.requests[request_id] = callback
    self.connection.Write([]byte(packet))
}
```
The callback is then issued when handling a response:
```go
func (self *XmppHandler) handleResponse(response *XmppElement) (bool) {
    fmt.Printf("handleResponse - State %s\n", self.State)
    fmt.Println(response.ToXml())
    if response.Tag == "iq" {
        request_id := response.Attributes["id"]
        callback := self.requests[request_id]
        callback(response)
        return true
    }

    if self.State == "ready" {
        if response.Tag == "message" {
            if self.HandleMessage != nil {
                self.HandleMessage(response)
            }
            return true
        }
    }
    ...
}
```
## EchoBot
With the *XmppHandler* in place, writing the actual bots is easy.
Here is *EchoBot*:
```go
func main() {
    handler := xmpp.NewXmppHandler()

    HandleMessage := func (element *xmpp.XmppElement) {
        sender := element.Attributes["from"]
        msg := element.Children[0].Text
        fmt.Printf("Got message from %s\n%s\n", sender, msg)
        handler.Message(sender, msg)
    }
    handler.HandleMessage = HandleMessage
    handler.Connect(
        "localhost",
        "localhost",
        "echobot",
        "echobot")

    exitSignal := make(chan os.Signal)
    signal.Notify(exitSignal, syscall.SIGINT, syscall.SIGTERM)
    <-exitSignal
}
```
Note that it is important that the main thread doesn't exit
prematurely - all the work is done in the *receive* Go routine.
The code above sets up a channel and waits on it, until you hit
Ctrl-C. This allows the receive to simply block on the Read call,
then process data as it comes in. The echo bot is purely reactive
so it only needs that one Go routine to do its thing.

Try this out with some XMPP chat client - I've been using
[Swift](https://swift.im/swift.html). Start a chat with *echobot@localhost*
and you should see everything you say echoed back.

## JumperBot
The *JumperBot* jumps between chat rooms, saying a few random
phrases in each room, then jumping to the next one. This can be
useful for generating traffic for load testing your chat server,
by running any number of those bots. See 
[here](https://ccpsnorlax.blogspot.is/2017/10/jumperbot.html)
for a discussion of my Python implementation of this.

```go
type JumperBot struct {
    xmppHandler *xmpp.XmppHandler
    name string
    host string
    numRooms int
    currentChannel string
}

func NewJumperBot(numRooms int) *JumperBot{
    return &JumperBot {
        xmppHandler:xmpp.NewXmppHandler(),
        numRooms: numRooms,
    }
}

func (self *JumperBot)Connect(server string, host string, username string, password string) {
    self.name = username
    self.host = host
    self.xmppHandler.Connect(server, host, username, password)
}

func (self *JumperBot)Run() {
    for self.xmppHandler.State != "ready" {
        fmt.Printf("Waiting for %s to log in\n", self.name)
        time.Sleep(time.Second)
    }

    for {
        self.joinRandomRoom()
        n := rand.Intn(5) + 5
        for i := 0; i < n; i++ {
            self.sayRandomPhrase()
            seconds := rand.Intn(10) + 5
            time.Sleep(time.Duration(seconds)*time.Second)
        }
    }
}

func (self *JumperBot) sayRandomPhrase() {
    choice := rand.Intn(len(PHRASES))
    phrase := PHRASES[choice]
    self.xmppHandler.GroupChat(self.currentChannel, phrase)
}

func (self *JumperBot) joinRandomRoom() {
    roomNumber := rand.Intn(self.numRooms)
    room := fmt.Sprintf("bot_room_%d@conference.%s", roomNumber, self.host)
    if room != self.currentChannel {
        if self.currentChannel != "" {
            fmt.Printf("%s: Leaving %s", self.name, self.currentChannel)
            self.xmppHandler.LeaveRoom(self.currentChannel)
        }
        self.currentChannel = room
        self.xmppHandler.JoinRoom(self.currentChannel)
    }
}
```
The *main* function instantiates a few bots:
```go
func main() {
    bots := make([]*JumperBot, 10)
    for i:= 0; i < 10; i++ {
        fmt.Printf("Creating bot %d", i)
        bot := NewJumperBot(10)
        go bot.Connect(
            "localhost",
            "localhost",
            fmt.Sprintf("jumperbot_%d", i, ),
            "jumperbot")
        go bot.Run()
        bots = append(bots, bot)
    }
    exitSignal := make(chan os.Signal)
    signal.Notify(exitSignal, syscall.SIGINT, syscall.SIGTERM)
    <-exitSignal
}
```
This time we're using more Go routines - the *Run* method is called
as a Go routine as the *JumperBot* is proactive, rather than simply
reactive as the *EchoBot* was.

## Trying it out
The bots do their chatter in rooms named *bot_room_0* through *bot_room_9*.
Connect to the server with Swift (or your favorite chat client) and join
one or more of those rooms to listen in.

## What's next?
I need to polish the *XmppHandler* - the *handleResponse* method is
kind of quick and dirty. The state machine for the handshake is simply
a series of if statements and doesn't deal with errors at all. I would
also like to see if there is a better way to handle incomplete XML
handling in *XmppContentHandler*. Hopefully some experienced Go
programmers out there will give me some constructive feedback on all
this.

Then I want to take a step back and compare the Go and Python
implementations. My first impression of the concurrent features of
Go are very positive. This feels easier and more natural in Go
than using Python 3 and asyncio.

Finally, I want to do some performance comparisons between Python
and Go. How many bots can I run on one machine before things start
breaking down?