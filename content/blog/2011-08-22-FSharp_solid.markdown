---
author: 7sharp9
date: '2011-08-22'
layout: post
slug: fsharp_solid
status: publish
title: F# and Design principles - SOLID
wordpress_id: '434'
comments: true
categories:
- programming
tags:
- design patterns
- Development
- F#
- solid
description: ""
type: post
---

## SOLID and its relevance to F# 

There has been an increasing amount of exposure for F# and functional
programming lately.  If you come from an object-orientated background a change
in mindset is required when working with functional programming, there is a
lot of misinformation on functional languages and their relationship with
object-orientated design.  In this post we run quickly through SOLID to see if these object-orientated principles apply to F#, and if so, how.<!-- more -->
  
This post assumes you are familiar with SOLID principles, if not here is a
[link](http://en.wikipedia.org/wiki/SOLID_(object-oriented_design).  

Lets take a quick overview of what SOLID stands for:

## Single responsibility principle

This is the notion that an object should have only a single responsibility.

An object-orientated program consists of layers of abstract classes with less
abstract classes layered on top of ones that are more abstract.  Functional
programming is similar, although abstractions are used throughout the design
and are composed into a final solution.  Programming in F# naturally forms
small succinct functions which should have a single purpose, so the single
responsibility rule holds strong here.

## Open closed principle

The notion that “software entities should be open for extension, but closed
for modification”

This comes down to building abstractions via inheritance, and behavior changes
through polymorphism.  Early on a decision must be made on which parts will
change, and which will be fixed.   The open closed principal is geared towards
languages where inheritance is a core concept, inheritance and polymorphism
are not strongly used in F#, so this principle is very weak here.  Composition
and Type augmentation are the core methods for extension in F#.

## Liskov substitution principle

Liskov substitution states “objects in a program should be replaceable with
instances of their subtypes without altering the correctness of that program”.

This principle is all about inheritance - derived types preserving the
specification of base types.  Functional languages like F# do not always use
inheritance, it is used quite rarely and only in certain situations.   The
idioms in the language like referential transparency lean strongly towards
enforcing this principle.  Functional languages also have a heritage in
mathematics and algebraic reasoning so referential transparency is key in this
respect.

## Interface segregation principle

The notion that “many client specific interfaces are better than one general
purpose interface.”

If you are violating single responsibility then your interface will probably
be bloated too with unnecessary properties and methods.  The same rule applies
to F#, keep your interfaces modular and keep indifferent concepts separated.

## Dependency inversion principle

The notion that one should “Depend upon Abstractions. Do not depend upon
concretions.”

  1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
  2. Abstractions should not depend upon details. Details should depend upon abstractions.
Dependency inversion is often solved with concepts such as dependency
injection; common techniques for this involves things like constructor
injection and interface injection.  Inversion of control is often solved with
a factory pattern or a service locator.  From a functional point of view,
these containers and injection concepts can be solved with a simple higher
order function, or hole-in-the-middle type pattern which are built right into
the language.

## Summary

So we can assume that in the functional paradigm:

  * Types should have only a single responsibility.
  * The open closed principle doesn't apply in the majority of cases unless we are using an object-oriented approach.  With functional programming we favour functional composition and type augmentation.
  * The same can be said of the Liskov substitute principle, pure functions uphold this tenant via immutability and referential transparency.
  * Interface segregation also applies and idiomatic F#  produces small concise functions with modular interfaces with a separation of concerns.
  * Dependency inversion is not a relevant issue for F# as the language itself supports higher-order-functions to encapsulate this concept.
So only the '**S**' and the '**I**' parts are relevant in functional
programming.  The other tenants are not fully representative as they stand.
We could probably explore the tenants of good functional design and form a
quirky acronym in the process, but Ill leave that for another time.

Until next time...

