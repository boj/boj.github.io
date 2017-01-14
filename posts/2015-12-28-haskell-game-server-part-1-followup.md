---
title: Haskell Game Server - Part 1 Followup
---

Before I get started, I've changed the color scheme of this blog after a few comments about the previous colors I was using.  Sorry for offending everyone's eyes!  Hopefully the current palette is easier to read.

Anyways, this is a quick followup to my previous post [Haskell Game Server - Part 1](/posts/2015-12-26-haskell-game-server-part-1.html) which covers a few changes I made after some great feedback, and grievous debugging of a crazy issue.

## Changes to Server.hs

The following things were pointed out to me:

* Use `forever` instead of recursive calls to the same function as it makes your intent clearer.
* `readMessage` should exist and be paired with `sendMessage` rather than wrapped up in the unrelated `runClient`.

In a completely unrelated debugging nightmare that is oddly related to this specific code, I've also subtly changed how these methods work.  With a high rate of messages flying back and forth we'd eventually lose a message here and there which was causing our event driven SDK to stall, even though it was clearly connected and doing other things.  After scratching my head for a week I finally narrowed it down to two things today:

* Use `lockingInputStream` and `lockingOutputStream` if your sockets are accessed from other threads.  This slows throughput down, but gives you thread-safety.
* Rather than repeatedly call `socketToStreams` in each read/send function utilize memoization and store the socket pair in the `Client` record.  This seemed to fix any lingering problems we had.

The following is the revised code:

```haskell
runClient :: Server -> Client -> IO ()
runClient server client = forever $ do
  (mid, msg) <- readMessage (client^.clientInput)
  handleMessage server (client^.clientId) mid msg
  return ()

readMessage :: Streams.InputStream BS.ByteString -> IO (Word8, BS.ByteString)
readMessage in' = do
  inS <- Streams.lockingInputStream in'
  mid <- handleStream 1 inS
  lth <- handleStream 2 inS
  out <- if msgSize lth <= maxMsgSize
         then handleStream (msgSize lth) inS
         else return BS.empty
  return (midData mid, out)
  where
    midData mid' = fromIntegral $ runGet getWord8 (BL.fromStrict mid')
    msgSize lth' = (fromIntegral $ runGet getWord16be (BL.fromStrict lth')) :: Int
    handleStream len str =
      handle (\(SomeException e) -> do debug $ "read failed: " ++ show e; return BS.empty) $
        Streams.readExactly len str

sendMessage :: (GameMessage m, Encode m) => m -> Streams.OutputStream BS.ByteString -> IO ()
sendMessage msg out' = do
  outS <- Streams.lockingOutputStream out'
  handle (\(SomeException e) -> debug $ "write failed: " ++ show e ) $
    Streams.write (Just $ messageOutWithIdAndLength msg) outS
```
