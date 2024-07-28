---
title: "Scriptable Entity"
date: 2024-07-27T17:58:27-07:00
draft: false
toc: false
images:
tags: 
  - engine
  - scripting
---


The **Entity** class is the base class in Tempest used for anything that can exist in the scene. This is comparable to the GameObject class in Unity or the Actor class in Unreal. Unlike the GameObject class in Unity however, entities do not require a transform component in order to exist in the scene. The class itself is very simple. All it has is an entity handle and a pointer to the scene that owns the entity. The entity handle comes from a framework called [Entt](https://github.com/skypjack/entt) and is what Tempest uses to map the relationship between components and the entity. 

Components in Tempest are structs that only contain data. There is no strict requirement for that in the engine other than personal taste. There are some component types that require logic and are handled by specialized systems that iterate over all of the components of that type. Updating components this way makes it very easy to use the job system to implement the logic update in a multithreaded manner. Component systems do not have to use multithreading though as sometimes there aren't enough components to update so using single threaded updates makes more sense. One such example of a component system is the system updating transform component data where if a transform has moved since the last frame then several matrices specific to that component get updated and cached for later use in the pipeline.

One special component type in the engine is the **ScriptComponent**. This is what makes it possible to script entities. It does this by storing an array of **ScriptableEntity** types which exposes the scripting API to objects deriving from this class. This API provides several hooks that will get called at specific locations within the engine's loop. Hooks like when the script is first created or attached to an entity, simulation callback for update logic, etc. It also provides ways to serialize properties for the entity and some editor functionality like custom property drawing.

Here is a screenshot depicting a high level flow chart of where the scriptable entity user callbacks fit into the engine.

![Execution Order](/images/scriptable-entity/TempestExecutionOrder.png "Execution Order")

## On Scene Load
When a scene is loaded these scriptable entity functions get called. In the editor the scene needs to go into play mode for **OnEnable** and **OnStart** to be called.

* **OnCreate** - Called once when a new instance of an entity is created. Will always be called before **OnEnable** and **OnStart**. It also does not matter if the entity is disabled by default. This is a good place to do any sort of allocation or caching.
* **OnEnable** - If the entity is enabled by default then this gets called right after **OnCreate** and before **OnStart**. After that it only gets called if the script was enabled after being disabled.

## Frame Update
While the scene is in play mode, these functions will get called.

* **OnStart** - Gets called the first time the script is updated and only if it is enabled.
* **OnFixedSimulate** - Can get called a variable number of times per frame depending on framerate or even not at all. Physics updates happen right after this phase in the pipeline.
* **OnSimulate** - Called once per frame and is the main location for any sort of update logic.

## On Object Destruction
* **OnDestroy** - Called at the end of a frame when the entity is destroyed

## On Object Disabled
* **OnDisable** - Called at the end of a frame when the script is disabled. Can also be called before **OnDestroy** when the entity is destroyed.

## Scriptable Entity Execution Order
Scripting API is executed synchronously not unlike what you see in other popular engines. They do have access to the job system so scripts can do multithreaded tasks if they need to. If script execution is synchronous then how do you specify execution order? There is a simple mechanism to solve this problem. When a new script class is created in C++, you can optionally specify its execution priority otherwise it uses the default value of 0. This integer value is used to sort scripts every frame to establish the execution order.
