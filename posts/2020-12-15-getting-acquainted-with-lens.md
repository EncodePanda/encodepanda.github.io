---
title: Getting acquainted with Lens
desc: All you wanted to know about Lens but dare to ask
author: Pawel Szulc
tags: haskell, lens
icon: lens.jpg
disclaimer: Content of this blog and the source code is <a href="https://github.com/EncodePanda/bp-getting-acquainted-with-lens">available on Github</a>. Repository is divided into small commits so that you can follow along if you prefer jumping straight into the code. <br/> This post is based on a <a href="https://www.youtube.com/watch?v=LBiFYbQMAXc">talk</a> I did at Haskell.Love 2020
---

## Introduction

In this post we will explore a concept of a Lens. More concretely the [Lens library](https://hackage.haskell.org/package/lens).

1. What problem they are trying to solve?
2. How we can use them?
3. How they are being implemented?

## What problem they are trying to solve?

It is my observation that any newcommer to the Haskell ecosystem, who gets excited with the langue, stumbles upon two limitations of the language design. Those are

1. Record syntax
2. Strings

Those two are suprising to newcomers, especially because both record syntax and Strings are considered a no-brainer in almost every other programming language. Writing about String representation in Haskell deserves a blog post on its own. Today we want to focus on a *record syntax*, trying to understand what are its limitations and main source of frustration. And frustrating in did it is. Below few quotes scrapped from the Internet to back that claim.

> *"The record system is a continual source of pain"*
- Stephen Diehl

> *"What is your least favorite thing about Haskell? Records are still tedious"*
- 2018 State of Haskell Survey


> *"Shitty records."*
- Someone on reddit

[Someone famous](https://en.wikipedia.org/wiki/Linus_Torvalds) once said "Talk is cheap, show me the code". In that sprit let's explore an example project in which those problems are clearly visible.

We will use the latest version of GHC Haskell

```bash
The Glorious Glasgow Haskell Compilation System, version 8.10.2

```

and a standard cabal project

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
      EncodePanda.Lens2
  hs-source-dirs:
      src
  build-depends:
        base >=4.7 && <5
  default-language: Haskell2010

```

Imagine we are writing a tool allowing conference organizers to maintain their events. We have a datatype `Conference`

```haskell
data Conference = Conference
  { organizer :: Organizer
  , speakers  :: [Speaker]
  } deriving Show

```

where `Organizer` is

```haskell
data Organizer = Organizer
  { name    :: Name
  , contact :: Contact
  } deriving Show

```

and `Speaker` is

```haskell
data Speaker = Speaker
  { slidesReady :: Bool
  } deriving Show

```

Organizer has a `Name` and `Address`. `Name` is a simple record with `firstName` and `lastName`

```haskell
data Name = Name
  { firstName :: String
  , lastName  :: String
  } deriving Show

```

and `Address` encapsulates `street`, `city` and `country`

```haskell
data Address = Address
  { street  :: String
  , city    :: String
  , country :: String
  } deriving Show

```

Now we just need example of a conference organizer, a value that we could play in the REPL with. While creating the example I could not miss the opportunity to pay my tribute to [Oli Makhasoeva](https://twitter.com/Oli_kitty) - one of the best conference organizers on the planet, master mind behind such events as [Haskell Love](http://haskell.love) or [Scala Love](http://scala.love/conf).

With here consent, let's create a `oli` value of type `Organizer`

```haskell
oli :: Organizer
oli = Organizer
  { name = Name "Oli" "Makhasoeva"
  , contact = classified
  }

```

We can observe that both `name` and `contact` are in fact accessor functions that allow us to retrieve values from records

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


This can even look nicer if we use `&` operator from `Data.Function`


```Haskell
ghci> import Data.Function ((&))
ghci> :t (&)
      (&) :: a -> (a -> b) -> b
```


Here we see that `&` is a simple function application, where instead of providing function name and the argument (as we would normally do)

```Haskell
ghci> length [4, 6, 8]
      3
```


we provide first the argument and then the function name

```Haskell
ghci> [4, 6, 8] & length
      3
```


This allow us to change previous call to `name` from

```Haskell
ghci> name oli
      Name {firstName = "Oli", lastName = "Makhasoeva"}
```


to

```Haskell
ghci> oli & name
      Name {firstName = "Oli", lastName = "Makhasoeva"}
```


It does not seem much at first, but you can observe that this approach composes nicely when you want to read value of a deeply nested record.

```Haskell
ghci> oli & name & firstName
      "Oli"
```


This resembles dot-like record access that is available in other languages. Correctly formated makes accessing deeply nested values a pleasure experience

```haskell
organizerCountry :: Conference -> String
organizerCountry conf =
  conf & organizer
       & contact
       & address
       & country

```

But when we do a slight modification to our code, where not only `Organizer` has a `name` field

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

the code suddenly stops compiling

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

There is one trick we could do make it work... Have you guess it? Yes, it is GHC, we can always add a language extension

```diff
+{-# LANGUAGE DuplicateRecordFields #-}
 module EncodePanda.Lens2 where

```

And things compile again!

```haskell
$> cabal build

Up to date

```

But when we try to use the `name` function

```haskell
organizerName :: Conference -> Name
organizerName conference =
  conference & organizer & name
```

compiler gives us a quick reality check

```
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
