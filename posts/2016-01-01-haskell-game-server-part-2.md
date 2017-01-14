---
title: Haskell Game Server - Part 2
---

## Previous Posts

* [Part 1](/posts/2015-12-26-haskell-game-server-part-1.html) - Networking & Message Protocols
* [Part 1 Followup](/posts/2015-12-28-haskell-game-server-part-1-followup.html) - A brief followup to my original post.

## Topics

In today's post I will cover which messages we shuffle between the server and clients and their purpose, how the game world is fundamentally managed, and how we use an `Actor` class to help manage scene objects.

## Messages Between Server & Clients

We've built our game so that the server only sends data the player AI has strictly requested, with a few minor exceptions such as game state changes.  The goal is to allow the player to drive as much of their robot's game play as possible through their AI decisions, of which is when and what to actually query.

There are three primary categories of messages:

* Configure
* Query
* Commit

We use techniques such as message size checking and API rate limiting to prevent people from abusing the server.

### Configure Messages

When the server is in the initial `ConfigureState` the player AI can configure their robot from a large selection of chassis, weapons, sensors, counter measures, reactors, engines, and more.  We liked the idea of giving players the ability to do this configuration in real time, especially if we add another reconnaissance phase giving them varying details about their enemy configurations, and the ability to save and load data about past matches should they desire to do so.

A configuration message may look something like this, where you simply define the documented model of a component to use:

```haskell
data SlugConfigureWeaponMessage
  = SlugConfigureWeaponMessage {
      weaponModel    :: Required 1 (Value String)
    , capacitorModel :: Optional 2 (Value String)
    , ammoModel      :: Required 3 (Value String)
    }
  deriving (Generic, Show)
```

The server then processes these configuration messages and makes sure that the correct amount of overall weight is available to add these components, and that you can actually plug a component into its receiving component.  Internally we call this a component *protocol* which defines the receiving _sockets_ and incoming _plugs_ it can handle.  For example, a specific arm model may only receive a total of two weapons, one of which is ballistic, the other a beam weapon, with no limitations.  Another arm model may only accept one missile weapon from a specific list of manufacturers.  This gives us a lot of power to mix, match, and limit which components can plug into what.

The very last action taken is a `SlugConfigureDoneRequest` message which simply signals that the AI is finished with their configuration.  If for any reason an AI does not configure itself before the end of the phase they are automatically assigned a default robot.  One can also optionally pass the fact that they want to use the default robot in the done request as well.

### Query Messages

Once the server is in the `GameState` the AI can then begin querying information about their robot.  Right now the primary query about one's robot is a large monolithic protobuf message.

```haskell
data SlugGetQueryWarMachineResponse
  = SlugGetQueryWarMachineResponse {
      state          :: Required 1 (Value String)
    , maxWeight      :: Required 2 (Value Int32)
    , currentWeight  :: Required 3 (Value Int32)
    , hasStorage     :: Required 4 (Value Bool)
    , storage        :: Optional 5 (Message SlugQueryStorageMessage)
    , hasCapacitor   :: Required 6 (Value Bool)
    , capacitor      :: Optional 7 (Message SlugQueryCapacitorMessage)
    , armCount       :: Required 8 (Value Int32)
    , legCount       :: Required 9 (Value Int32)
    , position       :: Required 10 (Message VectorMessage)
    , rotation       :: Required 11 (Message VectorMessage)
    , rotationRate   :: Required 12 (Value Float)
    , rotationTarget :: Required 13 (Value Float)
    , reactor        :: Required 14 (Message SlugQueryReactorMessage)
    , torso          :: Required 15 (Message SlugQueryTorsoMessage)
    , cockpit        :: Required 16 (Message SlugQueryCockpitMessage)
    , arms           :: Repeated 17 (Message SlugQueryArmMessage)
    , legs           :: Repeated 18 (Message SlugQueryLegMessage)
    }
  deriving (Generic, Show)
```

Each `Message` gives further details about the additional components inside the robot, such as data acquired by sensors, the firing and ammunition state of a weapoon, etc.  The player AI can then react on these values by doing things such as moving, rotating, firing weapons, communicating with teammates, and much more.

Another slightly smaller message defines data about enemy robots which were scanned by the sensors, with varying levels of detail based on how powerful the robot's equipped computer is.

As you can see we call our robots/mechs War Machines.

### Commit Messages

Interacting with the actual war machine is done with very simple messages such as the following:

```haskell
data SlugSetCommitEngineRequest
  = SlugSetCommitEngineRequest {
      state        :: Optional 1 (Value Int32)
    , acceleration :: Optional 2 (Value Float)
    }
  deriving (Generic, Show)

data SlugSetCommitArmWeaponRequest
  = SlugSetCommitArmWeaponRequest {
      weaponPosition :: Required 1 (Value Int32)
    , armPosition    :: Required 2 (Value Int32)
    , state          :: Optional 3 (Value Int32)
    , fireState      :: Optional 4 (Value Int32)
    }
  deriving (Generic, Show)
```

These allow the AI to change the state of a component (maybe they wish to power it down to utilize the additional reactor power for another component), and perform specific actions with the component.  In the above example setting the acceleration of the engine will begin moving the war machine.  Setting the fire state to `Reloading` will force the weapon to do a reload.

## Game World Management

This is one of the more interesting parts of the game server where we break away from the impure IO world and hand everything off into the pure simulation world.  The `World` state is stored inside a `TVar` so we can make use of Haskell's brilliant STM features to handle thread safety for us.

We store all relevant data using `IntMap`s, although I'm not sure if there is a better way to store game objects of various types.  Currently it seems the best method is to just make a map for each data type, of which we only have a few.

```haskell
type PlayerMap  = IntMap.IntMap Player
type ChassisMap = IntMap.IntMap Chassis
type AmmoMap    = IntMap.IntMap Ammo

data World
  = World {
      _worldState          :: !WorldState     -- track the phases the world is in
    , _worldCounter        :: !ObjectId       -- a simple incremented counter for object ids
    , _worldPlayers        :: !PlayerMap      -- track player api call rates and the like
    , _worldChassis        :: !ChassisMap     -- war machines in the world
    , _worldAmmo           :: !AmmoMap        -- projectiles currently in the world
    , _worldConfigurations :: !Configurations -- yaml loaded war machine data for configuration phase
    , _worldDefaultChassis :: !Chassis        -- the default chassis
    }

makeLenses ''World
```

The game loop looks like this:

```haskell
{-# LANGUAGE BangPatterns #-}

runWorld :: Server -> GameTime -> UTCTime -> IO ()
runWorld server !time before = do
  now <- getCurrentTime
  let dt = diffTime now before
  world0 <- readTVarIO (server^.serverWorld)
  when (world0^.worldState == GamePhase) $ do
    sim0 <- atomically $ iterateWorld server time dt
    mapM_ (broadcastSimulationMessages server) (sim0^.simulationMessages)
  threadDelay loopWait
  let next = time + dt
  runWorld server next now
  where
    diffTime = (realToFrac .) . diffUTCTime
```

I use `BangPatterns` because I want to force the `next` time value to be evaluated each tick.  While profiling for an unrelated space leak it turns out `let next = time + dt` was causing a small heap space leak because there are cases where the time value never gets evaluated downstream for awhile.

The main loop itself runs at about 60 updates per second.  This is good enough to smoothly move the simulation state forward and notify any listening external clients of the action going on.

```haskell
iterateWorld :: Server -> GameTime -> DeltaTime -> STM Simulation
iterateWorld server time dt = do
  world0 <- readTVar (server^.serverWorld)
  let sim0  = Simulation [] [] [] NoChassisState
  let (world1, sim1) = runState (stepWorld time dt world0) sim0
  writeTVar (server^.serverWorld) world1
  return sim1
```

One of the more interesting problems we faced was how to get a bunch of simulation state updates generated deep inside game objects and get them to percolate back up to IO so we can send them out to external clients.  We handled this by utilizing the `State` monad with which we thread a `Simulation` data type through each `run*` function that collects all outgoing movement and rotation messages, as well as notifies the `World` that objects have spawned (such as a weapon firing a projectile), etc.

The `World` iteration function is basically this:

```haskell
stepWorld :: GameTime -> DeltaTime -> World -> MTL.State Simulation World
stepWorld t dt world0 = do
  sim0 <- get
  -- a lot of processing steps happen here
  put sim1
  return world1
```

Where each component inside the world that gets iterated over is also passed the `Simulation` state so it can append any new messages to be acted upon.

## Actor Class

The final topic of this post is about our `Actor` class.  This is quite useful for providing a common interface which game objects must implement so that they can move, rotate, or collide.

The code is so incredibly dense I'll post the entire thing here.

```haskell
module Game.Actor where

import           Linear
import           Control.Lens

import           Game.Types
import           Game.Utils

class Actor a where
  getId         :: a -> ObjectId
  setId         :: ObjectId -> a -> a
  getObjectType :: a -> ObjectType
  setObjectType :: ObjectType -> a -> a
  getPosition   :: a -> Position
  setPosition   :: Position -> a -> a

class (Actor a) => Movable a where
  getAcceleration :: a -> Acceleration
  setAcceleration :: Acceleration -> a -> a
  getVelocity     :: a -> Velocity
  setVelocity     :: Velocity -> a -> a
  getVelocityMax  :: a -> Velocity

class (Actor a) => Rotatable a where
  getRotation       :: a -> Rotation
  setRotation       :: Rotation -> a -> a
  getRotationRate   :: a -> Float
  setRotationRate   :: Float -> a -> a
  getRotationTarget :: a -> Float
  setRotationTarget :: Float -> a -> a

class (Actor a) => Collideable a where
  getCollider :: a -> Collider
  setCollider :: Collider -> a -> a

moveActor :: (Movable a, Rotatable a) => DeltaTime -> a -> a
moveActor dt m =
  setVelocity vel . setPosition pos $ m
  where
    -- velocity
    vel = if getVelocity m < getVelocityMax m
          then getVelocity m + getAcceleration m * dt
          else getVelocity m
    -- position
    dir = rotate (getRotation m) (newPosition 0.0 0.0 1.0) -- forward on Z vector
    pos = getPosition m + (vel * dt *^ dir)

rotateActor :: (Rotatable a) => DeltaTime -> a -> a
rotateActor dt r =
  setRotation rot r
  where
    mid = axisAngle (newPosition 0.0 1.0 0.0) (toRadians $ getRotationTarget r)
    rot = slerp (getRotation r) (getRotation r * mid) (dt * getRotationRate r)

isColliding :: (Collideable a, Collideable b) => a -> b -> Bool
isColliding aA aB = do
  let (cAr, cAh) = getColliderValues (getCollider aA)
  let (cBr, cBh) = getColliderValues (getCollider aB)
  let cPa = getPosition aA
  let cPb = getPosition aB
  abs (distance (cPa^._xz) (cPb^._xz)) <= cAr + cBr && abs ((cPa^._y) - (cPb^._y)) <= cAh + cBh
```

Not all objects are created equal, so we provide more strict definitions of `Actor` as well.  A building will clearly not be `Moveable` nor `Rotatable`, while a base turret may be `Rotatable` but not `Moveable`.

Our `isColliding` function simply does a naive cylinder collision check.  We then take that data and calculate the direction the impact occurred, and in the case of a projectile impacting a war machine we then utilize that direction data to calculate which component the damage applies to, and which direction it came from so we can reduce the correct forward or rearward facing armor values.  In the future we'd like to replace this with actual model data so we can get pixel perfect collisions, however, we think the naive approach will work fine until then.

Each game tick a `Moveable/Rotatable` object in the scene is ran with `moveActor dt . rotateActor dt $ actor` and if anything occurred the correct `Simulation` message is generated and passed up to the main loop.  We also check each object against every other object in the scene to see if they collided and pass up relevant messages here as well.

## Conclusion

We were able to split our game server code up so that all IO related tasks happen in `Server` while all the simulation logic occurs in the pure `World`.  This makes it incredibly easy to reason about how all of our simulation logic runs and test and debug any logic issues we come across.  We also use various types of protobuf messages which allow the player's AI to interact with our server and control their war machines.  Finally, we've created an `Actor` class which simulation objects must implement so that they can interact inside the game world.  The logic for performing movement, rotations, and collision detection was incredibly easy, with many thanks to Edward A. Kmett and his powerful [linear](https://hackage.haskell.org/package/linear) library.

I don't have any plans for another post at this time, however, if there's anything you'd like to hear more detail about please feel free to reach out to me over [twitter](https://twitter.com/mojobojo).
