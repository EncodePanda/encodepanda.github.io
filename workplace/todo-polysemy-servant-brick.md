---
title: Building TODO app in Haskell with Polysemy, Aeson, Servant and Brick
desc: Building a TODO app using servant, aeson, polysemy and brick
author: Pawel Szulc
tags: haskell, polysemy, aeson, servant, brick, free monads
---

We will be building a simple TODO application in Haskell. Application will be running as a server that will expose its API through the HTTP endpoint. That API will be consumed by a terminal user interface (TUI) application.

This blog post will explore following libraries:

* [Polysemy](http://hackage.haskell.org/package/polysemy) - a library that allows creating powerful DSLs with ability to nicely separate business logic from implementation details. It gives us power to tame effects in a zero-cost, low-boilerplate manner
* [Servant]([https://hackage.haskell.org/package/servant]) - a library that allows defininig services APIs and serving them
* [Aeson](http://hackage.haskell.org/package/aeson) - a JSON parsing and encoding library
* [Brick](https://hackage.haskell.org/package/brick) - a declarative terminal user interface library

Example project is deliberately simple, so that we can focus on the main objectives:

1. Getting started with the previously mentioned libraries
2. Exploring the concept of separation of concerns in Haskell

What you need to know before reading this post:

1. Have a basic understanding of Haskell syntax.
2. Know how to build a Haskell application using [Stack](https://docs.haskellstack.org/en/stable/README/)
3. Understand the concept of GADTs

Both source code and blog post sources are available at [GitHub](https://github.com/rabbitonweb/bp-servant-polysemy-todo-app). Project is divided into small commits so you can follow along.

# Initial a Stack project

We will start with an empty `stack` project

```bash
.
├── ChangeLog.md
├── LICENSE
├── README.md
├── Setup.hs
├── app
│   └── Main.hs
├── bp-servant-polysemy-todo-app.cabal
├── package.yaml
├── stack.yaml
└── stack.yaml.lock

1 directory, 9 files

```

`stack.yaml`

```yaml
resolver: lts-14.18

packages:
- .

```
`package.yaml`

```yaml
name:                bp-servant-polysemy-todo-app
version:             0.1.0.0
github:              "https://github.com/rabbitonweb/bp-servant-polysemy-todo-ap"
license:             BSD3
author:              "Pawel Szulc (AskBlackRabbit)"
maintainer:          "paul.szulc@gmail.com"
copyright:           "2019 Pawel Szulc"

extra-source-files:
- README.md
- ChangeLog.md

description:         Please see the README on GitHub at <https://github.com/rabbitonweb/bp-servant-polysemy-todo-ap>

dependencies:
- base >= 4.7 && < 5

library:
  source-dirs: src

executables:
  app:
    main:                Main.hs
    source-dirs:         app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - bp-servant-polysemy-todo-app

```

With an empty `Main.hs`

```haskell
module Main where

main :: IO ()
main = undefined

```
# Domain

We start by defining our main domain object `Todo`.

```haskell
module Todo where

import           Data.Text (Text)

data Todo = Todo
  { title     :: Text
  , completed :: Bool
  } deriving (Eq, Show)

```

`Todo` has a `title` field that will be displayed on a list of TODO items. It also has a flag `completed` indicating whether item was already completed or not.

We are using `Text` from [Data.Text](https://hackage.haskell.org/package/text) package, which we have to add to our dependency list

```diff
diff --git a/package.yaml b/package.yaml
index 2d9a1e9..e6a9c23 100644
--- a/package.yaml
+++ b/package.yaml
@@ -14,6 +14,7 @@ description:         Please see the README on GitHub at <https://github.com/rabb

 dependencies:
 - base >= 4.7 && < 5
+- text

 library:
   source-dirs: src

```
We want to serialize and deserialize `Todo` to a `JSON` format. For that, we will use [aeson](http://hackage.haskell.org/package/aeson).

> "A JSON parsing and encoding library optimized for ease of use and high performance."

Let's add `aeson` to our dependency list

```diff
diff --git a/package.yaml b/package.yaml
index e6a9c23..3edbfaf 100644
--- a/package.yaml
+++ b/package.yaml
@@ -15,6 +15,7 @@ description:         Please see the README on GitHub at <https://github.com/rabb
 dependencies:
 - base >= 4.7 && < 5
 - text
+- aeson

 library:
   source-dirs: src

```
Aeson comes with two typeclasses [ToJSON](http://hackage.haskell.org/package/aeson-1.4.6.0/docs/Data-Aeson.html#t:ToJSON) and [FromJSON](http://hackage.haskell.org/package/aeson-1.4.6.0/docs/Data-Aeson.html#t:FromJSON), for serialization and deserialization (respectively). Once instances for those two typeclasses are created for a given type, it is a matter of calling `encode` and `decode` to map between runtime value and JSON format.
Nothing is stopping us to craft those instances by hand for our `Todo` data type (it's actually a good exercise if you want to have better understanding how `Aeson` works under the hood). For our example however we can leverage build-in mechanism in `Aeson` - ability to create default instances for types that have a `Generic` instance.

`Todo` does not have instance of `Generic` yet but we can quickly derive it. We first need to enable `DeriveGeneric` extension globally for our project.

```diff
diff --git a/package.yaml b/package.yaml
index 3edbfaf..8d626aa 100644
--- a/package.yaml
+++ b/package.yaml
@@ -12,6 +12,9 @@ extra-source-files:

 description:         Please see the README on GitHub at <https://github.com/rabbitonweb/bp-servant-polysemy-todo-ap>

+default-extensions:
+- DeriveGeneric
+
 dependencies:
 - base >= 4.7 && < 5
 - text

```

Note: we could have just add pargma to `Todo.hs`, enabling `DeriveGeneric` locally just in that one module. However I find this extension so universal, that I almost always make it default in my projects.

This allows us to derive `Generic` for our `Todo` data type:

```diff
diff --git a/src/Todo.hs b/src/Todo.hs
index f0bbe40..2d7427a 100644
--- a/src/Todo.hs
+++ b/src/Todo.hs
@@ -1,8 +1,9 @@
 module Todo where

-import           Data.Text (Text)
+import           Data.Text    (Text)
+import           GHC.Generics

 data Todo = Todo
   { title     :: Text
   , completed :: Bool
-  } deriving (Eq, Show)
+  } deriving (Eq, Show, Generic)

```
Now we can simply derive instances of `ToJSON` and `FromJSON` for our `Todo`

```haskell
module Todo where

import           Data.Aeson.Types
import           Data.Text        (Text)
import           GHC.Generics

data Todo = Todo
  { title     :: Text
  , completed :: Bool
  } deriving (Eq, Show, Generic)

instance ToJSON Todo
instance FromJSON Todo

```

Let's verify that they work as intended

```haskell
{-# LANGUAGE OverloadedStrings #-}
module TodoAesonSpec where

import           Data.Aeson
import           Data.Text  (Text)
import qualified Data.Text  as T
import           Test.Hspec
import           Todo

spec = describe "Todo Serialization" $ do
  it "should serialize Todo to JSON format" $ do
    -- given
    let todo = Todo "Get milk" False
    -- when
    let json = encode todo
    -- then
    json `shouldBe` "{\"completed\":false,\"title\":\"Get milk\"}"

  it "should deserialize Todo from JSON format" $ do
    -- given
    let json = "{\"completed\":true,\"title\":\"Take trash\"}"
    -- when
    let maybeTodo = decode json
    -- then
    maybeTodo `shouldBe` Just (Todo "Take trash" True)

```
