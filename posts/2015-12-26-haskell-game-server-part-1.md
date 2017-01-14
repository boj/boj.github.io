---
title: Haskell Game Server - Part 1
---

This post describes the project [Armored Bits](http://armoredbits.com/) which [myself](https://twitter.com/mojobojo) and my buddy [cerbermus](https://twitter.com/cerbermus) have been working on for over half a year in our spare time.  The goal is to build a realtime, network team based, player programmable 3d mech simulator.  As the title suggests the core game server is being written in Haskell, a choice I made after having been inspired by a [QuakeCon talk](https://www.youtube.com/watch?v=1PhArSujR_A) with John Carmack, the linked video a part of that talk specifically about functional programming and games.  There is also a client viewer written in Unity3d and our first SDK written in JavaScript both being tackled by cerbermus.

Not including a failed attempt several years ago I have only been using Haskell for the last 9 months and am by all accounts an amatuer with the language.  It has proven quite enjoyable to use, however, especially as of late now that I've gotten rid of some of my imperative programming notions and understand more functional programming concepts better.

My goal with this post is to show a concrete non-web example of a project that is being built with Haskell.  People in the [community](https://www.reddit.com/r/haskell/) have pointed out that there are a plethora of beginner level tutorials and high level you-need-a-PhD-to-read-it articles, but not much in the way of meat and potatoes concrete project guides.  This is my humble attempt to share back to the community I've learned so much from over the last several months.

## Defining the game server

First and foremost we need to make a list of our requirements.

* TCP networking which exposes a port for external 3d clients and a port for internal (local) SDK connections.
* A way for the server and clients to communicate via some kind of messaging protocol.
* A way to track client connections and link them to their corresponding game entities.
* Scene management and the game loop.
* A linear algebra library to manipulate scene objects.
* A configuration phase and configuration sanity checker.
* A query interface for SDK clients to configure their mech, get information about it, and put values to manipulate its behavior.

In today's post I will address the TCP networking and messaging protocol.

## Networking

We decided to strictly use TCP because this game doesn't require human input, nor all the magical UDP networking and iterpolation/extrapolation tricks of a first person shooter.  The 3d client simply processes messages and displays the scene based on that data, while the SDK clients run on the same network as the game server.

The primary `Main.hs` code is as follows:

```haskell
module Main where

import           Control.Concurrent    (forkIO)
import           Data.Yaml             (ParseException, decodeFileEither)
import           Network.Socket

import           Data.Configurations
import           Game.Server

internalPort :: PortNumber
internalPort = 4000

externalPort :: PortNumber
externalPort = 5000

maxInternalConns :: Int
maxInternalConns = 10

maxExternalConns :: Int
maxExternalConns = 10

maxPlayers :: Int
maxPlayers = 10

acceptClients :: (Server -> Socket -> IO ()) -> Server -> Socket -> IO ()
acceptClients handler server sock = do
  (next, _) <- accept sock
  setSocketOption next NoDelay 1 -- disable nagle
  _ <- handler server next
  acceptClients handler server sock

startListener :: PortNumber -> Int -> (Server -> Socket -> IO ()) -> Server -> IO ()
startListener port maxClients handler server = do
  sock <- socket AF_INET Stream defaultProtocol
  setSocketOption sock ReuseAddr 1
  bindSocket sock $ SockAddrInet port iNADDR_ANY
  listen sock maxClients
  acceptClients handler server sock

main :: IO ()
main = withSocketsDo $ do
  -- load configurations
  configData <- decodeFileEither "./data/configurations.yml" :: IO (Either ParseException Configurations)
  case configData of
    Left err ->
      print err
    Right configs -> do
      server <- newServer configs maxPlayers
      -- external client connections
      _ <- forkIO $ startListener externalPort maxExternalConns addExternalClient server
      -- internal client connections
      _ <- forkIO $ startListener internalPort maxInternalConns addClient server
      -- start server
      runWorld server
```

We'll break this down into simple to understand pieces.

The first section simply loads our `configurations.yml` which contains a large amount of mech configuration data (I'm in the process of moving server settings out to their own configuration file as well).  It simply loads the file and prints out why it failed with the `Left` evaluation of the `Either` type which is returned.

We apparently need `withSocketsDo` if this code is to ever run on a Windows machine.

```haskell
main :: IO ()
main = withSocketsDo $ do
  -- load configurations
  configData <- decodeFileEither "./data/configurations.yml" :: IO (Either ParseException Configurations)
  case configData of
    Left err ->
      print err
```

Upon successful YAML parsing the `Either`'s `Right` is evaluated and creates a new `Server` (to be defined later). It then runs the network listeners for the internal and external clients in their owns threads, and finally runs the game loop in the main thread.

```haskell
    Right configs -> do
      server <- newServer configs maxPlayers
      -- external client connections
      _ <- forkIO $ startListener externalPort maxExternalConns addExternalClient server
      -- internal client connections
      _ <- forkIO $ startListener internalPort maxInternalConns addClient server
      -- start server
      runWorld server
```

The `startListener` function is designed to take a handler function with the signature `Server -> Socket -> IO()` and listen on the designated port for new client connections.  These new connections then get passed off to the `acceptClients` function.

```haskell
startListener :: PortNumber -> Int -> (Server -> Socket -> IO ()) -> Server -> IO ()
startListener port maxClients handler server = do
  sock <- socket AF_INET Stream defaultProtocol
  setSocketOption sock ReuseAddr 1
  bindSocket sock $ SockAddrInet port iNADDR_ANY
  listen sock maxClients
  acceptClients handler server sock
```

New socket connections use the socket option `NoDelay 1` in order to disable the Nagle Algorithm and decrease network latency.  The handler then takes the newly connected socket and hands it off to the `Server` via the handler so it can map a connection to a `Client` entity, then the listener calls itself again and waits for the next incoming connection.

```haskell
acceptClients :: (Server -> Socket -> IO ()) -> Server -> Socket -> IO ()
acceptClients handler server sock = do
  (next, _) <- accept sock
  setSocketOption next NoDelay 1 -- disable nagle
  _ <- handler server next
  acceptClients handler server sock
```

## Communication

The next step is to determine a communication strategy.  We've used [Google's Protocol Buffers](https://developers.google.com/protocol-buffers/?hl=en) before on other projects and they seem like a reasonable way to efficiently pass streaming data back and forth over the wire.  The specific Haskell library I am using is [protobuf](https://hackage.haskell.org/package/protobuf).  Unfortunately it is not built to auto generate code from `.proto` files (yet, apparently) which does lead to headaches when you miss a field or put the wrong type in the `.proto` file your buddy uses, resulting in him scratching his head wondering why things don't work.  The library itself has been quite the pleasure to use though.

We handle message encoding/decoding with a couple simple functions. 

```haskell
{-# LANGUAGE DataKinds     #-}
{-# LANGUAGE DeriveGeneric #-}

module Game.Messages where

import qualified Data.Binary           as DB
import qualified Data.ByteString.Char8 as BS
import qualified Data.ByteString.Lazy  as BL
import           Data.Int
import           Data.ProtocolBuffers
import           Data.Serialize
import           Data.Word

class GameMessage m where
  getMessageId :: m -> Word8

messageIn :: (GameMessage m, Decode m) => BS.ByteString -> Either String m
messageIn =
  runGet decodeMessage

messageOut :: (GameMessage m, Encode m) => m -> BS.ByteString
messageOut =
  runPut . encodeMessage

messageOutWithIdAndLength :: (GameMessage m, Encode m) => m -> BS.ByteString
messageOutWithIdAndLength msg =
  BS.concat [mid1, len, out]
  where
    mid1 = BL.toStrict $ DB.encode (getMessageId msg)
    out  = messageOut msg
    len  = BL.toStrict $ DB.encode (fromIntegral (BS.length out) :: Int16)
```

We've chosen to make our messages 3 bytes at the very least.  The first byte denotes the message id, the second two bytes the size of the body of the message, and the remaining bytes the actual body if applicable.  We are unfortunately wasting 2 bytes in the case of a message that has no body, however, it means we don't have to pre-check what kind of message is being passed and avoid writing specific code to handle those exceptions.

The `messageOutWithIdAndLength` uses our handy `GameMessage` class to pull the predefined message id and `concat` it together with the message's length and body.

An example message looks something like this:

```haskell
{-# LANGUAGE DataKinds     #-}
{-# LANGUAGE DeriveGeneric #-}

module Game.Messages.ClientActionLoginRequest where

import           Data.ProtocolBuffers
import           GHC.Generics          (Generic)

import           Game.Messages

data ClientActionLoginRequest
  = ClientActionLoginRequest {
      username :: Required 1 (Value String)
    , password :: Required 2 (Value String)
    }
  deriving (Generic, Show)

instance Encode ClientActionLoginRequest
instance Decode ClientActionLoginRequest
instance GameMessage ClientActionLoginRequest where
  getMessageId _ = 200
```

And the relevant sections of code from `Server.hs` to send and receive messages.  I am using [io-streams](https://hackage.haskell.org/package/io-streams) which is a fantastically simple to use I/O library for processing streams.

The `runClient` function runs in its own thread and handles the incoming data from the associated client connection. `UserId` is a unique identifier which associates the socket with a `Client` and their `Player` data.

```haskell
-- receive messages
runClient :: Server -> Socket -> UserId -> IO ()
runClient server sock uid = do
  (inS, _) <- Streams.socketToStreams sock        -- utilize the incoming half of the socket
  mid <- Streams.readExactly 1 inS                -- parse out the message id
  let sid = fromIntegral $ runGet getWord8 (BL.fromStrict mid) -- turn the byte into a Word8
  lth <- Streams.readExactly 2 inS                -- parse out the message length
  when (msgSize lth <= maxMsgSize) $ do           -- check against a max message size to stop abuse
    msg <- Streams.readExactly (msgSize lth) inS  -- parse out the message body if applicable
    handleMessage server uid sid msg              -- handle the message
  runClient server sock uid                       -- loop and wait for the next message
  where
    msgSize lth = (fromIntegral $ runGet getWord16be (BL.fromStrict lth)) :: Int

-- send message
sendMessage :: (GameMessage m, Encode m) => m -> Socket -> IO ()
sendMessage msg sock = do
  (_, outS) <- Streams.socketToStreams sock
  handle (\(SomeException e) -> traceIO $ show e ) $
    Streams.write (Just $ messageOutWithIdAndLength msg) outS
```

## Conclusion

Building a networked game server takes an incredible amount of work and we've only begun to scratch the surface of all the components which bring it together.  Today I touched on some basic networking and communication techniques we are using such as low level `Network.Socket`'s wrapped in `System.IO.Streams`, and message passing techniques with `Data.ProtocolBuffers`.  In my next post I will discuss how clients interact with the game, how the game world is managed, and how our `Actor` class helps us manage scene objects.
