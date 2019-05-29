+++
title = "Twitch chat Client (part 1)"
date = "2019-04-14"
author = "Norberts Kakste"
cover = ""
tags = ["scala", "akka", "parsing", "irc", "TCP"]
description = "IRCv3 Protocol parsing"
showFullContent = false
+++
#### Introduction
In this 3 part tutorial series we're going to explore `TCP` Connection handling; `IRCv3` message parsing and connection handling for `Twitch.tv` Chat. Here in first part of the tutorial we're going to implement `IRC` protocol `Client` for our `Twitch.tv` Chat Client.

#### Research

First of all, we should start with research:

* How to connect to `IRC` server
* How to send `IRC` messages.
* Is `Twitch.tv` chat fully `IRC` capable?
* Are there any special `Twitch.tv` features?

Looking at [Chatbots & IRC Guide] (https://dev.twitch.tv/docs/irc/guide/) we can see that their IRC Server generally follows [RFC1459](https://tools.ietf.org/html/rfc1459.html), with some minor exceptions that are specific to **custom** chat commands, and message **tags**

Connecting to Twitch IRC is fairly straightforward as described in [docs](https://dev.twitch.tv/docs/irc/guide/#connecting-to-twitch-irc)

| WebSocket Clients               | IRC Clients                   |
|---------------------------------|-------------------------------|
| wss://irc-ws.chat.twitch.tv:443 [SSL] | irc://irc.chat.twitch.tv:6697 [SSL] |
| ws://irc-ws.chat.twitch.tv:80   | irc://irc.chat.twitch.tv:6667 |

We're gonna use `ncat` to explore how `IRC` protocol works

Establishing connection to `irc.chat.twitch.tv:6697` with `ncat`:

`ncat -C --ssl irc.chat.twitch.tv 6697`

Nothing interesting happens when we establish connection, so we need to send something that would **authenticate** us.
In `Twitch.tv` IRC server case that would be supplying our `PASS` (oauth token) and `NICK` (nickname), but since we don't want to implement Oauth2 flow for now, we can authenticate anonymously with `justinfan<numbers>` nickname

To send our `NICK` message we simply write it inside our terminal with connection open to `irc.chat.twitch.tv:6697`

<script id="asciicast-248859" src="https://asciinema.org/a/248859.js" async></script>

Ok, the server sent us a greeting message! So how can we join **channels**? With `JOIN` command that is! Simply join channel with `JOIN #channel_name` command

<script id="asciicast-248861" src="https://asciinema.org/a/248861.js" async></script>

We can see the incoming flow of messages now, while that is good we can do alot better with `CAP` (Client Capability Negotiation). In [Chatbots & IRC Guide] (https://dev.twitch.tv/docs/irc/guide/) you can see that `Twitch.tv` chat supports IRC v3 capability registration that allows us to *request* extra parameters that we would like to *receive* from `Twitch.tv` Chat.

Here's a list of all `irc.chat.twitch.tv:6697` CAP (capabilities) and their request **commands**

| Capability | Command |
|------------|-------------------------------|
| Membership | CAP REQ :twitch.tv/membership |
| Tags       | CAP REQ :twitch.tv/tags       |
| Commands   | CAP REQ :twitch.tv/commands   |

From [Chatbots & IRC Guide] (https://dev.twitch.tv/docs/irc/guide/):

> Membership — Adds membership state event data. By default, we do not send this data to clients without this capability.

> Tags — Adds IRC V3 message tags to several commands, if enabled with the commands capability.

> Commands — Enables several Twitch-specific commands.

Now that we know how to get **extra** chat features we should try and use them; as always establish new `ncat` session and send CAP requests to `irc.chat.twitch.tv:6697` server

<script id="asciicast-248862" src="https://asciinema.org/a/248862.js" async></script>

Looks like we're done with general `IRC` exploration with `ncat` so we're gonna move on and make our own `TwitchChatClient` in `Scala`

#### TwitchIrcClient

First of all we need to set-up a basic SBT project, simple `build.sbt` with `akka-actor` as a dependency will suffice for now
```java
name := "IrcClient"

version := "0.1"

scalaVersion := "2.12.8"

libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-actor" % "2.5.23"
)
```


##### Simple Irc Client
First of all we need to import these libraries to help us with TCP Connection handling

```java
import java.net.InetSocketAddress

import akka.actor.{Actor, ActorSystem, Props}
import akka.io.Tcp._
import akka.io.{IO, Tcp}
import akka.util.ByteString
```

> *If you're not familiar with `Akka` actors, you should read up on them [HERE](https://doc.akka.io/docs/akka/current/actors.html), since rest of the tutorial will involve working with them.*

Now we need to implement Actor that will connect to `Twitch.tv` `IRC` chat servers. To establish connection to `twitch.tv` chat we need to connect to `irc://irc.chat.twitch.tv:6667` address, splitting the address into `InetSocketAddress` manageable fields would look like this 
```java
val host = "irc.chat.twitch.tv"
val port = 6667

val remoteAddress = new InetSocketAddress(host, port)
```

Ok now we know *address* and *port* for connection, and we already created `InetSocketAddress` **object**, so we're ready to create our simple `TwitchChatClient` actor that will connect to our `remoteAddress`

```java
class TwitchChatClient(remoteAddress: InetSocketAddress) extends Actor {

  import context.system

  println(s"Connecting to ${remoteAddress.getHostName}:${remoteAddress.getPort}")

  // Connecting to TCP with our supplied remote Address
  IO(Tcp) ! Connect(remoteAddress = remoteAddress) 

  def receive = {
    // Connection failed
    case CommandFailed(_: Connect) =>
      println("Connection failed.")
      context stop self

    // Connection succeeded 
    case Connected(remote: InetSocketAddress, local: InetSocketAddress) => 
      println("Connection succeeded!")
      val connection = sender()
      connection ! Register(self)
      // Change actor behaviour to handle Received & ConnectionClosed messages
      context.become {
        case Received(data: ByteString) =>
          println(s"Message: ${data.utf8String}")
        case _: ConnectionClosed =>
          // Stop actor ir ConnectionClosed
          context.stop(self)
      }
  }
}
```

Create `TwitchChatClient` **actor**
```java
object Main extends App {
  implicit val actorSystem: ActorSystem = ActorSystem("irc-client")

  val host = "irc.chat.twitch.tv"
  val port = 6667

  val remoteAddress = new InetSocketAddress(host, port)
  val props = Props(classOf[TwitchChatClient], remoteAddress)

  val tcpActor = actorSystem.actorOf(props)
}
```

Running Main `App` you should see that TCP connection succeded
```
Connecting to irc.chat.twitch.tv:6667...
Connection succeeded!
```

###### Chat command handling

To connect further we need to write `NICK <nickname>` message to our TCP connection to register our IRC `nickname`. As we tried earlier we know that `Twitch.tv` Chat allows us to connect anonymously if we connect with `justinfan<numbers>` nickname, so we're gonna do that.

At the moment we will **hardcode** our `NICK` sending to TCP client, but further into tutorial we'll write proper *nickname* handling .

To `Write` messages to our TCP connect we need to use `Write(ByteString("message"))` function that's accesible inside our `receive` handler.

```java
case Connected(remote: InetSocketAddress, local: InetSocketAddress) =>
    println("Connection succeeded!")
    val connection = sender()
    connection ! Register(self)
    // Writing NICK justinfan5123 message to TCP connection
    connection ! Write(ByteString("NICK justinfan5123\r\n"))
    context.become {
    case Received(data: ByteString) =>
        println(s"Message: ${data.utf8String}")
    case _: ConnectionClosed =>
        context.stop(self)
    }
}
```

Since IRC is *delimited* by `\r\n` characters we need to add them to the end of our commands, in this example we added it to the end of `NICK justinfan5123` message.

Running our example we should see the initial Connection succeeded message + Welcome message from `tmi.twitch.tv`
```
Connecting to irc.chat.twitch.tv:6667...
Connection succeeded!
Message: :tmi.twitch.tv 001 justinfan5123 :Welcome, GLHF!
:tmi.twitch.tv 002 justinfan5123 :Your host is tmi.twitch.tv
:tmi.twitch.tv 003 justinfan5123 :This server is rather new
:tmi.twitch.tv 004 justinfan5123 :-
:tmi.twitch.tv 375 justinfan5123 :-
:tmi.twitch.tv 372 justinfan5123 :You are in a maze of twisty passages, all alike.
:tmi.twitch.tv 376 justinfan5123 :>
```

Ok now we can connect to `twitch.tv` chat and supply our anonymous nickname so next logical step for us would be to join `channel`/`room` to read incoming twitch chat messages.

While hardcoding `join` logic would be easier that writing proper `nickname` and `join` handlers the code would become unmaintainable and hard to read, so we're gonna implement custom IRC command handling logic.

We can start with **abstract class** that specifies `IrcCommand` logic.

```java
abstract class IrcCommand(private val command: String) {
  def getMessage: String

  def getCommand: ByteString = {
    ByteString(s"$command $getMessage\r\n")
  }
}
```

Here we define **abstract class** `IrcCommand` with `command` parameter to help us with `IRC` command management, while also providing `getMessage` method to help us with message handling (you will see why we need it later in tutorial). We also provide `getCommand` method that returns `ByteString` since our actor `Write` method expects us to supply `ByteString` type.

Implementing `Nick` and `Join` commands

```java
object IrcClient {
  case class Nick(message: String) extends IrcCommand("NICK") {
    override def getMessage: String = message
  }

  case class Join(channelName: String) extends IrcCommand("JOIN") {
    // Return channelName with # in front of it, since IRC channels start with #
    override def getMessage: String = s"#$channelName"
  }
}
```

Now we need to handle our custom `Nick` and `Join` commands inside our `IrcClient` actor, we could put them in our receive function **but** we need to think about our `Connected` logic carefully. If we're gonna send `Nick` message to our actor what guarantees that our connection will be open?

Here's an simple diagram that shows our message queue

![Example image](/no_stash.svg)

You can see that when we send our `Nick` and `Join` messages they're not guarranted to be after `Connected` message, and knowing that opening `TCP` connection is probably faster than sending message to actor, we can be sure that our **connection** won't be established when we send our `Nick` and `Join` messages. So how can we bypass this limitation? With `stash` trait, we can `stash` our unhandable **messages** untill our **connection** is open.

![Example image](/stash.svg)

Here i visualized stashing process, we sent `Nick` and `Join` commands, but they are `stashed()` before `Connected` is finished, when it's finished we can `unstash()` them and process them further.

Heres the code implementation

```java
class IrcClient(remoteAddress: InetSocketAddress) extends Actor with Stash {

  import context.system

  println(s"Connecting to ${remoteAddress.getHostName}:${remoteAddress.getPort}")

  IO(Tcp) ! Connect(remoteAddress = remoteAddress)

  def receive = {
    case CommandFailed(_: Connect) =>
      println("Connection failed.")
      context stop self

    case Connected(remote: InetSocketAddress, local: InetSocketAddress) =>
      println("Connection succeeded!")
      val connection = sender()
      connection ! Register(self)
      // Unstash all messages when Connected
      unstashAll()
      context.become {
        // IrcCommand handler
        case m: IrcCommand => connection ! Write(m.getCommand)
        case Received(data: ByteString) =>
          println(s"Message: ${data.utf8String}")
        case _: ConnectionClosed =>
          context.stop(self)
      }
    // Stash unknown messages until TCP Connection succeeds
    case msg: IrcCommand => stash()
  }
}
```

Using our custom `Nick` and `Join` messages should be trivial, just send them to our `tcpActor` like with every other message.

```java
object Main extends App {
  implicit val actorSystem: ActorSystem = ActorSystem("irc-client")

  val host = "irc.chat.twitch.tv"
  val port = 6667

  val remoteAddress = new InetSocketAddress(host, port)
  val props = Props(classOf[IrcClient], remoteAddress)

  val tcpActor = actorSystem.actorOf(props)

  tcpActor ! Nick("justinfan5123")
  tcpActor ! Join("asmongold")
}
```

Running the example shows that we connected with NICK `justinfan5123` and `JOINED` channel `#asmongold` (chose your own channel so you can check incoming flow of messages).

Inspecting console we can see the incoming flow of messages:

```
Message: :tinychimp!tinychimp@tinychimp.tmi.twitch.tv PRIVMSG #asmongold :WOAH

Message: :matthewz92!matthewz92@matthewz92.tmi.twitch.tv PRIVMSG #asmongold :DoritosChip TheIlluminati DoritosChip TheIlluminati DoritosChip TheIlluminati DoritosChip TheIlluminati

Message: :ginbay!ginbay@ginbay.tmi.twitch.tv PRIVMSG #asmongold :RETAIL - GOOD LULW

Message: :khitomer!khitomer@khitomer.tmi.twitch.tv PRIVMSG #asmongold :REEEEEEEEEEEEEEETAIL
```

Since Twitch Chat supports `Client Capability Negotiation` or in short `CAP` requests, we can implement Twitch Chat supported `CAP` requests, and those are:
* `CAP REQ :twitch.tv/membership` (Adds membership state event data)
* `CAP REQ :twitch.tv/tags` (Adds IRC V3 message tags)
* `CAP REQ :twitch.tv/commands` (Enables several Twitch-specific commands)

To implement them we need to create new **case classes** that extend our `IrcCommand` **abstract class** just as we did with `Nick` and `Join` messages.

```java
case class CapMembership() extends IrcCommand(command = "CAP REQ") {
    override def getMessage: String = ":twitch.tv/membership"
}

case class CapTags() extends IrcCommand(command = "CAP REQ") {
    override def getMessage: String = ":twitch.tv/tags"
}

case class CapCommands() extends IrcCommand(command = "CAP REQ") {
    override def getMessage: String = ":twitch.tv/commands"
}
```

To send them we use them as regular actor messages.

```java
tcpActor ! Nick("justinfan5123")
tcpActor ! CapMembership()
tcpActor ! CapTags()
tcpActor ! CapCommands()
tcpActor ! Join("asmongold")
```
Checking the output when we run our programm it shows us extended messages with `TAG` support and custom `twitch.tv` chat commands.

```
Message: @badge-info=;badges=;color=#008000;display-name=bugcatcher17;emotes=;flags=;id=d943e4cb-4a32-482a-8ef5-0d308bc9acf5;mod=0;room-id=26261471;subscriber=0;tmi-sent-ts=1559080976540;turbo=0;user-id=279173457;user-type= :bugcatcher17!bugcatcher17@bugcatcher17.tmi.twitch.tv PRIVMSG #asmongold :NEXOS

Message: @badge-info=;badges=;color=;display-name=yuri_bezmenov_cx;emotes=357:0-8;flags=;id=9d4dcbed-a7c0-4dc3-9c16-43c58d1fa53d;mod=0;room-id=26261471;subscriber=0;tmi-sent-ts=1559080976850;turbo=0;user-id=437145684;user-type= :yuri_bezmenov_cx!yuri_bezmenov_cx@yuri_bezmenov_cx.tmi.twitch.tv PRIVMSG #asmongold :HotPokket 7

Message: @badge-info=;badges=;color=;display-name=ezscaping;emotes=;flags=;id=dce1ede0-9d79-4e51-b8e1-bd4792032063;mod=0;room-id=26261471;subscriber=0;tmi-sent-ts=1559080977102;turbo=0;user-id=252590920;user-type= :ezscaping!ezscaping@ezscaping.tmi.twitch.tv PRIVMSG #asmongold :@Kalio666 :NEXOS
```

OK, we're done with part 1! Next time we'll implement incoming message parsing with `fastparser` and improve our `IrcMessage` handling logic with `Finite State Machine` implementation for our `IrcClient`
___


# Full code

```java
import java.net.InetSocketAddress

import IrcClient._
import akka.actor.{Actor, ActorSystem, Props, Stash}
import akka.io.Tcp._
import akka.io.{IO, Tcp}
import akka.util.ByteString

abstract class IrcCommand(private val command: String) {
  def getMessage: String

  def getCommand: ByteString = {
    ByteString(s"$command $getMessage\r\n")
  }
}

object IrcClient {
  case class Nick(message: String) extends IrcCommand("NICK") {
    override def getMessage: String = message
  }

  case class Join(channelName: String) extends IrcCommand("JOIN") {
    override def getMessage: String = s"#$channelName"
  }

  case class CapMembership() extends IrcCommand(command = "CAP REQ") {
    override def getMessage: String = ":twitch.tv/membership"
  }

  case class CapTags() extends IrcCommand(command = "CAP REQ") {
    override def getMessage: String = ":twitch.tv/tags"
  }

  case class CapCommands() extends IrcCommand(command = "CAP REQ") {
    override def getMessage: String = ":twitch.tv/commands"
  }
}

class IrcClient(remoteAddress: InetSocketAddress) extends Actor with Stash {

  import context.system

  println(s"Connecting to ${remoteAddress.getHostName}:${remoteAddress.getPort}")

  IO(Tcp) ! Connect(remoteAddress = remoteAddress)

  def receive = {
    case CommandFailed(_: Connect) =>
      println("Connection failed.")
      context stop self

    case Connected(remote: InetSocketAddress, local: InetSocketAddress) =>
      println("Connection succeeded!")
      val connection = sender()
      connection ! Register(self)
      unstashAll()
      context.become {
        case m: IrcCommand =>
          println(m.getCommand.utf8String)
          connection ! Write(m.getCommand)
        case Received(data: ByteString) =>
          println(s"Message: ${data.utf8String}")
        case _: ConnectionClosed =>
          context.stop(self)
      }
    case msg: IrcCommand => stash()
  }
}

object Main extends App {
  implicit val actorSystem: ActorSystem = ActorSystem("irc-client")

  val host = "irc.chat.twitch.tv"
  val port = 6667

  val remoteAddress = new InetSocketAddress(host, port)

  val props = Props(classOf[IrcClient], remoteAddress)

  val tcpActor = actorSystem.actorOf(props)

  tcpActor ! Nick("justinfan5123")
  tcpActor ! CapMembership()
  tcpActor ! CapTags()
  tcpActor ! CapCommands()
  tcpActor ! Join("asmongold")
}
```

#### Extra materials:
1. [AKKA TCP](https://doc.akka.io/docs/akka/current/io-tcp.html)
2. [IRCv3](https://ircv3.net/)
3. [Twitch Chat Docs](https://dev.twitch.tv/docs/irc/)