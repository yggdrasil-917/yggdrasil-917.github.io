---
title: "Choosing A Game AI System - Behavior Trees"
date: 2024-10-12T14:06:14-07:00
draft: false
toc: false
images:
tags: 
  - ai
  - editor
---

* [Behavior Trees](#behavior-trees)
* [Node Types](#node-types)
* [Implementation](#implementation)
* [Authoring BTs](#authoring-bts)
* [Downsides to BTs](#downsides-to-bts)


Once it was time to add an AI system to power the enemy AI for the game being made in Tempest I automatically defaulted to use behavior trees for it. This was mostly because it is what I was most familiar with at the time but wasn't sure if it was the right fit for a tactical RPG game that has various job classes that can be mixed and matched. Think of a job system like what is in Final Fantasy Tactics for an example. Long story short is that after implementing behavior trees I was not convinced that it was the right choice to move forward with. This post will go over some implementation details and how I got to the decision to look for something else.

## Behavior Trees
Behavior trees tend to be an attractive solution for game AI in part due to its hierarchical node structure that makes it easy to understand and construct complex behavior for AI agents. It's hierarchical structure is also important when talking about performance implications as large parts of the tree could be skipped if the agent decides not to go down a certain branch given the state of the world and other factors. AI designers also like the hierarchical structure as it makes it trivial to organize an agent's decision making process. If there are missing AI nodes, designers can request for an AI programmer to implement the desired features. Since the trees are built from a modular and reusable node API, programmers can implement new nodes with it and automatically have them usable in any behavior tree. So it's easy to see why behavior trees are so appealing in game development.

## Node Types
On the topic of behavior tree nodes, lets go over the main fundamental types supported in Tempest. These nodes form the building blocks on which newer more complex nodes can inherit from.

### Control Nodes
These nodes can dictate the flow of execution and can have one or more child nodes.

* **Selector Node**: This node will execute its children from left to right until one succeeds so if a child returns **failure** then it moves to the next. If all children fail then the selector node returns a **failure** value.
* **Sequence Node**: Similar to a Selector node in that it executes its children from left to right but stops at the first **failure**.
* **Parallel Node**: Can execute all its children simultaneously so you can have an enemy that is firing its weapon while also moving at the same time.

### Action Nodes
Action nodes represent specific actions the agent can perform and usually have a return value of **success**, **failure**, or **running** when they are executed. These are also the leaf nodes in a behavior tree. Some concrete examples include nodes for moving a character to a location or performing an attack.

### Condition Nodes
Condition nodes check for a specific condition and return **success** or **failure** based on the result of the condition evaluation. Some examples include checking to see if an enemy is in attack range or checking if the agent has low health.

### Decorator Nodes
Decorator nodes modify the behavior of their child nodes. For instance, they can repeat a certain action node N times or add a delay in between actions.

## Implementation
A behavior tree in Tempest contains an array to hold all of its nodes and has a pointer to an optional blackboard instance as you don't always need agent specific data to simulate the tree. I'll talk a bit more about blackboards later in this post. Tempest uses a factory pattern to create new nodes trivially. There's a macro that a node class can use to declare the required functions that will later be used to self register with the behavior tree node factory on application start. It essentially hashes the name of the node class to create a hopefully unique value that serves as a key to the map the factory has with the value being a wrapper to function that knows how to construct that node. The hashing function used has already been covered before in the [asset system](https://yggdrasil-917.github.io/posts/asset-system/asset-system/) post so check it out if you haven't already. Suffice to say it uses the [FNV-1A](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) hashing function which is very fast and has low collision rates. 

Statically self registering to a factory object is a pattern used a couple times throughout the engine and for the most works really well. There is one annoying issue with it though and it is in regards with c++ compilers being "smart" and removing what it thinks is unused code at compile time. This can happen when statically initialized variables are not referenced anywhere in the codebase. There are two solutions to this that I am aware of, one is to create a dummy function that uses those static variables and have it called somewhere in the codebase effectively tricking the compiler into keeping the code. The other is to configure compilation arguments to tell the compiler not to strip things in a library and this is what Tempest uses on the libraries where it is required.

Simulating a behavior tree is not like calling a function. Any part of the tree can take any number of ticks to complete and because we have to worry about framerate we can't wait until the entire tree has been simulated. In practice what this means is that you need to yield behavior tree execution and perform the simulation over several frames. When we return to update the simulation on a new frame we keep track of the currently active node so we don't need to traverse the entire tree to find it every single time we simulate it. To give an example of this, consider the **MoveToTile** node which all it does is tell the movement system to begin moving the character to the target tile. The node will return a value of **running** so the behavior tree simulation function returns in order for other things to run on that thread. Every tick the tree is simulated it returns **running** so long as the movement system is animating the movement. Once it finishes moving the character, the **MoveToTile** node will return a value of **success** and the simulation moves on to the next node or finishes if that was the last node.

Since behavior trees are part of the asset system, they are shared among all agents referencing the same asset. This poses a problem if we want to slightly alter the tree at runtime. Think of adding a cooldown timer in between action nodes that is a different length based on the enemy type. The solution to this problem comes in the form of blackboards and the ability to allocate dynamic data for a node if instance specific data is required.

### Blackboard
So what are blackboards? These essentially serve as the memory for the AI agent's brain. It is usually implemented as a key-value map where the key tends to be a string and the value can be a lot of different types such as bools, ints, or even pointers. These days you can find blackboards being used in areas not related to AI as you can find them in any system that may need a key-value map that can be updated at runtime. You can see an example of that in Unity's Shader Graph. In Tempest, a blackboard is treated as an asset so the Asset Manager will own it. This means multiple behavior trees can make use of it and could in theory share data that way or even communicate with one another.

In the current version of the Tempest engine, the blackboard would probably be implemented through the scriptable object interface but it was implemented long before that existed. Due to that the blackboard asset has a different way of creating and editing its data but it is still kept simple. Since it is just a map the asset is defined in a text file with a specific key-value pair format where you specify the string for its key and the value right next to it. An example of what a blackboard asset looks like in Tempest is below with some comments left in as well. In the editor the importer will parse the text and convert it to a fast binary format that is used at runtime.

```
# Format version number
1

# Format is keyName optionalDefaultValue
# for float[3] the values are separated by whitespaces
# if default value is omitted then it assigns a value of 0 to it

: Bool
myBool true

: Int
myInt 123

: Float
myFloat 0.123f

: Vector3
myV3 1.1 2.5 3.8

: Ptr
myObject
```

## Authoring BTs
Short of only doing very shallow and simple behavior trees, you will need a proper authoring tool to create the tree. In Tempest I opted to create a UI tool built into the editor that takes a lot of inspiration from the behavior tree editor found in Unreal Engine. Prior to this however, Tempest did not have a node editor system and instead of making my own I decided to look for a library that does that. Since I'm using Dear ImGui for the editor UI and considering it is quite popular these days I assumed a node editor library using that would exist. Once I found one that looked promising it took about half a day to integrate into the engine and now that forms the foundation for the behavior tree editor in Tempest. 

Here's a screenshot of the behavior tree editor implemented in Tempest and you will see plenty of similarities to the Unreal one. This shows the basic steps the enemy AI will go through.

![Pawn BT](/images/behavior-trees/PawnBehaviorTree.png "PawnBehaviorTree")

It uses the same layout conventions as the one found in Unreal Engine. You'll find the root node at the very top of the tree and node execution follows a pre-order traversal approach if no nodes are skipped. The screenshot also has the ordinal numbers rendered to show what the execution order would be. If you want to change the order of execution then all you have to do is move the nodes just like you do in Unreal Engine so it is fairly easy and familiar.

Another feature in the editor is modifying any properties specific to a node that has been selected. For instance, if you have a node for pausing execution for a certain amount of time you'd be able to specify that amount when the node is selected and that data will be serialized with the asset. 

We also need to worry about serialization but thankfully, the [previously](https://yggdrasil-917.github.io/posts/serialization/serialization/) outlined serialization method works well here. The only difference is that it also has to serialize some editor only data with the behavior tree asset. That editor only data is node positions and links used to construct the tree seen in the editor tool. That pretty much amounts to a long string containing node indices layed out hierarchically to define the link structure, the 2D positions in the canvas, and other small things like camera zoom level. This editor only information can be stripped from the asset when cooking a build so we can optimize file size that way.

## Downsides to BTs
So after doing all of this work to support behavior trees I was still a little unsatisfied at the results they were providing in the game. Granted I could improve it a lot more if I spent more time making game specific extensions but I felt like I might be fighting against what makes behavior trees what they are. For example, after setting up a basic behavior tree for enemies the actions they took felt very predictable and quickly got boring. In part this is due to the static nature of a behavior tree. They are created in the editor and the game just loads it as is. If you add enough conditionals or branches to the tree this predictable pattern is hidden a lot better but otherwise people can figure them out rather quickly. In a game where strategy is important I feel like this is a pretty big limitation and can lead to very easily manipulating the enemy AI.

Something I desire to reach is a system that can provide opportunities to create emergent behaviors. I don't feel like this is possible with behavior trees in part because of its static nature. The most I feel I can do is provide some randomness to certain decision making processes to make it look a bit more emergent but that's not enough. At the end of the day nobody is forcing me to use behavior trees so I am free to explore other options instead of trying to make a solution fit my criterias.

The last thing I'll mention is that the current implementation does require a lot of code to be written. Not just for the runtime usage but also all the editor specific implementations that need to be done before a new node can actually be used. Some of the burden would be reduced if there happened to be a good reflection API to help automate certain things but alas that is not really available. I also don't hold out much hope that reflection built into c++ would be the answer given how many times that has been delayed.

In the end, I opted to start looking for a different alternative and after researching for a bit I came to a couple of other options that could fit my criterias. In the next post on AI I will go over my implementation of Utility AI and some of its pros and cons.