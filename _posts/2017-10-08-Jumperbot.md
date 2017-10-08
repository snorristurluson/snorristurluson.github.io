In a [previous blog](../Echobot.html)
I described a simple echo bot, that echoes back anything you say to it. This time
I will talk about a bot that generates traffic for the chat server, that can
be used for load-testing both the chat server as well as any chat clients
connected to it.

I've dubbed it *JumperBot* - it jumps between chat rooms, saying a few random
phrases in each room, then jumping to the next one. This bot builds on the
same framework as the *EchoBot* - refer to the previous blog if you are interested
in the details. The source lives on GitHub: [https://github.com/snorristurluson/xmpp-chatbot]()

## Configure the server
In an [earlier blog](https://ccpsnorlax.blogspot.is/2017/09/working-with-xmpp-in-python.html) 
I described the setup of Prosody as the chat server to run against. Before we can connect bots 
to the server we have to make sure they can log in, either by creating accounts for them:

```
prosodyctl register jumperbot_0 localhost jumperbot
prosodyctl register jumperbot_1 localhost jumperbot
...
```

or by [setting the authentication up](http://modules.prosody.im/mod_auth_any.html) 
so that anyone can log in.

We also need to enable multi-room chat - do this by adding this to the Prosody
configuration file (near the bottom of the file, in the Components section):
```
Component "conference.localhost" "muc"
```

## JumperBot
The *Jumperbot* is similar to the *EchoBot*, but rather than handling
incoming messages we set up an asyncio task as this bot is proactive
where the echo bot was reactive. The task is created when the connection
is made:
```python
    self.task = asyncio.get_event_loop().create_task(self.run())
```

The run method looks like this:
```python
    async def run(self):
        while True:
            self.join_random_room()
            n = random.randint(5, 10)
            for i in range(n):
                if self.transport.is_closing():
                    return
                self.say_random_phrase()
                try:
                    await asyncio.sleep(random.random() * 10.0 + 5.0)
                except asyncio.CancelledError:
                    return

```
This implements the bot behavior as described above - joins a random room,
says a few random phrases, then repeats the process in the next room. The
*asyncio.sleep* command is very important - this allows other tasks to
run concurrently with this loop.

## BotManager
Running a single *JumperBot* doesn't really generate much traffic and rather
than running multiple processes, let's make use of *asyncio* and create
multiple bots as tasks. For that, we set up a *BotManager* to create and
monitor bots:

```python
    manager = BotManager()

    manager.create_bots(args)
    loop = asyncio.get_event_loop()

    loop.create_task(manager.monitor_status(args.monitor))
```

The bot manager looks like this:
```python
class BotManager(object):
    def __init__(self):
        self.bots_running = {}
        self.bots_logged_in = {}
        self.args = None

    def create_bot(self, botname, args):
        bot = JumperBot(self, args.host_name, botname, "jumperbot",
                        args.num_rooms)
        if args.listener:
            bot.listener = True

        self.connect_bot(bot, args)

        return bot

    def connect_bot(self, bot, args):
        loop = asyncio.get_event_loop()
        handler = loop.create_connection(lambda: bot, args.server_name, 5222)
        loop.create_task(handler)

    def create_bots(self, args):
        self.args = args
        for i in range(args.num_bots):
            botname = "jumperbot_{0}".format(i)
            bot = self.create_bot(botname, args)
            self.bots_running[bot.username] = bot
```
Each bot is a protocol instance associated with its connection and
gets a callback, *data_received* whenever something is received from
the server. The bot also runs its own task for initiating its
chattiness, as described above.

There is one more task - the monitor:
```python
    async def monitor_status(self, display_stats):
        blinkers = [" ", ".", ":", "."]
        blinker_index = 0
        template = "{2} bots running, {3} logged in {4}"
        while True:
            await asyncio.sleep(1)
            if display_stats:
                print(template.format(
                    len(self.bots_running),
                    len(self.bots_logged_in),
                    blinkers[blinker_index]),
                    end="\r"
                )
            blinker_index += 1
            blinker_index %= len(blinkers)

            if len(self.bots_running) == 0:
                asyncio.get_event_loop().stop()
                return
```
If no bots are running, the loop is stopped.

## Trying it out
The bots do their chatter in rooms named *bot_room_0* through *bot_room_9*.
Connect to the server with Swift (or your favorite chat client) and join
one or more of those rooms to listen in. You can also run the jumperbot
with a ```--verbose``` flag to see all the XMPP traffic in the log.