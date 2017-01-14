---
title: Haskell Bake and Kubernetes
---

## Background

I work at a relatively large 150 employee Japanese social game company as the Lead Infrastructure Engineer on one of their various game projects. My primary duties include building server infrastructure, automating all the things, helping the server programming team out so they can focus on their jobs and not worry about the rest, and handling things if it all goes south. (DevOps, SRE, etc.)

## Discussion

We have a small Kubernetes cluster which I run in order for the client team to be able to test their changes against the stable staging server build and the ever changing development build. Since these server instances see quite a bit of idle time I've decided to utilize their extra resources for other things my team needs, such as a build server. I wanted to avoid using memory hungry CI/CDs like Jenkins or Bamboo, and after having recently stumbled across a Haskell CI project called Bake decided to give it a shot.

This post describes how I have leveraged [Bake](https://github.com/ndmitchell/bake) and the power of Kubernetes to create an easily scalable build system which does automated testing (and eventually docker builds and deploys) for my server team. One of the exciting things about this setup is that it becomes relatively trivial to create build servers tuned towards what your project needs. In my case our server team uses Python's Django which does integration tests against the database. Utilizing Kubernetes *pods*, I was able to link a MySQL container along with a Bake client container inside the same *pod* to use for locally ran integration tests.

Note: This post is fairly code heavy and doesn't go into deep philosophical reasons on why I made various decisions. It is meant it be more of a guide to help others bootstrap their own Bake build + K8S cluster.

## Haskell

I'm a huge Haskell fan. If I wasn't working primarily on server infrastructure I could see myself coding in this beautiful language instead. Unfortunately there's little chance of buy in where I currently work, so sneaking it into side projects is the next best thing.

### Step 1 - Build Server

For starters we need our Bake CI server. The project maintainer is still actively working on the project (currently v0.5), although there's not much in the way of examples on advanced usage and configuration yet. That said it's fairly trivial to chain actions together and use `cmd` to run what you want.

```haskell
module Main where

import Data.Monoid ((<>))
import System.Environment

import Development.Bake
import Development.Shake.Command

-- Lifted from Bake's src/General.Extra.hs
putBlock :: String -> [String] -> String
putBlock title body = unlines $
  let s = "-- " ++ title ++ " --" in
  (s ++ replicate (70 - length s) '-') :
  body ++
  [replicate 70 '-']

data Action = RunTests deriving (Show, Read)

instance Stringy Action where
  stringyTo   = show
  stringyFrom = read

-- Our team uses python - run the tests here
runTests :: IO ()
runTests =
  cmd (Cwd "/tmp/my-project/src") "python3.5 ./manage.py test --noinput"

execute :: Action -> TestInfo Action
execute RunTests =
  run $ do
    incrementalStart
    runTests
    incrementalDone

allActions :: [Action]
allActions = [RunTests]

-- ChatWork is a popular web chat in Japan - push notifications to here
ovenNotifyChatWork :: String -> String -> Oven s p t -> Oven s p t
ovenNotifyChatWork t r =
  ovenNotifyAdd $ \a s b ->
    let m = putBlock "Bake Build" [ "Author: " <> a, "Result: " <> s, b]
    in unit $ cmd "curl -X POST -H" [ "X-ChatWorkToken: " <> t ]
                                    "-d" [ "body=" <> m ]
                                    ("https://api.chatwork.com/v1/rooms/" <> r <> "/messages")

main :: IO ()
main = do
  -- grab the ChatWork tokens read in from the Kubernetes Deployment file
  token <- getEnv "CHATWORK_TOKEN"
  room  <- getEnv "CHATWORK_ROOM"
  bake $
    ovenNotifyChatWork token room $
    ovenPretty $
    ovenIncremental .
      ovenGit "git@bitbucket.org:my-company/my-project.git" "master" (Just "/tmp/my-project") $
    ovenNotifyStdout $
    ovenTest (return allActions) execute
    defaultOven { ovenServer = ("127.0.0.1", 5000) }
```

There's not really much to discuss in regards to the above except that more functionality will be added in the future, specifically building and pushing new docker builds out to the development cluster. If all of the tests pass it will automatically merge into `master`, and the result published to our ChatWork build channel.

Currently the body of the message posted to ChatWork comes out as html. I need to examine this a little further to see how to make better output.

### Step 2 - Hook Server

Next is the web hook server. Our company uses [https://bitbucket.org](https://bitbucket.org) and I needed something that could act as an intermediary between it the Bake server for web hook triggers. I rolled my own with Servant.

Right now it only processes Push triggers which were applied to our `develop` branch.

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE TemplateHaskell #-}

module Main where

import Control.Lens
import Control.Monad (mzero)
import Data.Aeson
import qualified Data.ByteString.Lazy.Char8 as BS (unpack)
import Data.Monoid ((<>))
import Data.Text
import Network.HTTP.Simple (httpLBS, parseRequest, getResponseBody)
import qualified Network.Wai.Handler.Warp as Warp
import Servant

-- The bitbucket Push JSON
-- This is only grabbing the fields we want and not parsing the entire structure

data BBUser = BBUser { _bbuUsername :: Text }
makeLenses ''BBUser
instance FromJSON BBUser where
  parseJSON (Object o) = BBUser <$> o .: "username"
  parseJSON _ = mzero

data BBAuthor = BBAuthor { _bbaUser :: BBUser }
makeLenses ''BBAuthor
instance FromJSON BBAuthor where
  parseJSON (Object o) = BBAuthor <$> o .: "user"
  parseJSON _ = mzero

data BBTarget = BBTarget { _bbtAuthor :: BBAuthor, _bbtHash :: Text }
makeLenses ''BBTarget
instance FromJSON BBTarget where
  parseJSON (Object o) = BBTarget <$> o .: "author" <*> o .: "hash"
  parseJSON _ = mzero

data BBNew = BBNew { _bbnTarget :: BBTarget, _bbnName :: Text }
makeLenses ''BBNew
instance FromJSON BBNew where
  parseJSON (Object o) = BBNew <$> o .: "target" <*> o .: "name"
  parseJSON _ = mzero

data BBChange = BBChange { _bbcNew :: BBNew }
makeLenses ''BBChange
instance FromJSON BBChange where
  parseJSON (Object o) = BBChange <$> o .: "new"
  parseJSON _ = mzero

data BBPush = BBPush { _bbpChanges :: [BBChange] }
makeLenses ''BBPush
instance FromJSON BBPush where
  parseJSON (Object o) = BBPush <$> o .: "changes"
  parseJSON _ = mzero

data BBData = BBData { _bbdPush :: BBPush }
makeLenses ''BBData
instance FromJSON BBData where
  parseJSON (Object o) = BBData <$> o .: "push"
  parseJSON _ = mzero
  

-- The Bake CI server - localhost due to Kubernetes magic
buildURL :: Text -> Text -> Text
buildURL a p = "http://localhost:5000/api/add?author=" <> a <> "&patch=" <> p

type BuildApi =
       "check" :> Get '[JSON] Text
  :<|> "build" :> ReqBody '[JSON] BBData :> Post '[JSON] Text

buildApi :: Proxy BuildApi
buildApi = Proxy

server :: Server BuildApi
server = checkOK :<|> receiveBuild

checkOK :: Handler Text
checkOK = return "OK"

receiveBuild :: BBData -> Handler Text
receiveBuild bb =
  -- Check for the branch we want (I was initially merging *all* pushes regardless of branch, oops)
  if b == "develop"
  then do
    u <- parseRequest (unpack $ buildURL a h)
    r <- httpLBS u
    return $ pack $ BS.unpack $ getResponseBody r
  else return $ pack "Not develop branch; Skipped"
  where
    n = bbdPush . bbpChanges . ix 0 . bbcNew
    t = n . bbnTarget
    b = bb ^. n . bbnName
    a = bb ^. t . bbtAuthor . bbaUser . bbuUsername
    h = bb ^. t . bbtHash

main :: IO ()
main = Warp.run 8080 $ serve buildApi server
```

There is some configuration cleanup that can go into this, but so far it does what it is supposed to which is simply ferrying the commit author and hash to the Bake server.

The output from the Bake server gets pushed back to BitBucket and can be reviewed in your project's web hook page.

## Docker

In order to connect all of the pieces of the puzzle together we use Docker containers as the glue. I build the above projects with docker enabled in the `stack.yaml` for each project.

### Build Server Entrypoint

```bash
#!/bin/sh

case "$BUILD_TYPE" in
    "client")
        /usr/local/bin/build-server client \
                                    --host ${BUILD_MASTER_SERVICE_HOST} \
                                    --port ${BUILD_MASTER_SERVICE_PORT}
        ;;
    *)
        /usr/local/bin/build-server server
        ;;
esac
```

### Build Server

Uses the stack-run docker container provided by FPComplete.

This also pulls in all the requirements the server team needs so their Django app can be ran inside the container.

```bash
FROM fpco/stack-run
MAINTAINER "Brian Jones" &lt;jones@my-company.com&gt;

# system
RUN apt -y update
RUN apt -y install wget python3.5 python3.5-dev libmysqlclient-dev libssl-dev libffi-dev gcc

# python
RUN cd /tmp && wget https://bootstrap.pypa.io/get-pip.py
RUN cd /tmp && python3.5 get-pip.py

# pip
ADD requirements.txt /tmp/requirements.txt
RUN pip install -r /tmp/requirements.txt

# bake server
# this path is somewhere deep in .stack-work specific to your setup
ADD .stack-work/???/bin/build-server /usr/local/bin/build-server
ADD run.sh /usr/local/bin/run.sh
RUN chmod +x /usr/local/bin/run.sh

EXPOSE 5000

ENTRYPOINT ["/usr/local/bin/run.sh"]
```

### Hook Server

```bash
FROM fpco/stack-run
MAINTAINER "Brian Jones" &lt;jones@my-company.com&gt;

# hook server
ADD .stack-work/???/bin/build-hook /usr/local/bin/build-hook

EXPOSE 8080

ENTRYPOINT ["/usr/local/bin/build-hook"]
```

## Kubernetes

For the record our cluster is built on top of AWS with the CoreOS team's [kube-aws](https://github.com/coreos/kube-aws) tool.

### Namespaces

We use Kubernetes namespaces to keep our environments logically separated. I started by creating a namespace for our build system.

```bash
$ kubectl namespace create my-project-build
$ kubectl get namespaces
NAME                     STATUS    AGE
default                  Active    17d
kube-system              Active    17d
my-project-build         Active    1d
my-project-development   Active    17d
my-project-staging       Active    17d
```

### Master Build and Hook Service

This defines two Kubernetes *service*s which run on port 5000 (build) and 8080 (hook). It also defines whatever env variables are needed internally, such as ChatWork tokens for reporting.

One final thing to note is that it loads SSH keys into a container volume via Kubernetes *secrets*. We want to avoid baking them directly into our container so they can be quickly revoked and/or replaced.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: build-master
  labels:
    run: build-master
spec:
  ports:
  - port: 5000
    targetPort: tl-master-port
    protocol: TCP
    name: tls-master-port
  - port: 8080
    targetPort: tl-hook-port
    protocol: TCP
    name: tls-hook-port
  selector:
    run: build-master
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: build-master
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: build-master
    spec:
      containers:
        - name: build-master
          image: my-repo/build-master:latest
          ports:
            - containerPort: 5000
              name: tl-master-port
          env:
            - name: CHATWORK_TOKEN
              value: "???"
            - name: CHATWORK_ROOM
              value: "???"
          volumeMounts:
            - name: build-ssh-keys-volume
              readOnly: true
              mountPath: /root/.ssh
        - name: build-hook
          image: my-repo/build-hook:latest
          ports:
            - containerPort: 8080
              name: tl-hook-port
      volumes:
        - name: build-ssh-keys-volume
          secret:
            secretName: build-ssh-keys
            defaultMode: 256
      imagePullSecrets:
        - name: us-west-2-ecr-registry
```

The command for loading your project SSH keys into Kubernetes:

```bash
ssh-keyscan bitbucket.org >> /tmp/known_hosts
kubectl --namespace build delete secret build-ssh-keys
kubectl --namespace build create secret generic build-ssh-keys \
	--from-file=id_rsa=tool/build-server/.ssh/tbs \
	--from-file=id_rsa.pub=tool/build-server/.ssh/tbs.pub \
	--from-file=known_hosts=/tmp/known_hosts
```

I add known_hosts this way so I don't have to bake them it into the docker container and can update them when I do a security audit.

You **probably** don't want to be loading known_hosts this way without manually verifying the SSH fingerprints yourself first.

Consult the Kubernetes documentation on *secrets* to get an idea of how you may want to limit access via *services accounts*.

### Client Build Service

The main thing to note is that we tie the Bake client container together with a MySQL container so that it can run the server team's integration tests. One could easily swap that out for another database, maybe add a cache server container to the pod, and whatever else is required to get all of your tests passing. Each of these things are exposed locally as if they were all running on the same server.

I use "127.0.0.1" instead of "localhost" for MySQL because it expects to be connecting to a socket if the latter is defined.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
data:
  mysql-config: |
    [mysqld]
    skip-character-set-client-handshake
    character-set-server=utf8
    collation-server=utf8_general_ci
    init-connect = SET NAMES utf8
    [client]
    protocol = TCP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: build-client
spec:
  replicas: 4
  template:
    metadata:
      labels:
        run: build-client
    spec:
      containers:
        - name: build-client
          image: my-repo/build:latest
          env:
            - name: BUILD_TYPE
              value: "client"
            - name: CHATWORK_TOKEN
              value: "???"
            - name: CHATWORK_ROOM
              value: "???"
            - name: MYSQL_DATABASE
              value: "my-project"
            - name: MYSQL_USER
              value: "my-project"
            - name: MYSQL_PASSWORD
              value: "my-project2016"
            - name: MYSQL_HOST
              value: "127.0.0.1"
            - name: MYSQL_PORT
              value: "3306"
          volumeMounts:
            - name: build-ssh-keys-volume
              readOnly: true
              mountPath: /root/.ssh
            - name: mysql-config-volume
              mountPath: /etc/mysql/conf.d/
        - name: mysql
          image: mysql:5.7
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "my-project"
            - name: MYSQL_DATABASE
              value: "test_my-project"
            - name: MYSQL_USER
              value: "my-project"
            - name: MYSQL_PASSWORD
              value: "my-project2016"
            - name: MYSQL_CHARSET
              value: "utf8"
          ports:
           - containerPort: 3306
             name: mysql
          volumeMounts:
           - name: mysql-config-volume
             mountPath: /etc/mysql/conf.d/
          args:
            - --character-set-server=utf8
            - --collation-server=utf8_unicode_ci
      volumes:
        - name: build-ssh-keys-volume
          secret:
            secretName: build-ssh-keys
            defaultMode: 256
        - name: mysql-config-volume
          configMap:
            name: mysql-configmap
            items:
              - key: mysql-config
                path: config-file.cnf
      imagePullSecrets:
        - name: us-west-2-ecr-registry
```

This service creates and maintains 4 build clients. As your commit volume and test time increases you can simply use kubectl commands to scale out and keep your developers happy.

### Ingress

The next piece of the puzzle is allowing outside world access to our build cluster. There's numerous ways to handle this, I've chosen to use an Ingress and expose it behind a LoadBalancer.

First we need an *ingress controller* and a default backend for it. The following snippets are available via the Kubernetes contrib GitHub repo.

* [rc-default.yaml](https://github.com/kubernetes/contrib/blob/master/ingress/controllers/nginx/examples/default/rc-default.yaml)
* [default-backend.yaml](https://github.com/kubernetes/contrib/blob/master/ingress/controllers/nginx/examples/default-backend.yaml)

And finally our actual *ingress* which routes domain names to our backend services.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: build-ingress
spec:
  rules:
  - host: build.my-company.net
    http:
      paths:
      - path: /
        backend:
          serviceName: build-master
          servicePort: 5000
  - host: hook.my-company.net
    http:
      paths:
      - path: /
        backend:
          serviceName: build-master
          servicePort: 8080
```

The last step is to expose this to the world behind a LoadBalancer.

```bash
$ kubectl --namespace toulove-build expose rc nginx-ingress-controller \
  --type=LoadBalancer --port=80 --target-port=80
```

In my specific case this makes an ELB on AWS, after which I lock it down with a Security Group which only allows access from my company and the bitbucket web hook IP range.

## Conclusion

![](/images/bake-build.png)

My team's commits are successful being sent to the cluster, divvied out to the Bake clients, tested, and the results published to our build chat. Success!

If you'd like to get in touch with me and ask questions don't hesitate to convert the domain this blog is attached to into an email address and reach out to me.
