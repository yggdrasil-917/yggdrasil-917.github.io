---
title: "Tempest Engine"
date: 2024-06-23T06:40:54-07:00
draft: false
toc: false
images:
tags: 
  - engine
---

# Table of Contents
1. [Introduction](#introduction)
2. [Why Build A New Engine](#why-build-a-new-engine)
3. [Pros](#pros)
4. [Cons](#cons)
5. [Conclusion](#conclusion)



## Introduction

About two years ago I had started building a new engine in C++ with the initial intention of having some base framework I could experiment with and also challenging myself to build a proper game editor since I had yet to do that. I had built a few engines in the past but it was always with the intention of learning more things so I generally avoided using third party libraries. This meant I would implement everything myself or at least the vast majority of it. For this engine though, it was more about making things with it so I was ok with using third party libraries. After a year into the development of the engine, I had a suite of tools that were good enough to convince me to use them to make a game and thus fulfill my intent of creating something with this engine. At that point what I call the Tempest Engine was sort of born. Since then I've been working on the engine whenever the game needed a feature I had not implemented yet.


To get a little bit ahead of myself, I added a screenshot below of what the editor currently looks like while running one of the combat scenarios in the game. I'll eventually go over the various things about the engine in different posts. I should also mention all the art assets are placeholder at this point.

![Editor Screenshot](/images/tempest-engine/editor.png "Tempest Editor")

## Why Build A New Engine

Some of you may ask why use an unproven engine for my game instead of using an off the shelf solution like Unity or Unreal that has been around for a long time. I have many reasons why I'm not going down that route even though I'm familiar with both of those engines. I would say that's also why I don't want to use them for anything other than work these days. I had mainly been using Unity for the last few years and right now is just not a great time to use that engine. Too many packages are in development hell or have been prematurely marked as production ready when they really aren't. That codebase is in a lot of flux right now. Solutions that worked for a certain package version become obsolete very quickly while the problems still linger so a brand new solution needs to figured out. 

As for Unreal, it has all I need but I also don't care for how bloated it is these days. Editor can be fairly unstable depending on what you're doing. Running Unreal makes my PC work so hard it almost sounds like I'm launching a rocket to space with how fast the fans are spinning. I also have never really liked blueprints or how Epic has pushed that workflow on most things in the engine. C++ in Unreal feels worse than it should be and I say that while understanding some of the limitations c++ bring to the scripting side of game development. Namely hot reloading c++ code is a nightmare and will not always work. It might not ever feel as well integrated as c# in Unity. Maybe c++ modules can help change some of that but that's never gonna happen in Unreal without a full rewrite.

There's also something that's been on my mind since I started this and that's the fact that I wouldn't have to pay Epic or Unity money to use my own engine. I may have to pay some amount for some of the libraries I'm using, assuming the game makes enough money that I need to pay them. Considering platforms like Steam take a 30% cut, console platforms do the same, and other things that take away from your total I think every bit you can save helps out in the end.

Bottom line is that I could make my game in either engine but I feel I would be fighting against engine issues due to how they were made or design principles where things have to be done a very specific way that engine forces you to adopt. I'm confident in my abilities to make Tempest good enough to release a game on it some day. That said I'll first talk about some of the pros of building my own engine and then I'll mention some of the cons.

## Pros
Lets start with the obvious pro. I've implemented everything myself. This means I have intimate knowledge on how all the systems in the engine work. There is no blackbox so to speak. Whenever bugs happen I tend to have a good idea of what the problem is and what the solution should be. This results in bugs being fixed in a much quicker pace than if I was using Unity or Unreal. 

Another pro is that I have a pretty good idea of what I want my game to be so the requirements are well defined. Because of that I don't have to rely on a highly generic game engine with loads of features I will never need or use and possibly even deal with problems some of those features may cause during development. Having a good idea of what the game will be also leads to a lean codebase. I keep track of how large the codebase is just for comparing against other engines out there. Without considering third party libraries the engine and game codebase totals around 150K lines of code at the time of this writing. Compare that to the millions of lines of code engines like Unreal and Unity have just for the engine alone.

A lean codebase has several advantages. Keeping it small keeps the learning curve manageable, making it easier to bring other developers on board. Build times are considerably faster than, for example, building Unreal from source so we can make engine modifications without having to wait an excessive amount of time compiling and linking. This also means I don't need to have a beast of a computer to have fast compile times. I currently develop on an almost 5 year old laptop and iteration times are still fast. With that laptop, it would be horrible to compile Unreal from source.

Custom pipeline for everything. You decide how the content is cooked for your game, how the builds get made, shader compilation, etc. I can't stress enough how great it is to be able to make a new build and only have to wait seconds for it to finish. These days engines are taking a very long time to make even the simplest of builds. Take Unity, for example, where using URP and making a build of a simple scene takes a very long time. 

There's plenty of other pros but I'll leave it at that and move on to some of the cons.

## Cons
Custom codebase means it is less mature than long standing engines. There will be problems other engines have already solved. The codebase will likely be lacking in features that others may be accustomed to. Things like editor workflows, keyboard shortcuts, etc. It is up to me to spend time on what is valuable as far as extending the engine feature set.

Currently the engine does not make it trivial to add new shaders. Adding a new shader will usually require adding new c++ code as well. This is in part by design at the moment as I tend to have fewer shaders that rely on dynamic branching for slightly different behaviors instead of doing what most developers do and make a new shader variant for a feature.

## Conclusion
All in all I'm pretty happy with what the engine is capable of and where it's going. At this point it has many of the features I would expect out of an off the shelf engine and I'm confident they will mature over time. There's lots of room for improvement and I'll definitely be talking more about the engine in later posts.