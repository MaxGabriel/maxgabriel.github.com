---
layout: post
title: Adding Redis to Yesod
---

> It's been a long time since I've worked on the design of this blog. It's honestly much prettier [as a gist](https://gist.github.com/MaxGabriel/c8687688089d094a34a3), if you'd rather read it there.

This is a quick run-through of how I connected to Redis from a Yesod site (which used the default scaffolding). There isn't much specific to Redis here, so this information should apply to connecting to any database or service from Yesod.

## Background: Basics of Hedis

First, a brief intro of the basics of Hedis:

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main where

import qualified Database.Redis as R

main :: IO ()
main = do
  -- Establish a connection to Redis (actually creates a connection pool)
  conn <- R.connect R.defaultConnectInfo

  -- Using one of the connections, run commands against Redis.
  result <- R.runRedis conn $ do
               R.set "key" "value"
               R.get "key"
  print (result == Right (Just "value")) -- True

```

If you're not doing Pub/Sub, that's all there is to using Hedis. For more information, consult the [Hedis Haddocks](http://haddock.stackage.org/lts-5.5/hedis-0.6.10/index.html).

## Installation

Add `hedis` to the cabal file, then install using either Stack or Cabal:

```haskell
build-depends: base
             , yesod
               ...
             , hedis
```

You saw before that we needed to establish a connection to Redis before issuing commands to it. We'll want to access that connection in our `Handler`s instead of making a new connection to Redis for each request. To do this, we'll add the connection pool as a field on the `App` data type, which we can later access in Handlers using `getYesod`:

```haskell
-- Foundation.hs
import qualified Database.Redis as R


data App = App
    { appSettings    :: AppSettings
    , appStatic      :: Static
    , appConnPool    :: ConnectionPool
    , appHttpManager :: Manager
    , appLogger      :: Logger
    , appRedisPool   :: R.Connection -- Our addition
    }
```

When we instantiate our `App` in the `makeFoundation` function, we'll need to add a Redis `Connection` as one of the fields:

```haskell
-- Application.hs
import qualified Database.Redis as R

makeFoundation :: AppSettings -> IO App
makeFoundation appSettings = do
    ...

	appRedisPool <- R.connect R.defaultConnectInfo

	let mkFoundation appConnPool = App {..} -- RecordWildCards will fill out the appRedisPool field
```

Now we can access the `App` in our `Handler`s using `getYesod`, and from it the Redis connection pool:

```haskell
import qualified Database.Redis as R


getHomeR :: Handler Html
getHomeR = do
    app <- getYesod
    let redisPool = appRedisPool app

    liftIO $ R.runRedis redisPool $ do
      _ <- R.set "key" "value"
      R.get "key"
```

At this point you're using Redis from Yesod. The rest of this guide will just improve on the code shown so far.

### Make using Redis from a `Handler` more convenient:

Our current way of calling into Redis is a little tedious. Similar to functions like `runDB`, we'll create a function to remove the boilerplate from calling into Redis:

```haskell
-- I added this to a new module in Handler/Util.hs

handlerRunRedis :: R.Redis a -> Handler a
handlerRunRedis redisAction = do
    redisPool <- appRedisPool <$> getYesod
    liftIO $ R.runRedis redisPool redisAction
```

Usage:

```haskell
handlerRunRedis $ do
      _ <- R.set "hello" "hello"
      _ <- R.set "world" "world"
      hello <- R.get "hello"
      world <- R.get "world"
      liftIO $ print (hello,world)
```


### Allow configuring Redis from a config file

If you look at `makeFoundation`, you can see it takes an `AppSettings` as a parameter to configure the `App` it returns. Hedis uses a `ConnectInfo` record for configuration, so we'll add that as a field to `AppSettings`:


```haskell
-- Settings.hs
import qualified Database.Redis as R


data AppSettings = AppSettings
    { appStaticDir              :: String
    -- ^ Directory from which to serve static files.
    , appDatabaseConf           :: PostgresConf
    -- ^ Configuration settings for accessing the database.
    , appRedisConf              :: R.ConnectInfo
    -- ...
    }
```

The `AppSettings` record is parsed with a `FromJSON` instance defined in `Settings.hs`, so we'll need a way to create a Hedis `ConnectInfo` from JSON as well. Typically you'd derive an instance, but we'd like to use the default values of `ConnectInfo` and just override them as necessary. 

For this I used the `hedis-config` package, which provides a newtype wrapper around `ConnectInfo` called `RedisConfig` (the newtype wrapper is just used to avoid orphan instances). We'll parse a `RedisConfig` from JSON and unwrap it into a `ConnectInfo`:

```
import qualified Database.Redis.Config as RC


instance FromJSON AppSettings where
    parseJSON = withObject "AppSettings" $ \o -> do
        -- ...
        appRedisConf <- RC.getConnectInfo <$> (o .: "redis")

        return AppSettings {..}
```

Now we'll add a `redis` section to our config file:

```yaml
redis:
    max-connections: 100
```

> Note that we're writing `FromJSON` instances even though we're parsing a YAML settings file. This is because the `yaml` package (`Data.Yaml`) parses the YAML into an aeson `Value`, which can then be parsed with `FromJSON` instances. (This just lets us reuse all our existing `FromJSON` instances for convenience).

Back in our `makeFoundation` function, we'll just pass the `ConnectInfo` from the `AppSettings` when creating our connection pool:

```haskell
makeFoundation :: AppSettings -> IO App
makeFoundation appSettings = do
  ...
  appRedisPool <- R.connect (appRedisConf appSettings)
```

To confirm your changes took effect, you can print the configuration before using it:

```haskell
import Debug.Trace


let redisConf = appRedisConf appSettings
traceShowM $ "Redis Conf is: " ++ show redisConf
appRedisPool <- R.connect redisConf
```

## Conclusion

Connecting to Redis from Yesod is pretty simple—the core of it is:

1. Add a connection pool field to your `App` data type
2. Create the connection pool in `makeFoundation`
3. Access the connection pool from `Handler`s using `getYesod`

Even if you don't end up using Redis, this pattern applies to connecting to other services as well, so it should be useful regardless. Enjoy using Redis!

