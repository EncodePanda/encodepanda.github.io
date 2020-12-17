---
title: Getting acquainted with Lens (part 1)
desc: All you wanted to know about Lens but dare to ask
author: Pawel Szulc
tags: haskell, lens
icon: lens.jpg
disclaimer: Content of this blog and the source code is <a href="https://github.com/EncodePanda/bp-getting-acquainted-with-lens">available on Github</a>. Repository is divided into small commits so that you can follow along if you prefer jumping straight into the code. <br/> This post is based on a <a href="https://www.youtube.com/watch?v=LBiFYbQMAXc">talk</a> I did at Haskell.Love 2020 <br/> I want to personally thank <a href="https://twitter.com/cateroxl">@cateroxl</a> for proof-reading and helping me fix all the typos I've created while writing this post. <a href="https://twitter.com/cateroxl">@cateroxl</a> you rock!
---

### Introduction

In this post we will explore a concept of a Lens. More concretely the [Lens library](https://hackage.haskell.org/package/lens).

1. What problem they are trying to solve?
2. How we can use them?
3. How they are being implemented?

### What problem they are trying to solve?

It is my observation that any newcomer to the Haskell ecosystem, who gets excited with the language, stumbles upon two limitations of the language design. Mainly:

1. Record syntax
2. Strings

Those two are suprising to newcomers, especially because both record syntax and strings are considered a no-brainer in almost every other programming language. Writing about string representation in Haskell deserves a blog post on its own. Today we want to focus on **record syntax**, trying to understand what are its limitations and main source of frustration.
And frustrating indeed it is. Below are a few quotes scrapped from the Internet to back that claim.

> *"The record system is a continual source of pain"*
- Stephen Diehl

> *"What is your least favorite thing about Haskell? Records are still tedious"*
- 2018 State of Haskell Survey

> *"Shitty records."*
- Someone on reddit

### An example

[Someone famous](https://en.wikipedia.org/wiki/Linus_Torvalds) once said "Talk is cheap, show me the code". In that sprit let's explore an example project in which those problems are clearly visible.

We will use the latest version of GHC Haskell:

```bash
The Glorious Glasgow Haskell Compilation System, version 8.10.2

```

and a standard cabal project:

```bash
cabal-version: 2.1

name:           bp-getting-acquainted-with-lens
version:        0.1.0.0
license-file:   LICENSE
author:         Paweł Szulc
maintainer:     Paweł Szulc
copyright:      Paweł Szulc
category:       BlogPost
build-type:     Simple

extra-source-files:
    README.md

source-repository head
  type: git
  location: https://github.com/EncodePanda/bp-getting-acquainted-with-lens

library
  exposed-modules:
      EncodePanda.Lens
  hs-source-dirs:
      src
  build-depends:
        base >=4.7 && <5
  default-language: Haskell2010

```

Imagine we are writing a tool allowing conference organizers to maintain their events. We have a datatype `Conference`:

```haskell
data Conference = Conference
  { organizer :: Organizer
  , speakers  :: [Speaker]
  } deriving Show

```

where `Organizer` is:

```haskell
data Organizer = Organizer
  { name    :: Name
  , contact :: Contact
  } deriving Show

```

and `Speaker` is:

```haskell
data Speaker = Speaker
  { slidesReady :: Bool
  } deriving Show

```

Organizer has a `Name` and `Address`. `Name` is a simple record with `firstName` and `lastName`:

```haskell
data Name = Name
  { firstName :: String
  , lastName  :: String
  } deriving Show

```

and `Address` encapsulates `street`, `city` and `country`:

```haskell
data Address = Address
  { street  :: String
  , city    :: String
  , country :: String
  } deriving Show

```

Now we just need an example of a conference organizer, a value that we could play with in the REPL. While creating this blog post, I could not miss the opportunity to pay my tribute to [Oli Makhasoeva](https://twitter.com/Oli_kitty) - one of the best conference organizers on the planet, the master mind behind such events as [Haskell Love](http://haskell.love) or [Scala Love](http://scala.love/conf).

Let's create a value of type `Organizer` called `oli`:

```haskell
oli :: Organizer
oli = Organizer
  { name = Name "Oli" "Makhasoeva"
  , contact = classified
  }

```

### Fetching values from records

We can observe that both `name` and `contact` are in fact accessor functions that allow us to retrieve values from records:

```Haskell
ghci> :l src/EncodePanda/Lens.hs
ghci> :t name
      name :: Organizer -> Name
ghci> name oli
      Name {firstName = "Oli", lastName = "Makhasoeva"}
ghci> :t contact
      contact :: Organizer -> Contact
ghci> contact oli
      Contact {address = Address {street = "Class", city = "ified", country = "Classified"}, email = "oli@haskell.love"}
```


This can even look nicer if we use `&` operator from `Data.Function`:


```Haskell
ghci> import Data.Function ((&))
ghci> :t (&)
      (&) :: a -> (a -> b) -> b
```


Here we see that `&` is a simple function application, where instead of providing function name and the argument (as we would normally do):

```Haskell
ghci> length [4, 6, 8]
      3
```


we provide first the argument and then the function name:

```Haskell
ghci> [4, 6, 8] & length
      3
```


This allow us to change previous call to `name` from:

```Haskell
ghci> name oli
      Name {firstName = "Oli", lastName = "Makhasoeva"}
```


to:

```Haskell
ghci> oli & name
      Name {firstName = "Oli", lastName = "Makhasoeva"}
```


It does not seem much at first, but you can observe that this approach composes nicely when you want to read the value of a deeply nested record:

```Haskell
ghci> oli & name & firstName
      "Oli"
```


This resembles dot-like record access that is available in other languages. Correctly formatted makes accessing deeply nested values a pleasure experience

```haskell
organizerCountry :: Conference -> String
organizerCountry conf =
  conf & organizer
       & contact
       & address
       & country

```

But when we do a slight modification to our code, where not only `Organizer` has a `name` field:

```diff
 module EncodePanda.Lens2 where
 
 import Data.Function ((&))
 
 -- start snippet conference-datatype
 data Conference = Conference
-  { organizer :: Organizer
+  { name :: String
+  , organizer :: Organizer
   , speakers  :: [Speaker]
   } deriving Show
 -- end snippet conference-datatype
 
 -- start snippet speaker-datatype
 data Speaker = Speaker
-  { slidesReady :: Bool
+  { name :: Name
+  , slidesReady :: Bool
   } deriving Show
 -- end snippet speaker-datatype
 

```

the code suddenly stops compiling:

```haskell
$> cabal build

src/EncodePanda/Lens2.hs:22:5: error:
    Multiple declarations of ‘name’
    Declared at: src/EncodePanda/Lens2.hs:15:5
                 src/EncodePanda/Lens2.hs:22:5
   |
22 |   { name :: Name
   |     ^^^^

src/EncodePanda/Lens2.hs:22:5: error:
    Multiple declarations of ‘name’
    Declared at: src/EncodePanda/Lens2.hs:7:5
                 src/EncodePanda/Lens2.hs:22:5
   |
22 |   { name :: Name
   |     ^^^^
```

There is one trick we could do make it work...Have you guessed it? Yes, it is GHC. We can always add a language extension:

```diff
+{-# LANGUAGE DuplicateRecordFields #-}
 module EncodePanda.Lens2 where

```

And things compile again!

```haskell
$> cabal build

Resolving dependencies...
Build profile: -w ghc-8.10.2 -O1
In order, the following will be built (use -v for more details):
 - bp-getting-acquainted-with-lens-0.1.0.0 (lib) (configuration changed)
Configuring library for bp-getting-acquainted-with-lens-0.1.0.0..
Preprocessing library for bp-getting-acquainted-with-lens-0.1.0.0..
Building library for bp-getting-acquainted-with-lens-0.1.0.0..
[1 of 2] Compiling EncodePanda.Lens ( src/EncodePanda/Lens.hs, /Users/rabbit/projects/bp-getting-acquainted-with-lens/dist-newstyle/build/x86_64-osx/ghc-8.10.2/bp-getting-acquainted-with-lens-0.1.0.0/build/EncodePanda/Lens.o, /Users/rabbit/projects/bp-getting-acquainted-with-lens/dist-newstyle/build/x86_64-osx/ghc-8.10.2/bp-getting-acquainted-with-lens-0.1.0.0/build/EncodePanda/Lens.dyn_o ) [flags changed]
[2 of 2] Compiling EncodePanda.Lens2 ( src/EncodePanda/Lens2.hs, /Users/rabbit/projects/bp-getting-acquainted-with-lens/dist-newstyle/build/x86_64-osx/ghc-8.10.2/bp-getting-acquainted-with-lens-0.1.0.0/build/EncodePanda/Lens2.o, /Users/rabbit/projects/bp-getting-acquainted-with-lens/dist-newstyle/build/x86_64-osx/ghc-8.10.2/bp-getting-acquainted-with-lens-0.1.0.0/build/EncodePanda/Lens2.dyn_o )

```

But when we try to use the `name` function:

```haskell
organizerName :: Conference -> Name
organizerName conference =
  conference & organizer & name
```

The compiler gives us a quick reality check:

```haskell
src/EncodePanda/Lens2.hs:76:28: error:
    Ambiguous occurrence ‘name’
    It could refer to
       either the field ‘name’, defined at src/EncodePanda/Lens2.hs:23:5
           or the field ‘name’, defined at src/EncodePanda/Lens2.hs:16:5
           or the field ‘name’, defined at src/EncodePanda/Lens2.hs:8:5
   |
76 |   conference & organizer & name
```

It seems that we can define multiple records with the same field name, but we are not allowed to use it.

### Is that it? Are we done?

Before we give up, there is fortunately one more trick we can use. It's called `OverloadedLabels` but it requires a bunch of other extendsions to be enabled. We enabled them for the whole project by modifing the cabal file (along with the `DuplicatedRecordFields` that've used before):

```diff
   exposed-modules:
       EncodePanda.Lens
       EncodePanda.Lens2
+      EncodePanda.OverloadedLabels
   hs-source-dirs:
       src
   build-depends:
         base >=4.7 && <5
   default-language: Haskell2010
+  default-extensions:
+       DataKinds
+     , DuplicateRecordFields
+     , FlexibleInstances
+     , MultiParamTypeClasses
+     , OverloadedLabels
+     , TypeApplications

```

What `OverloadedLabels` gives us is a typeclass `IsLabel` from `GHC.OverloadedLabels`:

```haskell
module GHC.OverloadedLabels where
(...)
class IsLabel (x :: Symbol) a where
  fromLabel :: a
```

It is a typeclass that we define for a particular type `a` and a `Symbol` (think of it as type level `String`). As an example we can create an instance of it for a `Speaker` and `"encodepanda"`:

```haskell
instance IsLabel "encodepanda" Speaker where
  fromLabel = Speaker
    { name = Name "Pawel" "Szulc"
    , slidesReady = False
    }

```

And now we can use it as we would normally use any other typeclass:

```haskell
pawel :: Speaker
pawel = fromLabel @"encodepanda"

```

But `OverloadedLabels` comes with a nice syntatic sugar where we can reference just the symbol directly by prefixing the `Symbol` with a hash `#` and let the type inference magic do the rest work for us.

```diff
 
 -- start snippet pawel
 pawel :: Speaker
-pawel = fromLabel @"encodepanda"
+pawel = #encodepanda
 -- end snippet pawel

```

Underneath, it desugars to a function call to `fromLabel`.

### Leveraging OverloadedLabels for record access

We can now take `OverloadedLabels` for a spin, to see if they can help us with our issue. As a reminder, the problem at hand is a fact that (even though we have `DuplicateRecordFields` turned on) we can not reuse duplicated accessor function:

```haskell
$> cabal build

src/EncodePanda/Lens2.hs:22:5: error:
    Multiple declarations of ‘name’
    Declared at: src/EncodePanda/Lens2.hs:15:5
                 src/EncodePanda/Lens2.hs:22:5
   |
22 |   { name :: Name
   |     ^^^^

src/EncodePanda/Lens2.hs:22:5: error:
    Multiple declarations of ‘name’
    Declared at: src/EncodePanda/Lens2.hs:7:5
                 src/EncodePanda/Lens2.hs:22:5
   |
22 |   { name :: Name
   |     ^^^^
```

The idea is to provide different instances for `"name"` and different accessor functions called `name`:

```diff
 import Data.Function ((&))
+import GHC.OverloadedLabels (IsLabel (..))
 
 -- start snippet conference-datatype
 data Conference = Conference
   } deriving Show
 -- end snippet conference-datatype
 
+instance IsLabel "name" (Conference -> String) where
+   fromLabel = name
+
 -- start snippet organizer-datatype
 data Organizer = Organizer
   { name    :: Name
   } deriving Show
 -- end snippet organizer-datatype
 
+instance IsLabel "name" (Organizer -> Name) where
+   fromLabel = name
+
 -- start snippet speaker-datatype
 data Speaker = Speaker
   { name :: Name
   } deriving Show
 -- end snippet speaker-datatype
 
+instance IsLabel "name" (Speaker -> Name) where
+   fromLabel = name
+
 -- start snippet name-datatype
 data Name = Name
   { firstName :: String

```

It is suprising that even though we are reusing `name` functions to implement those typeclasses, the compiler is suddenly happy. In other words: we've build up this boilerplate in order to workaround the fact that different accessor functions that have the same names could not be used. At the same time, that workaround is making use of those functions. Why this is happening is probably a good idea for a different project.
For now lets ignore this suprising effect, and celebarate the fact that we can finally can write a function like this

```haskell
organizerName :: Conference -> Name
organizerName conference =
  conference & organizer & #name

```

So yes, if looking at how accessing records is encoded in Haskell makes you want to do the following

![](../images/eyes.jpg)

I honestly don't blame you. But remember, this is only half of the story, we still need figure out the way how to set values into records.

# Setting values

Imagine we have a value of type `Conference`:

```haskell
conference :: Conference
conference = Conference
  { name = "Haskell.Love"
  , organizer = oli
  , speakers = []
  }

```

We also have two speakers who would love to be part of this conference:

```haskell
pawel :: Speaker
pawel = Speaker
  { name = Name "Pawel" "Szulc"
  , slidesReady = False
  }

marcin :: Speaker
marcin = Speaker
  { name = Name "Marcin" "Rzeznicki"
  , slidesReady = True
  }

```

At first, setting a new value does not seem to be scary.

```
conference { speakers = [ pawel, marcin ] }
```

It seems pretty reasonable - it reuses syntax for the creation of a value. Small, nice, compact.
However, the moment you try to do something a bit more complicated, things get really messy really quickly.

For example, something as simple as making all speakers for a given conference marked as not ready:

```haskell
allSpeakersNotReady :: Conference -> Conference
allSpeakersNotReady conference =
  let
    oldSpeakers = conference & speakers
  in
    conference {
      speakers =
	    fmap (\s -> s { slidesReady = False}) oldSpeakers
    }

```

Or modifing an organizer's email address using a function:

```haskell
changeOrganizerEmail :: (String -> String) -> Conference ->  Conference
changeOrganizerEmail modifyEmail conference =
  let
    oldOrganizer = conference & organizer
    newContact = (oldOrganizer & contact)
      { email = modifyEmail (oldOrganizer & contact & email)
      }
    newOrganizer = oldOrganizer { contact = newContact}
  in
    conference { organizer = newOrganizer }

```

In both example we have to very explicitly say each little tiny detail of how the new value should look like. If that is not imperative programming, I don't know what is.

### Now what?

Is that it? Is there now hope for us? Where do we go from here? You will have to wait for Part 2 to get your questions answered. But fear not, you might get it yet before Christmas :) Stay tooned!
