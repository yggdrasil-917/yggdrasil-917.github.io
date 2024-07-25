---
title: "Tempest Engine Loop"
date: 2024-07-24T20:30:08-07:00
draft: false
toc: false
images:
tags: 
  - engine
---

The previous [post](https://yggdrasil-917.github.io/posts/tempest-gpu-frame/tempest-gpu-frame/) did an in-depth look into how a Tempest frame is put together on the GPU. In this post I would like to do the same but looking at things from the CPU side. Essentially we will dive into what the engine loop does to create a brand new frame.

This time around we will look at a new scene in the middle of a combat scenario playing out as more things will happen on the CPU than the rendering test scene we looked at previously.

![Combat Scene](/images/tempest-engine-loop/Combat.png "Combat Scene")

Tempest has support for the [Tracy](https://github.com/wolfpld/tracy) profiler and you can see it in action below. The screenshot is a zoomed out view of the CPU timeline where you can see multiple frames being shown. We will zoom in and look at an individual frame while going over different systems that run in the engine loop.

![Tracy Timeline](/images/tempest-engine-loop/cpuTimeline.png "Timeline")

And here is a screenshot showing only one frame. I suggest opening the image in a new tab so you can view the different performance markers spread throughout the frame. You may notice in the Tracy captures that there is a job system being used. The laptop running when this trace was captured has support for 12 threads and as such you can see 11 threads allocated for the job system plus the main thread to divide the work.

![Tracy Timeline](/images/tempest-engine-loop/cpuTimelineZoomedIn.png "Frame Timeline")

I should note the editor allocates a few more threads on top of what the job system needs. These threads are used for things like handling network packets to support inter-process communication, file watching, etc. For the most part those threads are not doing anything and just waiting for work to come in which is why they didn't show up in the Tracy capture.

## Running The Editor
Here is the totality of the main function used by the editor but the player uses pretty much the same function.

```cpp
// Tempest editor's main function
int32 main(int32 argc, char** argv)
{
    const CommandLine cmdLine(argc, argv);
    TempestEditor editor;
    if (!editor.Startup(cmdLine))
        return 1;

    editor.RunEngineLoop(); // where the work happens

    editor.Shutdown();

    return 0;
}
```

We start by extracting any information from the command line before we pass that to the initialization function where it will handle any setup the various systems need. This involves things like reading configuration files, creating the application's window, crash reporter hookup, and many other things. The editor has several things more to initialize (i.e. file watcher) that the player does not have to worry about and this is where it happens. If something failed during initialization then that will likely get logged and then exits. Otherwise, we fall into the engine loop where most things happen. Then we have the shutdown phase which as the name suggests is where all the cleanup happens before the application exits.

## Engine Loop Structure
To nobody's surprise every engine has a game loop however, no two are exactly the same. Despite the differences they tend to share the same ideas and have similar structure. Below is pseudocode depicting what goes on in the Tempest engine loop and you can also use the Tracy screenshot above to follow along.

```cpp
// Simplified Tempest loop structure
while (keepRunning)
{
    PumpWindowEvents();
    UpdateInput();

    SimulateFixedTimeStep();
    Simulate(deltaTime);

    Render();

    WaitForTargetFramerate();
}
```

### Start Of Frame
The very first thing done in the loop is to detect if there is any scene transition queued up and if so then the engine will load the new scene before doing anything else. In the editor, there is a concept of editing or playing the currently loaded scene. This is where the editor will change state and go from edit to play or vice-versa. Right after this is where the new frame's delta time is computed before sharing with all the other systems in the engine. Something to be aware of when computing the delta time is that the value can be completely wrong when you step through the application with a debugger. It's very easy to take a while stepping through the debugger and by the time the delta time is computed it ends up being much higher than your target framerate. A trick to minimize the problem is to clamp the delta time to your target framerate when the debugger is attached thereby avoiding spikes in the delta time values.

At this point OS specific events are processed. In Windows terminology, this is where the events and messages are pumped and determine the keyboard and mouse state, window state, etc. Once that is done the engine can update the engine specific input data structures. Currently the last thing it does before world simulation is to update some engine subsystems like the online subsystem. On PC this is responsible for handling the Steam API so things like achievements get updated on the backend here.

### World Simulation
At this part of the loop the engine will run all the systems that update world's state. From the pseudocode above, this is where **SimulateFixedTimeStep** runs. Its function is to any system that wants to be updated once per frame using a fixed delta time. This is where the physics engine gets updated. In Tempest, Physx is supported although at the time of this writing it is disabled since the game being developed with Tempest does not require it. In addition to the physics engine, any scriptable entity that implements their fixed time step simulation function will be updated here.

The next step of the world simulation is the **Simulate** function. This is probably what most people are most familiar with. The engine will iterate through all the systems that want to be updated at this point in the frame. For instance, the engine has a system that will iterate through all entities that have a transform component to update them before any other subsystem reads them. As you can tell these component systems have a update order since some systems will refer to values that other systems may modify. This is handled in the engine by assigning each component system to a specific update phase determined by a enum value. Then these components get sorted every frame based on that enunm value so we have a easy way of specifying the update order. 

Another example of a system that runs here is the particle emitter update. It will go through all particle emitters in the scene and simulate them based on their configuration. Again, it will do this update using the job system in order to be performant considering some emitters can have hundreds of particles. Job system support is available for any and all component systems in the engine. For some it does not make much sense to use the job system since there may not be that many entities using those components and could actually be detrimental to the performance as there is a base cost to use the job system. You can see the multithreading in effect in the Tracy screenshot, specifically under the Update Game Data (High Priority) marker where it is updating all the transform components using every thread available.

### Rendering
After simulating the world comes the rendering phase, in other words, the engine can start preparing all the data to submit to the GPU for rendering. The most important task that happens here is going through all the renderer components and assigning them to the correct set of render queues. Due to the flexibility of the renderer some things may opt to not render shadows or not render motion vectors. As such the engine keeps track of separate render queues for different features. This also lets the engine sort these render queues differently in ways that can potentially offer specific benefits for whatever render pass may use the queue. Since there can be a lot of models needing to be assigned to their queues each frame, this system emulates thread local arrays so each thread in the job system can have exclusive access to a region of memory and make it easy to add models to the render queue array. It is later in the frame that all these thread local arrays get merged into one array so it can easily be sorted and later used for rendering. Various component systems opt to not use thread synchronization primitives like mutexes for performance reasons but at the cost of more memory. With some profiling this architecture was proven to be faster than locking the critical section as it would be a resource with a lot of contention in a frame.

It wasn't mentioned but this is also where frustum culling would happen before assigning entities to their render queues. This is also done using the job system so multithreading is supported with the frustum culling system. However, it is disabled by default since the game will have most things visible on the screen at all times so performing frustum culling was a net negative for performance. That said it can be easily toggled on using the built in game console for submitting debug commands.

Now the engine can build its frame graph that can be compiled and executed right after. The execution can be done with multithreading which is what the renderer does. It uses a couple of threads from the job system to record all the command lists before submitting them to the GPU. The render passes that compose that frame graph are fairly easy to add to the engine. The complexity is determined by whether it is a render pass that will be rendering models or just a fullscreen pass or compute pass to be used for things like post process. The main complexity comes with creating pipeline state objects which can be a bit of a pain when the render pass supports all kinds of draw call state setup like depth testing, depth writing, and many more. At the moment the renderer uses an intermediate struct to represent the render state features a draw call will need and hashes them to create an integer value that can be looked up in a PSO map. If the map does not contain a PSO with that key then it knows it has to create a new one and then cache that afterwards. Here we do use a mutex to lock the modification of the map since this part is multithreaded.

After all the scene rendering is done, there is still one rendering task remaining for the editor. That task is rendering the editor's UI which in Tempest means rendering with Dear ImGui. Thankfully ImGui makes this really easy to do once you have the rendering backend integrated. In the case of Tempest this needed to be custom made as none of the provided ones were enough but offered good guidance on what needed to be done to make it work.

### End Of Frame
The very last thing to do is to present the frame. You may or may not have noticed that nothing was mentioned about the CPU working on the current frame while the renderer works on the previous frame's data. This is what most modern engines do but Tempest does not. It uses a more old school approach where the both the CPU and GPU are working on the same frame but with some modern ideas implemented as well. There may not be overlapping work done between adjacent frames but there are modern concepts sprinkled throughout the engine's architecture to balance things out. The job system is one such concepth. All of this helps to create a very fast frame on the CPU as you can see on the Tracy screenshot where the frame is done in about 5ms.

After that, if vSync is disabled then the engine will wait for a target framerate assuming the frame has not blown past that. It achieves that with a simple while loop but can decide to sleep the thread instead if there is more than 5ms before achieving the target framerate. Sleeping can help keep the CPU from heating up too much and also allows other external processes access to the CPU core the main thread will be on. A word of caution though, if you rely on sleeping a thread on Windows then know that the default resolution of periodic timers need to be configured in case you sleep a thread with a small value. The lowest supported resolution is 1ms which is what Tempest will set it to when the application starts. If you don't do this the thread will be awoken at a much later time than what you may specify.