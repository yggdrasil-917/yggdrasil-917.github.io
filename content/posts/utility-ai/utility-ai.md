---
title: "Choosing A Game AI System - Utility AI"
date: 2024-10-22T12:12:21-07:00
draft: false
toc: false
images:
tags: 
  - ai
---

* [Where To Start](#where-to-start)
* [Response Curves](#response-curves)
* [Consistent Utility Scores](#consistent-utility-scores)
* [Practical Usage Of Scores](#practical-usage-of-scores)
* [Inertia](#inertia)
* [Closing Thoughts](#closing-thoughts)

In a [previous post](https://yggdrasil-917.github.io/posts/behavior-trees/behavior-trees/) I talked about my experience implementing behavior trees in Tempest and evaluating their results. As stated in that post, I did not feel behavior trees are the right solution for the kind of game I'm making so I set out to look for another alternative. Here I will cover one of the two alternatives I decided to look deeper into and that is **Utility AI** or sometimes refered to as **[Utility Theory](https://www.gameaipro.com/GameAIPro/GameAIPro_Chapter09_An_Introduction_to_Utility_Theory.pdf)**.

Generally speaking, behavior trees evaluate their results one node at a time and try to answer the question **"should I execute this node?"**. Simulating the AI this way can mean that their decision making process does not look at the whole picture and given their static nature can lead to sub-optimal decisions and at times predictable. The latter can be a desired property depending on the type of game being made but I would like my AI to be a bit less predictable and possibly produce emergent behaviors to hopefully surprise players. In contrast, utility AI has to process all of its actions and assign scores to them prior to making a decision. Essentially it tries to answer **"what is the best action to take?"** given the state of the world. In a way, this probabilistic model can feel more natural as it sort of mimics how humans reason about problems.

## Where To Start

So first things first, developers have to figure out the set of actions AI agents are allowed to make. In a game that has combat, these actions can be things like **move to location**, **attack**, **heal**, **run away**, etc. To be clear, not all actions have to be decided from the start as it is fairly easy to add new actions. In my case, I started with actions that allowed enemies to move around the map with a goal in mind such as attacking or running away. I feel like that is a good starting point to build your custom AI system around, especially when you are trying to evaluate the effectiveness of the AI system before fully committing to it.

Once you have your actions it is time to figure out the set of scorers to add to each action. In a utility AI system, a scorer is as simple as a mathemathical function to evaluate a specific factor that you can later combine with other scorers under a single action in order to give that action a rating. The higher the rating the more optimal that action is relative to the other possible actions the agent may take.

To give a concrete example of a scorer, lets use the **heal** action. An important thing for the agent to consider before taking that action is how low their health is. It makes no sense for them to heal if they have most of their health. You could use a linear curve to express the urgency to heal in which the closer its health is to 0 then the more likely it will want to heal. It is rare for an action to only have one scorer contributing to its evaluation and in this case just looking at the agent's health would not be enough to evaluate the heal action. It should also consider whether it has a healing item, an ability to heal, or if a teammate close by may need healing more. You can combine all these scorers and create a sophisticated process to decide if the agent should heal. This same process carries over to any other action implemented in the game.

In Tempest, actions and scorers follow a C-like API as I felt an OOP design for this didn't fit that well. Scorers are simple enums and actions are POD structs where they know what scorers matter to them by keeping an enum array. The data needed to configure this is stored in scriptable objects making this whole AI system data-driven. Combine that with asset hot-reloading and we get fast iteration when tweaking values all in the middle of gameplay.

## Response Curves

Now we can dive into world of response curves. These are responsible for providing tunable ways to configure your scoring functions and it is safe to say that there is a ton of material to read on this subject. Creating easy to adjust performant curves is a challenging task on its own and it is something that many areas of mathemathics and engineering desire. I'll only cover a handful here but know that there's a lot more to learn for anyone curious about it. There's going to be plenty of screenshots of different curve functions and all of them made with [Desmos](https://www.desmos.com/calculator). I can't stress enough how valuable Desmos can be when coming up with new functions and visually seeing their output instantly.

### Linear Curve

The simplest curve in the toolbag that allows you to control its slope and offset.

![Linear Curve](/images/utility-ai/LinearCurve.png "Linear Curve")

### Exponential Curve

This curve can raise the input to a configurable exponent, the higher the exponent the higher its rate of change is and with a slight tweak to the function we can generate its inverse. The curve function also has a parameter to offset the curve.

![Exponential Curve](/images/utility-ai/ExponentialCurve.png "Exponential Curve")

![Inverse Exponential Curve](/images/utility-ai/InverseExponentialCurve.png "Inverse Exponential Curve")

### Logistic Curve

I first encountered this curve in [Dave Mark's GDC presentation](https://media.gdcvault.com/gdc10/slides/MarkDill_ImprovingAIUtilityTheory.pdf) back in 2010 and it is very easy to customize. It provides a parameter to control steepness and the midpoint of the S-curve.

![Logistic Curve](/images/utility-ai/LogisticCurve.png "Logistic Curve")

### Logit Curve

These are logistic curves turned on its side as Dave Mark remarked in his 2010 talk. Like the logistic curve it provides a steepness parameter and can be extended to offset the midpoint vertically.

![Logit Curve A](/images/utility-ai/LogitCurveA.png "Logit Curve A")

![Logit Curve B](/images/utility-ai/LogitCurveB.png "Logit Curve B")

### Sigmoid Curve

This sigmoid function comes from a [blog post](https://dhemery.github.io/DHE-Modules/technical/sigmoid/) made years ago. I like this function a lot for its simplicity and lack of exponents so floating point inaccuracies are going to be minimal. It can also provide normalized J-curve shapes for inputs in [0, 1] range but can also provide S-curve outputs for inputs in [-1, 1] range with the same function. The function is also very simple to evaluate.

![Sigmoid](/images/utility-ai/Sigmoid.png "Sigmoid")

As previously said, these are only a handful of curves out there. I didn't show curves like cosine or sine waves where you could change parameters like amplitude, phase, and wavelength to configure them. The ones covered will be useful for most scenarios you will encounter though.

Having these curve types that are data driven rather than hand crafting them feels like it is a better approach than providing raw configurable Catmull-Rom type curves like I've discussed in a previous post. 

### Consistent Utility Scores

It's rather unfortunate but if you read a lot of the available information online on scoring based AI systems and you'll seldom find any mention of keeping your scoring scales consistent so that comparing them makes any sense. All too often the posts seem to mix in different scoring scales for different actions with their examples that you don't notice until you try to implement your own utility computations. I've found that a good starting point is to normalize your scores into a [0, 1] range. This allows keeping whatever range you want for the individual scorers and as long as they are normalized before averaging or comparing then you maintain a consistent scale while allowing some flexibility for scorers. That said it doesn't need to be a [0, 1] range as long as the range across the system is consistent.

### Practical Usage Of Scores

A consistent range can also open up additional possibilities to influence your scoring system. For instance, after normalizing you could have separate biasing weights to make certain actions more or less likely to happen. In my game, I have a concept of personality traits to create a bit more variation while still using the same configured set of scorers. With this system I could have some enemies be more aggressive and I could express this trait by biasing the attack action to be more likely or I could bias the positioning scorer to not care about being surrounded by enemies after it moves to a position it can use to attack. This situation can also be reversed by implementing a trait for "fear" in which the enemy is less likely to stray from its team or be quick to run away from combat. I'd say the possibilities are endless so long as you can express the scorers with mathemathical functions.

Since you have to compute the value for each score used in the utility system, you can create a sorted list of them starting with the best possible action going down to the worse possible action. Having this data opens up some additional possibilities from a gameplay perspective.

Say you don't want to always use the best action, think of difficulty settings in games, with this sorted list you can pick a random action from the top N best scored actions or you could have a more sophisticated system that keeps a history of what actions were taken before and you want the AI to use an action it hasn't used in a while. For an easy difficulty setting you could always discard the first few best scored actions so it consistently uses sub-optimal actions. Another use case is if you have a sort of "intelligence" trait, going back to the personality system, then that inlligence factor could be used to introduce noise into the scoring system. The smarter the enemy is the lower the noise amount will be.

Another interesting gameplay use case is implementing the status effect **Confuse**. This is often implemented by having the confused NPC randomly pick an action. If you have a sorted list of possible actions the NPC can perform then you could pick randomly from the N worse rated actions or pick from a mix of high and low scored actions just so it isn't always a negative.

## Inertia

Inertia is a very important concept to talk about that I also rarely see mentioned or it just tends to be implied. It's important to know when to have your AI agent decide what it will do. If it is updating it every frame then you run the risk of running into oscillating issues where in frame N in picks action A, on frame N+1 it picks action B, and in the next frame goes back to picking action A. In the case of utility AI this can happen when several actions are scored similarly. An example of this could be an when an AI in an FPS decides it is in a bad spot as it starts to get shot at by the player. The enemy AI is scoring an "attack the player" and " run away" actions and the results are both equal. It picks "run away", then soon after decides it needs to "attack the player", and finally decides it needs to "run away" again. This behavior looks bad to the player and probably also feels equally bad to play. There are plenty of solutions to deal with this but just know that is something to be aware of when building AI systems as this isn't unique to utility AI. Some systems even have inertia built into its decision making process. Any planning-based system like [GOAP](https://medium.com/@vedantchaudhari/goal-oriented-action-planning-34035ed40d0b) or [HTN](https://en.wikipedia.org/wiki/Hierarchical_task_network) will create a plan for the agent and only update when the plan is done or something in the world invalidated the plan so the agent needs to make a new one.

## Closing Thoughts

After working with this system for a bit, I really do like it a lot. It's very easy to add new actions and scorers. Unlike behavior trees, the amount of code required to add new functionality is a lot less. That said it can be tricky to have a general sense of what the utility model will do once you get it to a certain size. At the end of the day you are balancing a bunch of different weights and response curves so I think investing in some tools to visualize the scoring results could be valuable. Balancing the model to get the AI agents to use many of their actions in the middle of gameplay can be tough if you are relying only on the raw results from evaluating the actions. This is why I think adding things like the personality traits to bias and transform the model somewhat can be useful. That said I will probably spend more time expanding this system and see how far I get but before that I will likely finish implementing and evaluating the other AI system I wanted to properly investigate. That other system being hierarchical task networks or HTN. If I get far enough with it then I will likely follow up with another AI post in the future.