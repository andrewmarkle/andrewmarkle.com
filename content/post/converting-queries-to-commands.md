+++
title = "Michael Feathers - Converting Queries into Commands"
description = ""
date = "2017-07-24"
categories = [
  "Development",
  "OO"
]
+++

This is a key insight into OO from Michael Feather's article on [Converting Queries into Commands](https://michaelfeathers.silvrback.com/converting-queries-to-commands)

<!--more-->

> OO’s primary advantage is decoupling. We send messages to objects and it is up to them to decide what to do. This view of OO comes from Alan Kay and it takes quite a while to internalize. One of the things you have to accept is that when you tell an object to do something there’s no guarantee that it will actually do it. You could, for instance, tell a graphical widget to move but it may not. It could be a null object that simply receives messages and does nothing. These objects can be very useful in systems but you have to maintain the mental frame: what is done depends upon the object. The method calls we make communicate intent but the object bears the responsibility.


