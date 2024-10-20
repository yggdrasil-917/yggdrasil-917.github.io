---
title: "Dynamic Resolution"
date: 2024-10-18T15:43:50-07:00
draft: false
toc: false
images:
tags: 
  - rendering
---

![DynRes](/images/dynamic-resolution/DynRes.png "DynRes")

* [Timing Your Application](#timing-your-application)
* [Configuring DRS](#configuring-drs)
* [How It Works](#how-it-works)
* [Temporal Issues](#temporal-issues)
* [Shader Considerations](#shader-considerations)
* [Debugging](#debugging)
* [Example](#example)

Dynamic resolution scaling or DRS is a technique where the rendering resolution is changed dynamically at runtime in order to improve GPU performance. This can help provide a smoother overall framerate for an application at the cost of lowering the image quality but still retaining the graphics settings specified by the player. However, DRS only makes sense when the application is bottlenecked by the GPU performance and, if properly implemented, will only do work when that is the case. That said, many modern games tend to be GPU bound so it is a good idea to implement this feature but also provide ways to disable it in the engine for debugging purposes.

## Timing Your Application

So how does this work in practice? First and foremost the engine must be able to track both the CPU and the GPU times it takes to render a frame. For CPU times, any high resolution clock will do and these days you could opt to use the c++ chrono library for that so you don't have to worry about platform specific implementations. This is what Tempest uses to keep track of time on the CPU. For CPU and GPU timing comparisons to make sense in this context the engine should also keep track of the amount of time the CPU waits for the GPU to present a frame. You can use that time to subtract from the overall CPU time to render a frame before you compare with the GPU timings. This way you remove any idle time spent waiting on the GPU.

Timing the GPU involves a bit more work though, in part because the GPU will likely be working on a different frame than what the CPU just processed. In the case of being GPU bound then the CPU will finish processing it's new frame before the GPU finishes its frame so if we have the CPU request the GPU timer's value it will likely have to inject a synchronization point and wait until the value becomes available. Doing it this way will also inflate the CPU time as it stalls until it gets the value from the GPU. What Tempest does to solve this is buffer every GPU timer query in an array whose length is the max number of GPU frames in flight plus one to account for the one the driver may be working on. Buffering will introduce some latency in getting the GPU time however, it removes any possibility to stall the CPU while it waits for the current GPU time value. To make it easy to implement this in Tempest, every rendering pass derives from a base class that has several features most or all passes will need and one of those things are the GPU timer query objects alongside a simple API to manage them.

## Configuring DRS

Now that we can properly time things both on the CPU and the GPU, we can start making use of them in DRS but first lets go over initializing a DRS system. Tempest exposes certian parameters to configure the behavior of DRS. 

* **Target Framerate** - the ideal framerate we want the application to have, in our case 60 FPS or 16ms.
* **Update Interval** - how often we want DRS to adjust the scaling factor. It makes sense to not adjust this on every frame as it will be obvious to the user and could become distracting.
* **Minimum Height Resolution** - configures the lower bound of the height rendering resolution allowed to use when computing the scaling factor. Typically this is set to 50% of the native height rendering resolution so a 1440p native resolution would have this set to 720p.
* **GPU Headroom Before Increasing** - this is a threshold used when deciding to increase the rendering resolution if the GPU is too underutilized at the current scale.
* **Decrease Rate Of Change** - a constant that controls how aggressive DRS will lower the scaling factor.
* **Increase Rate Of Change** - a constant that controls how aggressive DRS will increase the scaling factor.

With all of these parameters you end up with a pretty flexible system and there are still a handful of other internal parameters that help drive some of the features in this system which will be covered later. Now we can look at the decision making flow that DRS in Tempest takes every frame. The diagram below shows what it looks like.

![Flow](/images/dynamic-resolution/Flow.png "Decision Flow")

## How It Works

Note that all of the following work is only performed if DRS is enabled. At the start of a new frame the renderer will add up all the individual GPU timers for each rendering pass submitted to figure out the overall GPU time similar to what the sample code below does. 

```cpp
float totalGpuTime = 0;
for (const IRenderPass* p : passes)
{
    totalGpuTime += p->GetGpuTime(device, readIdx);
    p->ResetGpuTimerQuery(device, readIdx);
}
```

At this point we have all the data needed to send to the dynamic resolution handler. It will decide if it should adjust the resolution scale value based on the updated CPU and GPU times plus some other internal heuristics. Using the decision making flow diagram above, the first thing to check is whether the application is GPU bound. Since we know how long it took to process the previous frame on the CPU, excluding the time spent waiting for present, and now we also have the latest GPU time then we can check if the GPU time is slower than the CPU time. There is one problem with this scheme and it has to do with comparing raw times instead of keeping track of the times for several frames such that it is possible to compute an average for example. This can lead to problems where a single frame performance spike can trigger DRS to lower the rendering resolution and this is a huge reason why Tempest computes the average GPU time. 

Consider the scenario where the game loads into a new map and it has to render the environment probe plus do all the usual convolution passes to generate all the mips. This is a very expensive process which is why it is always cached afterwards and if DRS is running it will see the GPU taking a long time to render a frame. In this case however, the long frame time was due to a one time event but if the DRS system is not aware of that then it will decide it has to lower the rendering resolution. Having a history buffer tracking the GPU times over several frames can mitigate this but it may not be enough. This is why Tempest also offers a sort of proactive API to tell the DRS system that something expensive and temporary is about to happen so it should ignore a specified amount of frames before it continues to work as normal. This API is also usable from the game scripting API so it is possible for the game to inform the DRS system about these types of events very easily.

Alright so lets say we are GPU bound, if we are at the minimum allowed resolution scale then there is nothing left to do and proceeds to render the frame. Otherwise, the system will compute a **delta scaling factor** that will modify the current dynamic resolution scale. Tempest uses an equation for this that is mostly based on the one discussed in [this paper](https://www.intel.com/content/dam/develop/external/us/en/documents/dynamicresolutionrendering-183334.pdf) and you can see it below.

![DeltaScale](/images/dynamic-resolution/DeltaScale.png "DeltaScale")

Where **S'** is the new resolution scale, **S** is the current resolution scale, **k** is the rate of change constant, **T** is the desired frame time, **t** is the current frame time (averaged in our case). In Tempest the rate of change constant is different depending on whether DRS is increasing or decreasing the resolution scale. When an application is GPU bound it's not a bad idea to lower the rendering resolution fast so that the application can re-establish a smooth framerate quickly enough and then increase it slowly over several frames if the GPU has enough headroom to give it more work.

Now lets consider the situation where the application is not GPU bound but DRS has already lowered the rendering resolution. This means there is an opportunity to increase the rendering resolution by increasing the resolution scale value which typically goes from [0, 1] or more realistically [0.5, 1] as rendering at half the chosen rendering resolution is a common minimum. Lowering the resolution scale can happen at any frame if DRS decides it has to happen but increasing it is done at a specified interval instead. This is in part to increase it gradually so it isn't too obvious to players when it happens but also mitigates a situation where the application ping pongs between lowering and increasing the resolution scale. It's a good time to talk about some of the other configurable parameters in DRS that are specific to increasing the resolution scale. As you'll note, there is a **GPU Headroom Before Increasing** parameter discussed previously. This threshold is used to determine if there is enough unused time on the GPU that it makes sense to increase the GPU work by incrementing the resolution scale. You compute the difference between the target framerate and the average GPU time and if that difference is greater than or equal to the headroom threshold then it is safe to increment the resolution scale.

One other important thing to do in relation to increasing the resolution scale is to incorporate a sort of dampening logic to your rate of change. What I mean by this is you can likely end up in a situation where DRS decides to increase the rendering resolution and in the next frame the application is GPU bound once again so you fall into that ping-pong situation mentioned earlier. You can extend the system to be able to identify when scenarios like that happens. Once you have that, then you can attempt to mitigate those scenarios and I think a good way to mitigate it is to have a "dampening" factor modulating your rate of change curve. Dampening can be increased by some engine specific amount such that gradually incrementing the rendering resolution can actually converge to the ideal scaling factor for the underlying hardware. You can also have logic to reduce this dampening based on any heuristics you decide to have. This can be as simple as reducing the dampening if there have been many consecutive frames where the GPU performance is good.

With dynamic resolution running, the engine needs to be aware of what the scaled resolution is and what the unscaled resolution is as both will be used to render a frame. In Tempest, DRS only affects the rendering resolution used to render the world but after post processing is done the renderer uses whatever the unscaled resolution is to render things like the UI. The unscaled rendering resolution is also needed to allocate the render targets used and the renderer uses the scaled resolution to figure out what region of that render target to render into. This avoids the situation that a lot of naive dynamic rendering resolution implementations fall into in which they re-allocated all of their render targets to the specified scaled rendering resolution. This is simple to implement but terrible for performance not to mention the memory fragmentation this introduces will likely cause a crash at some point.

One key thing to make draw calls work with this scheme is to set your rendering state such that it renders into the scaled region of the render target the pass will write into. In other words, your viewport needs to be set to the scaled rendering resolution. If you're dispatching compute shaders that write to the entire render target then those need to use the scaled rendering resolution when deciding on how many threads to dispatch.

## Temporal Issues

DRS can introduce some complications with certain rendering features. Depending on your implementation a rendering pass that depends on some temporal resolution stability can be problematic when combined with DRS. In Tempest, volumetric fog falls into this category if it is rendered with temporal smoothing as the froxels are created based on a given rendering resolution. If that changes at any given time, the data stored in the froxels is invalid and if the rendering resolution changes often then this data will never converge to what it should so it looks wrong to the player. The dynamic resolution handler informs the renderer if the resolution scale has changed before any actual rendering work is done. This information is used in the volumetric fog passes to inform the shaders not to use the previous frame data when blending its history with the current frame and also clears any of the temporal buffers so there is no stale data in the froxels. This means that temporal smoothing is disabled for the fog if it is normally enabled but it only does so for a few frames before enabling it again and carrying on as normal.

## Shader Considerations

Shaders that can sample any region of a render target need to take care not to sample a region that has not been rendered into when DRS is enabled. This typically means clamping the uv coordinates to range between [0, **scaledUV**] where **scaledUV** is the value of your dynamic resolution scale. This assumes it is never larger than 1.0 like it could be if supersampling is supported with DRS. 

If the shader relies on the texture sampler repeating the uv coordinate interval then this will need to be handled manually as now the max is not always going to be 1.0.

Another consideration is when a shader uses **Load** to sample a texture as the pixel coordinate used may need to be adjusted by the resolution scale.

Computing normalized screen space coordinates is also something that needs to be handled correctly.

Bloom and other features like it can have a black halo around the edges of the screen if uv coordinates are not properly handled.

## Debugging

In my opinion, there are three important debugging features that any DRS implementation should have. The first one is to be able to turn it off completely at runtime. This is helpful when looking for bugs. The second one is a simple way to report what the current value of the dynamic rendering scale is at. In Tempest, a simple screen space debug text is rendered alongside things like the CPU and GPU times and is toggled on through the in-game console. The other feature is to be able to manually trigger DRS to downscale the rendering resolution and have it automatically increase the rendering resolution when this debug feature is disabled. This is very important when looking for correctness in your rendering passes. If there are any issues this will uncover them very quickly and then you can address the problem.

## Example

With all of that said, here is a screenshot comparison of the usual test scene rendered at the unscaled rendering resolution and at 50% of that using the debug command to trigger downscales. For this example, that means unscaled is rendered at 1440p and the other one at 720p. You can see the UI looks exactly the same, as it should since it is rendered at 1440p for both, and the only difference is the quality of the 3D environment.

![Example](/images/dynamic-resolution/DRS_Example.png "Example")