---
title: "Game UI"
date: 2024-08-02T16:12:16-07:00
draft: false
toc: false
images:
tags: 
  - rendering
  - UI
---

* [UI Requirements](#ui-requirements)
* [Logic Updates](#logic-updates)
* [Rendering](#rendering)
* [Stitching Triangle Strips](#stitching-triangle-strips)
* [9-Slicing Sprite](#9-slicing-sprite)
* [Basic Rich Text Support](#basic-rich-text-support)
* [Localization](#localization)
* [Frosted Glass Look](#frosted-glass-look)
* [Creating GUI Content](#creating-gUI-content)
* [Limitations](#limitations)
* [Examples](#examples)

The Tempest engine has support for two UI systems. It makes heavy use of [Dear ImGui](https://github.com/ocornut/imgui) for the editor UI and the in-game debug console but there is also a custom UI system made to handle the game's UI needs. As great as Dear ImGui is, you will rarely see it used in released titles, at least not as the main system behind a game's UI. There are multiple reasons for it but in my opinion one of the big reasons is that it is not trivial to re-skin the UI elements. The immediate mode design of the API can also be a big factor for some. Eventually a custom game UI system was designed to meet the needs of the game being made with Tempest and most of it has been developed by this point. This post will cover many aspects of the game UI system developed so far.

## UI Requirements
The most important idea driving the requirements for this game UI system is that it is perfectly fine that it does not have many features but the features it does have work great. Having a general idea of what the UI will be like for the game helps tremendously to narrow down the requirements for the system. 

So what are the specific UI requirements needed for the game? 

Well performance means a great deal to me so being able to render the game's UI in under a millisecond will be a huge win. The GPU will already be taxed enough with all the supported rendering features in Tempest so keeping this as low as possible will be very important to overall performance. Ideally having only one draw call to render all the UI visible on the screen is the target to hit. This would mean all the UI uses the same "material" to reduce draw call count and should be achievable with a moderately complex shader.

Since the game will support gamepad and mouse and keyboard, the UI needs to be able to handle both for navigation and interaction. As a gamer, I find it frustrating when you play a game that may push you to use one form of hardware input but their UI doesn't support it so you have to switch to another form of hardware input like mouse and keyboard in order to interact with it. This is a terrible user experience and I would prefer not contribute to that messy experience. 

Many of the common UI widget types need to be supported. In Tempest this means support for labels, buttons, sliders, toggles, dropdowns, and scrolling lists. With that set of widgets supported you can pretty much make anything you will need and with texturing support too you will be able to make it look however you want.

Loading up any UI panel needs to be instantaneous while running the game. There have been many games over the years where opening up a menu can trigger a half second stutter as the menu resources are loaded from disk, shaders compiled, etc. Menu navigation will be common enough that I want that to feel responsive as you open and close different menus.

UI assets need to support hot-reloading in the editor. This is crucial to develop the layout of new panels quickly.

Support for localizing text in the UI. It is really nice to have a system like this in place from the start as otherwise, it can be pretty laborious to add localization support to a system that was implemented without it.

Finally, any custom scripting logic needs should be easily solved with the scripting system in Tempest.

## Logic Updates
Many UI systems have a way to establish some sort of hierarchy with an object that contains any number of child widgets. In Unity that is the canvas object and in Tempest that is the GuiPanel. This object contains a list of UI widgets so it is easy to update all of them every frame. Like many UI systems, you can setup the states for all the widgets. So you can control whether or not a widget is visible on the screen, or if you want it to be hidden but still have its logic updated, or skipped entirely. With the GuiPanel object, you can set its state to be hidden and then all its children widgets will be skipped from being updated so you can have multiple panel objects loaded into memory and only pay the performance cost for the active ones.

The engine has a built in component system that will iterate over all GuiPanels loaded in the scene. It can use the job system to update widget states. This logic update is meant for updating widget types that can respond to user inputs such as a button press or mouse click. On PC we need to also keep track of the mouse position so we can highlight any button underneath the mouse position. At this point in time there is no spatial data structure to optimize searching for such a widget since there aren't enough of those to make it worth it and multithreading is used anyway so the speed is incredibly fast.

One thing I find to be important these days on PC is to be able to give the user the choice of what gamepad icons to use in the UI. It's fairly easy to add support for the icons specific to a platform and let the user choose which ones to use for their playthrough. I hope this becomes the norm in the future and I do see more games doing this so that's encouraging. A nice to have feature on PC is for the UI to dynamically update what gamepad icons are used based on the most recent input received. I kinda like having the game switch to using mouse and keyboard icons if the most recent input came from the mouse but that might just be a preference of mine.

An important design of the UI system is to use event driven UI events. Widgets that can respond to user inputs have a way to assign lambda functions to the events the widget supports. For instance, a button widget will have an event for when the button is pressed. During its logic update, it will detect if this happened on the current frame and if so it will call the lambda for that event if it is set. This gives a very easy way to script the behavior of these widgets and a performant one too.

## Rendering
As previously stated, performance is the main priority in the design of this system. One of the core principles behind that being to use one draw call to render all the visible elements or at least as few as possible. For most game UIs, this means solving the problem of rendering many quads in a performant way.

So how can we render the UI with one draw call?

Well the renderer manages one large vertex buffer to store the vertices required to render all the visible UI elements. Using the Tempest job system the CPU data is updated every frame before sending it to the GPU. Right before updating the GUI rendering data, all the visible UI elements are sorted based on their Z-order depth value. This essentially creates layers where low values are rendered behind larger values. We combine that with all widget types using the same shader and then we have the perfect recipe to draw all the elements in one draw call. The shader supports two UI atlases and a couple of font textures to handle all the texturing requirements present in the game. This does mean the shader is a bit more complex than it would be otherwise and this does mean using branching in shaders to handle all the supported widgets. I know some will get scared at the sight of branches in a fragment shader but in the UI shader they are done based on constant buffer values so they are very fast.

You may have noticed I mentioned nothing about an index buffer to go along with the UI vertices. I must admit I haven't done any recent performance tests but my gut feeling is using indexed drawing here is not going to be that benefitial and may be worse from a memory standpoint since we'd have to manage the index buffer on CPU and GPU. It would also complicate the logic for preparing the vertices. Originally this system specified all the triangles individually, so rendering a quad required 6 vertices instead of 4 like you would with an indexed drawing method. I don't know if this is a good idea but I have since moved the UI system to use triangle strips instead of individual triangles. This brings the vertex requirement for one quad back down to 4 but it does mean padding the vertex buffer with [degenerate triangles](https://en.wikipedia.org/wiki/Glossary_of_computer_graphics#degenerate_triangles) in order to be able to draw the whole UI in one draw call. 

### Stitching Triangle Strips
This topic isn't mentioned a lot but it is very useful to know about it. If you aren't aware of what a [triangle strip](https://en.wikipedia.org/wiki/Triangle_strip) is then very quickly it is a memory efficient way of packing vertex data where the driver has a predefined way of using them. For example, if you have four verts [A, B, C, D] defining a quad and you render it using a triangle strip primitive topology then the first triangle is formed from [A, B, C] and the driver will use the last two vertices combined with the next vertex in the buffer to make the next triangle. So the second triangle in that quad is formed by [B, C, D] and every subsequent triangle only needs to have one vertex specified as it will reuse the previous two vertices.

This is a great method to use when all your vertices are next to each other but how do you use it to render triangles that may not be a part of any given triangle strip? Lets say strip S is defined by the vertex buffer [A, B, C, D] and strip T is defined by [E, F, G, H]. How do we render them both in one draw call without any visual artifacts? Well as stated before this is where degenerate triangles come in to save the day. We create a strip U defined as [A, B, C, D, D, E, E, F, G, H]. That is to say we have a strip that contains both strip S and T and to bridge the gap between the two we append two vertices after strip S. One vertex is a duplicate of the last vertex of strip S and the second one is the first vertex of strip T. If you decide to resolve how the driver will interpret this buffer with a triangle strip topology you end up with something similar to what is below.

* A B C
* C B D
* C D D - deg
* D D E - deg
* D E E - deg
* E E F - deg
* E F G
* G F H

The degenerate triangles above do not get rasterized as they have no area so they will not be visible in the rendered image. The one downside of course is that it will inflate the vertex buffer, depending on how many independent triangle strips you may need to stitch together.

### 9-Slicing Sprite
Since texturing widgets is possible we also need to be able to handle texturing large sprites without warping or stretching the underlying texture. The usual method to solve that problem is using the [9-slicing method](https://docs.unity3d.com/Manual/9SliceSprites.html) where you customize how the texturing is applied to the sprite.

![Slicing Example](/images/game-ui/9SliceExample.png "Slicing Example")

In Tempest, you specify how you want it sliced using the text format to create the UI panels but more on that later. From the example image above, the regions marked as A, C, G, and I make up the corner of the sprite and those do not do anything special really. The other regions however, can be bigger than what the texture being applied to them is and in those cases tiling is used in order to avoid distorting the image. 

As far as I know, there are two ways of implementing this. One way is to have a complex shader that figures out what region the fragment is in and textures accordingly. The other is to tessellate the sprite in the CPU taking into consideration how the slicing regions are laid out. Doing it this way means the shader does not need to change and the logic is kept in the CPU which could open opportunities for optimization like if you cache that mesh for example. Tempest renderer does the latter since we are already building the vertex buffer every frame it makes sense to add this additional logic for sprites that need to be sliced and everything else works the same.

This method does clash a bit with triangle strip rendering though. The regions that will tile the texture sampling need to have duplicate vertices so that we can specify the correct texture coordinates for them. 

### Basic Rich Text Support
It is very common these days for labels to contain a substring using a different color than the rest of the label, or an image, an animation, etc. This is typically solved by having support for rich text format. All this means is embedding special tags inside of your label that will determine some sort of transformation to apply to a subset of characters. The Tempest renderer has very basic support for this. In fact, the only thing supported at this moment is changing the color of parts of a label as that is the only thing that has been needed so far. That said, the architecture is there to add other kinds of tags of similar complexity to changing colors. 

### Localization
While this is out of scope for this post, localizing UI text is supported in Tempest. There's a whole pipeline around creating and using these special kinds of strings that will be covered in a later post.  

### Frosted Glass Look
A lot of games these days like to have that frosted glass look on their UI. It sort of simplifies the work in designing your but I also do kinda like the look so that was implemented in this UI system. The renderer already generates a blurred frame containing pretty much the entire lit scene so it can use this effect for "free". There's a example screenshot below of the prototyped graphics settings in the game that showcases the frosted glass look.

![Settings Menu](/images/game-ui/SettingsMenu.png "Settings Menu")

This effect is achieved by sampling the blurred texture and doing alpha blending in the shader between the widget's color and the blurred sampled color.

## Creating GUI Content
After talking about all these features we still haven't mentioned how the UI panels are created. Sadly there is no fancy editor tool to create the UI panels instead opting to use a text based solution. Since GUIs have a hierarchy embedded into them I decided to use json as the text format to allow me to create the UI. The engine already supported reading json files so nothing new needed to be done there. All the properties for a widget is specified in the json file and given its hierarchical structure it can also easily let me specify relative positions for child widgets and the importer takes care of computing things like absolute positions. The way json keys work can also be used to create unique paths to any widget defined on that file or panel. This is used in the scripting system to request references to specific widgets in a panel and those references can be cached for later use. I think I can safely say a lot of the built in features that make up a json file are being taken advantage of to create the game UI system to great effect.

The json files specific to the UI system have a special kind of file extension so the asset manager can handle cooking, loading, and hot reloading of those assets. Since there is no editor gui to create these panels, it is very important to easily and quickly iterate on these UI panels so hot reloading them has been the best feature with this system by far.

One final note on authoring these panels is that they are made with a 16:9 aspect ratio as the target aspect ratio. Then the UI is scaled to handle other aspect ratios at runtime based on the rendering resolution. There will likely come a time where a more complex method might be needed but for now this works well.

## Limitations
To no surprise there are some limitations at the moment that may or may not need to be addressed before the game is finished and I'll point some of them out here.

There is no support for clipping regions. At the moment things only get clipped based on the alpha value of the texture they sample or the vertex alpha. It could be benefitial to be able to discard fragments in specific regions of the UI but at the moment the game's UI design does not need it.

The UI panels are static, meaning there are no spatial animations. Supporting this could be a large endeavour as this would also imply providing ways to do animations from the content authoring tools and scripting. Most things wouldn't really need this but there could be a handful of combat related UI elements that could benefit from some animations.

No visual editing tool to create new UI panels, it is only text based. 

## Examples
I've added a few more examples of different panels made with this system. Just note that these are all prototypes and in no way final but thought I should give a few more examples of what the game UI can produce.

![Party Menu](/images/game-ui/PartyMenu.png "Party Menu")

Thought it would be interesting to post a wireframe of what the mesh looks like when stiching together a bunch of strips into one. As you can tell, it is a little messy when you see those stray line segments connecting other strips all throughout the mesh.

![Party Menu](/images/game-ui/PartyMenuWireframe.png "Party Menu Wireframe")

Here is an example of what the combat UI can look like at this point.

![Party Menu](/images/tempest-engine/editor.png "Combat UI")