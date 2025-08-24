---
title: "Fluid Fog"
date: 2025-08-22T20:21:31-07:00
draft: false
toc: false
images:
tags: 
  - rendering
---

![FluidSample](/images/fluid-fog/FluidSample.png "Sample")

* [Motivation](#motivation)
* [Technique Overview](#technique-overview)
* [Fluid Simulation Architecture](#fluid-simulation-architecture)
* [FluidEmitterComponent](#fluidemittercomponent)
* [FluidContainerComponent](#fluidcontainercomponent)
* [Fluid Rendering](#fluid-rendering)


Before getting into the details below is an example of the fluid fog running in one of the levels of my game. I recommend putting the video in fullscreen to see the details. You'll see how the fog is reacting to the characters moving in the fog.
<video controls width="640" height="360">
  <source src="/images/fluid-fog/fog_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Motivation
Ground fog has been a rendering feature used in games many times before by this point however, it typically is an effect that does not react to players or the world around it. Instead the fog relies on animated noise to give it structure and in some more advanced cases fake some level of dynamism. I wanted to iterate on this feature and try to bring something new yet familiar.

Several years ago i had prototyped an idea after implementing volumetric fog. I had thought that generating a 3D density texture from a fluid simulation could then be piped into the volumetric fog framework I had just finished and it would render that density field as if it was fog. I already had most things working since I had support for static fog volumes and this would be an extension of sorts to them. In the end it worked as expected and you could see the density field updating in real time as it got advected by the velocity field. Performance issues aside the prototype worked and the idea has stuck around in my head ever since.

Fast forward to today and I now have an opportunity to put this idea to the test. The game I'm building this engine for has a camera up in the sky looking down to the world that will only rotate around the global up axis. This gave me the idea to try 2D fluid simulation as a cheaper alternative to the expensive 3D fluid simulation I had tried in the past. Since then techniques like this [one](https://staticctf.ubisoft.com/J3yJr34U2pZ2Ieem48Dwy9uqj5PNUQTn/6c99QTT4AoxazNKyJ7TJGC/dbeb06e6d52684a6b45bef3ffe487e13/fast_eulerian_fluid_simulation_in_games_using_poisson_filters.pdf) had been published proving you could have 2D fluid sims look convincing in 3D worlds.

My goal was to make the fog something more than a purely visual element of the world. I wanted it to react to events ocurring around it as well. Things like characters moving through it or explosions affecting the velocity field. All this to hopefully surprise players by creating a moment or two that maybe they haven't seen before.

## Technique Overview
The fluid simulation is done on the GPU using a grid based technique. GPU textures map cleanly to this grid structure as that is pretty much what they are. This also facilitates dividing the simulation space into square cells. At the center of each cell the fluid state is stored, which consists of the density and velocity. After the simulation is done you end up with the final pair of 2D textures storing the density and velocity field of the fluid.

To visualize the fluid fog the engine raymarches against a cube mesh that represents the volume of the fog. This is done as a material instead of a post process effect largely for performance reasons. This means raymarching only ever happens on fragments that get rasterized on the screen. The raymarching implementation is your usual volumetric solution with the addition of faking some volume from using the 2D density texture. It does this by using a custom falloff function with a few parameters that can be tuned by a developer to control the extent of the extrusions performed at each step along the ray. This creates the illusion of volume but only to an extent before it starts to break down. Given the type of camera used in the game however, this does not become a problem.

## Fluid Simulation Architecture
Like many other systems in Tempest a component-based architecture is used to setup the fluid simulation system. This makes it fairly easy and intuitive to configure. There are two main components that make up the fluid sim system, the **FluidEmitterComponent** and **FluidContainerComponent**.

### FluidEmitterComponent
This represents the source of the fluid. As the name implies this component is responsible for emitting fluid inside the container it belongs to. It uses the engine's transform component to easily allow setting its position within the fluid container. With this component you are also able to specify things like the shape of the emitter, through a texture, size, velocity, density using a texture, external forces like gravity and many other properties. By default these emitters add fluid to the container but they can also be configured to be subtractive or in other words they remove density. This is the feature that allows characters to affect the fluid fog's density and velocity fields.

### FluidContainerComponent
As you may have guessed, this component contains all fluid emitters asigned to it as there can be more than one fluid container component active at once. Containers keep a list of all emitters they are responsible for. They also allow configuring fluid properties like pressure or turbulence, boundary conditions, etc. When the container's shape is configured to be a plane then you can also configure it to be camera facing like a typical billboard with the added extension of optionally locking any of the 3 rotation axis. This opens the posibility of using the fluid system for VFX other than the fluid fog this post talks about.

### Fluid Rendering
In order to support multiple fluid containers, Tempest uses atlas textures to store all the density fields and velocity fields from each fluid container. The resolution of the density and velocity texture are allowed to be different. In my case the density atlas texture has a higher resolution than the velocity atlas texture given that bilinearly sampling the lower resolution velocity texture offered better performance while not degrading the overall quality of the fluid simulation. Currently the density atlas is 512x512 while the velocity atlas is 256x256. Since the camera is far enough away I can get away with using lower texture resolution and the bonus of course is better memory usage and better performance.

During each frame the renderer makes sure to update the individual regions of the atlas textures that correlates to each fluid container active in the scene. Then for each fluid container the renderer splats each emitter the container has before proceeding to doing all the required simulation steps using compute shaders. Using compute also opens the possibility of using async compute in the future which can be a pretty powerful performance optimization. It's important to note that a fluid container's overall world space size will dictate how much of the fluid texture atlases it will consume. So the bigger the container is the more important it is to the renderer. 

I hadn't mentioned it yet but being able to art direct the fluid simulation is very important in game development. Giving structure to the ground fog is something that I need to be able to specify for any of the levels I decide to use this feature in. Take for example the level from the video at the top of the post. It's a level designed to mimic a marsh environment and as such it has a flat body of water in it. The water plane uses planar reflections through the use of a secondary camera to generate a reflection texture. It is very important to me to maintain that feature while also adding the fluid fog on top of the body of water. I needed to strike a balance between those two rendering features. This is where using a texture to determine the overall shape of the fluid's density comes in very handy. I wanted a type of ground fog that was pretty patchy so you could still easily see the reflections on the water. I was able to achieve that by using a caustic texture for the fluids density. The texture has a voronoi structure which made it perfect for the patchy ground fog I was looking for. Below is a screenshot of the texture I'm using in all the videos in this post.

![FluidCaustic](/images/fluid-fog/FX_Caustics_A.png "Caustics")

Here's an example of what the fluid density and velocity texture atlases can look like after simulation.

![FluidDensity](/images/fluid-fog/FluidDensity.png "Density")
![FluidVelocity](/images/fluid-fog/FluidVelocity.png "Velocity")

You can also make out the circles where the characters are standing in the fog considering they are configured to remove any fog density and velocity in a circular shape.

In regards to the raymarched fog, not much to add here other than it is using some basic lighting. It doesn't respond to local light sources at the moment or even shadows just to keep the overall effect performant. It is however, something I would like to revisit in the future once the game is farther along. I am overdue to update the Deferred+ feature in the engine so it would make sense to use the cluster structure storing light data for lighting during raymarching.

Finally, performance has been very good thus far. Simulating the fluid fog with about 8 characters affecting it in combat is costing around 1.1 milliseconds on the GPU. This is great, especially considering that the simulation steps are executed multiple times per frame to ensure proper convergence of the simulation. I am playing around with the idea to tie the number of iterations with the quality settings the user has which could improve performance at the cost of visual fidelity.

To end things this here is a quick clip of what the marsh environment looked like before adding the fluid fog implementation. Overall I was pretty happy with it but still felt like it was missing something and I think that was the fluid fog I have since added.
<video controls width="640" height="360">
  <source src="/images/fluid-fog/marsh.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>