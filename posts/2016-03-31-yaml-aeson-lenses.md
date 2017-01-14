---
title: YAML + Aeson Lenses
---

## Update: A Warning

As pointed out by [/u/tdammers](https://www.reddit.com/r/haskell/comments/4coyqo/yaml_aeson_lenses/d1k7s4l):

> "Note that this isn't a win-win; you're sacrificing some type safety for convenience. One particular advantage of the typed approach is that the only bit of configuration-related code that can possibly fail is the part that parses the YAML file into your typed data structures; any mistakes you make elsewhere will amount to compiler errors. By contrast, the "dynamic" approach defers all schema errors to run time, so if your YAML file happens to not match the expectations of your code, you won't find out until that code actually runs."

In my excitement at not having to spend an hour or two writing boilerplate I overlooked one of the most important reasons I am using Haskell: type safety at compile time. The solution below is still incredibly cool and shows the power of various libraries the Haskell ecosystem has to offer, however, I ended up reverting these changes and spending the rest of the evening writing the more concise type safe version.

# The Story

I was refactoring some [Armored Bits](https://armoredbits.com/) code over the last couple of days to pull some hard coded test values out into the more sane yaml configuration file I've started using recently. As I begun adding more fields to the yaml I realized that this was not going to scale well with regards to my time when fleshing out the corresponding Haskell data structures.

## In The Beginning

The initial yaml wasn't too terribly large.

```yaml
server:
  ports:
    internal: 4000
    external: 5000
  connections:
    internal: 10
    external: 10
  maxPlayers: 10
game:
  deltaTime: 0.016
  force:
    wait: 10
    configure: 5
    start: 5
    end: 300
```

The Haskell code to load this was a tad verbose but still somewhat manageable. I originally set it up to use the lens library so that I could easily drill into the structure with the original yaml field names, therefore manually defined all of the FromJSON instances instead of simply using Generics.

```haskell
-- File: ServerConfig.hs

{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell   #-}

module Data.ServerConfig where

import           Control.Lens
import           Data.Yaml

data ConfigServerPorts
  = ConfigServerPorts {
      _portsInternal :: Int
    , _portsExternal :: Int
    }

makeLenses ''ConfigServerPorts

data ConfigServerConnections
  = ConfigServerConnections {
      _connectionsInternal :: Int
    , _connectionsExternal :: Int
    }

makeLenses ''ConfigServerConnections

data ConfigServer
  = ConfigServer {
      _serverPorts       :: ConfigServerPorts
    , _serverConnections :: ConfigServerConnections
    , _serverMaxPlayers  :: Int
    }

makeLenses ''ConfigServer

data ConfigGameForce
  = ConfigGameForce {
      _forceWait      :: Int
    , _forceConfigure :: Int
    , _forceStart     :: Int
    , _forceEnd       :: Int
    }

makeLenses ''ConfigGameForce

data ConfigGame
  = ConfigGame {
      _gameDeltaTime :: Float
    , _gameForce     :: ConfigGameForce
    }

makeLenses ''ConfigGame

data Config
  = Config {
      _configServer :: ConfigServer
    , _configGame   :: ConfigGame
    }

makeLenses ''Config

instance FromJSON ConfigServerPorts where
  parseJSON (Object v) = ConfigServerPorts
                         <$> (v .: "internal")
                         <*> (v .: "external")
  parseJSON _ = error "Can't parse ConfigServerPorts from YAML/JSON"

instance FromJSON ConfigServerConnections where
  parseJSON (Object v) = ConfigServerConnections
                         <$> (v .: "internal")
                         <*> (v .: "external")
  parseJSON _ = error "Can't parse ConfigServerConnections from YAML/JSON"

instance FromJSON ConfigServer where
  parseJSON (Object v) = ConfigServer
                         <$> (v .: "ports")
                         <*> (v .: "connections")
                         <*> (v .: "maxPlayers")
  parseJSON _ = error "Can't parse ConfigServer from YAML/JSON"

instance FromJSON ConfigGameForce where
  parseJSON (Object v) = ConfigGameForce
                         <$> (v .: "wait")
                         <*> (v .: "configure")
                         <*> (v .: "start")
                         <*> (v .: "end")
  parseJSON _ = error "Can't parse ConfigGameForce from YAML/JSON"

instance FromJSON ConfigGame where
  parseJSON (Object v) = ConfigGame
                         <$> (v .: "deltaTime")
                         <*> (v .: "force")
  parseJSON _ = error "Can't parse ConfigGame from YAML/JSON"

instance FromJSON Config where
  parseJSON (Object v) =  Config
                          <$> (v .: "server")
                          <*> (v .: "game")
  parseJSON _ = error "Can't parse Config from YAML/JSON"
```

## Adding More Fields: Ughhh

Then things got a little out of hand. I added all of the fields we needed to experiment with various mech configurations:

```yaml
server:
  ports:
    internal: 4000
    external: 5000
  connections:
    internal: 10
    external: 10
  maxPlayers: 10
game:
  deltaTime: 0.016
  force:
    wait: 10
    configure: 5
    start: 5
    end: 300
  defaultMech:
    model: Mech Type A
    capacitor: Capacitor Type A
    gyro: Gyro Type A
    reactor: Reactor Type A
    torso:
      model: Torso Type A
      engine: Engine Type A
      armor: Armor Type A
      weapons:
        - model: Weapon Type A
          capacitor:
          ammo: Projectile Type A
      counterMeasures:
        - model: Counter Measure Type A
      actuator: Actuator Type A
      storage:
        ammo:
          model: Projectile Type A
          type: ballistic
          round: basic
          amount: 100
    cockpit:
      model: Cockpit Type A
      computers:
        - model: Computer Type A
      communications:
        - model: Communication Type A
      counterMeasures:
        - model: Counter Measure Type A
      sensors:
        - model: Sensor Type A
      armor: Armor Type A
    arms:
      - model: Arm Type A
        armor: Armor Type A
        weapons:
          - model: Weapon Type A
            capacitor:
            ammo: Projectile Type A
        counterMeasures:
          - model: Counter Measure Type A
      - model: Arm Type A
        armor: Armor Type A
        weapons:
          - model: Weapon Type A
            capacitor:
            ammo: Projectile Type A
        counterMeasures:
          - model: Counter Measure Type A
    legs:
      - model: Leg Type A
        armor: Armor Type A
      - model: Leg Type A
        armor: Armor Type A
```

No way was I going to write out all of those definitions in Haskell!

## Aeson + Lens

I decided that I wanted to use lenses to access this mess without all the boilerplate. After a little research I found this article [Lens/Aeson Traversals/Prisms](https://www.schoolofhaskell.com/user/tel/lens-aeson-traversals-prisms) which explained how to use [Data.Aeson.Lens](https://hackage.haskell.org/package/lens-aeson) against a JSON structure. Since the [yaml](https://hackage.haskell.org/package/yaml) package on hackage uses [aeson](https://hackage.haskell.org/package/aeson) internally it was quite trivial to make this technique work against the above yaml structure.

Instead of loading the target file as yaml and parsing the structure I simply changed it to load it all as a JSON `Value`.

```haskell
-- Before
loadServerConfig :: String -> IO Config
loadServerConfig file = do
  serverData <- decodeFileEither file :: IO (Either ParseException Config)
  case serverData of
    Left err ->
      die $ show err
    Right config ->
      return config
 
-- After
loadServerConfig :: String -> IO Value
loadServerConfig file = do
  handle <- decodeFile file
  case handle of
    Nothing ->
      die "error loading confg.yml"
    Just config ->
      return config
```

And accessing the config fields changed to this:

```haskell
-- Before
(config ^. configServer . serverConnections . connectionsExternal)

-- After
(fromMaybe 10 (config ^? key "server" . key "connections" . key "external" . _Integral))
```

## Conclusion

Not a huge difference, right? I gained some sane defaults by using `fromMaybe`, and the best part, the whole point of this exercise: **I was able to completely get rid of `ServerConfig.hs`**.

