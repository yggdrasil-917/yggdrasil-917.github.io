---
title: "Decals"
date: 2024-07-15T18:11:49-07:00
draft: false
toc: false
images:
tags: 
  - rendering
---

* [Tempest GBuffer](#tempest-gbuffer)
* [How They Work](#how-they-work)
* [Sampling Stencil](#sampling-stencil)
* [Sort Order](#sort-order)
* [Batch Rendering](#batch-rendering)
* [GBuffer Modidfication](#gbuffer-modifications)
* [Unique Applications](#unique-applications)
* [Performance](#performance)


There are a couple of different ways to implement decals, each with their pros and cons. For this post I'll go over how Tempest implements screen space deferred decals to modify the gbuffer in various ways. Decals tend to offer very easy ways to modify the scene while keeping shader complexity at a manageable level but at the cost of additional draw calls to render the decals.

## Tempest GBuffer
Before going into details about the decal implementation, it is worth going over some of the details of Tempest's gbuffer layout. 

![GBuffer Layout](/images/decals/GBufferLayout.png)

There is also one additional render target that isn't technically part of the gbuffer but still very important to the final render. That is emission and GI which is stored in the render target that stores the final scene lighting. It's done this way so as to not have a dedicated render target for both emission and global illumination, in other words, a memory optimization.

For anyone familiar with other gbuffer layouts, this shouldn't come as any surprise as it is very similar to a lot of other layouts out there so I'll only call out a few details about it. The gbuffer normals are stored in a two component render target. This is done by converting a typical world normal unit vector to a [octahedron encoding](https://jcgt.org/published/0003/02/01/). There are very fast ways to encode and decode using this method that it allowed the usage of a render target format that has a smaller bandwidth than other formats without encoding. This is one area where I feel like every deferred renderer solves the storage of normal vectors differently. For instance, another popular solution is to use no encoding and have a render target format of RGB10A2 so it adds up to 32 bits per fragment, the same as the format Tempest uses for world normals. I found that using octahedron encoding and 16 bits per channel yielded slightly better results than the latter option but again the differences are minimal. The other thing to expand upon is what exactly is stored in the last gbuffer render target. The alpha channel of GBuffer2 stores the id of what BRDF is to be used for that fragment during lighting and GBuffer3 stores any data needed to accomplish that. One example of data stored in GBuffer3 is subsurface color and its intensity used for foliage lighting. 

## How They Work
These decals modify gbuffer content and as such they are rendered right after the gbuffer pass is done. All the decals use the same underlying mesh, a unit cube centered around the origin. The depth test and depth writing is turned off when drawing a decal. This is largely done as the shader needs to sample the depth texture and the stencil values. Unfortunately it isn't possible to bind the depth texture in read only mode and sample it in the shader as otherwise the DirectX debug layer screams at you. You can of course copy the depth texture and get around it that way but there aren't enough reasons to copy the depth texture at the moment so it isn't worth paying the performance cost to copy the depth texture. The last render state to configure before drawing the decal is the blend mode to use. What blend mode to use depends on what type of information is being modified and will discussed in more detail later in this post.

With drawing a cube to represent the volume of the decal we also need to somehow carve out the correct shape of the decal based on the geometry the decal is projecting onto. Luckily we can do that with the depth texture and a little bit of math. We can compute the fragment's world position using the depth texture and then we can transform that to the decal's local space. Once we have that we can do a simple box bounds check to determine if the fragment is actually part of the decal or not. 

```hlsl
// Bounds check in decal's local space to discard fragments that
// are not part of the decal. This is where the fact we use a
// unit cube for the volume comes into play.
clip(0.5 - abs(decalPosOS));
```

At this point we have the decal's shape and we can now compute the texture coordinates using the decal position in object space so we can sample any textures the decal will use.

```hlsl
// decalPosOS is in the [-0.5, 0.5] range so remap it to [0, 1] and then we can do the usual 
// UV scale and offset transform for any sort of tiling support.
float2 decalUV = (decalPosOS.xz + 0.5) * g_Material.scaleAndOffset.xy + g_Material.scaleAndOffset.zw;
```

## Sampling Stencil
It's pretty rare to find any information about sampling the depth texture and getting the stencil value from it. Most of the time you will have the hardware do the stencil test so you don't have to worry about handling the stencil values directly but there are times where you have to. This is one such time as we are not doing depth testing when rendering the decal for reasons outlined previously. In order to do this we have to properly setup the shader resource view for the depth texture so that we can access the stencil value. Tempest uses a **D24S8** format and that means we need to use **DXGI_FORMAT_X24_TYPELESS_G8_UINT** when setting up the resource view. Then we can sample the depth texture like the sample below in order to get the stencil value for any given fragment.

```hlsl
Texture2D<uint2> SceneStencil : register(...);
// Check if we can project on this fragment based on the stencil value
{
  const uint projectionMask = (uint)g_Material.buffer[1].y;
  const uint receiveDecal = (SceneStencil.Load(uint3(i.screenPos.xy, 0)).y >> STENCIL_RECEIVE_DECALS_BIT) & 0x7;
  if ((projectionMask & receiveDecal) == 0)
    clip(-1);
}
```

With the ability to sample the stencil value in the shader we can now exclude specific models from receiving decals. There are times where we may have moving models that intersect the decal volume but we don't want them to receive any of the decal projection. The way Tempest handles this is to use 3 bits from the stencil buffer to create decal channels. With 3 bits we have 3 channels to use and with a little bit manipulation we can check for overlapping channels and decide if a fragment receives the decal being projected.

## Sort Order
If you have mutiple overlapping decals, the renderer should have a way to determine the blend order. How this is achieved is sort of engine specific but there's often some underlying sorting system for generating the order of draw calls. Then using a sort bias you can change the priority of how the decal is rendered. Without getting into details as that topic is out of scope for this post, this is what Tempest ends up doing. Tempest has support for generating a sort key based on various parameters and using that for sorting. The decals have a sort order they can specify that will end up modifying the generated sort key value and in effect changes the sorting results. The higher the sort order value the later the decal is rendered. 

In the image below, there are two overlapping decals. On the left half both have the same sort order and on the right half the red tinted decal has a higher sort order so it gets rendered on top of the other decal.

![Sort Order](/images/decals/DecalOverlapMerged.png)

## Batch Rendering
It is very easy to get into a situation where you have a lot of decals being rendered and while each individual decal is fast to render it adds up quickly. Tempest has a batch rendering solution used by all models it renders except in a few specific cases. This made decals use this system without doing any extra work and so any decal using the same material will be instanced. There are some unique cases that need to be handled for decals to support all the desired features. For example, using the sort order mentioned above can break batching but the flexibility it provides outweighs the costs.

## GBuffer Modifications
The decal's shader can modify any property of the gbuffer in its draw call. All gbuffer render targets are bound during the decal's draw call as it was decided to have one decal shader instead of having lots of permutations that only write to the properties it is configured to change. All render targets have the typical transparency blend mode except for the emission output which uses an additive one. 

Here is an example of a decal that changes the emissive values. Tempest supports a screen space global illumination pass which ends up picking up the emission changes as well as you can see the bounced lighting on the wall.

![Decal Emissive](/images/decals/DecalEmissive.png)

Here is another example that blends the normal stored in the gbuffer normal. This one however, isn't entirely accurate as the blending is being done between encoded values where as it should be done with the decoded values before encoding back to the octahedron format. That would require copying the gbuffer normals into another texture and just isn't worth it. The blending strength can be adjusted until you get something good enough and so far has worked well enough.

![Decal Normal](/images/decals/DecalNormal.png)

## Unique Applications
Using a lot of this foundation to render a screen space decal you can also extend this system to handle a few extra situations. One such example is a cheap omni light source without actually using a point light and going through the PBR pipeline. These decals do just about everything the typical decal does however you do not have to do the local space box bounds check to create the decal shape. In the case of the omni light you can use a circle gradient function that is tinted by a user specified color and intensity. The size of it is determined by the size of the decal itself and the result is added to the emission values.

Here is an example of an omni light that is tinted red.

![Decal OmniLight](/images/decals/DecalOmniLight.png)

You can apply essentially the same ideas as the omni light decal but for ambient occlusion stored in the gbuffer. Using the same circle gradient function you can modulate the AO but you must make sure to have a blend mode that modulates the results for this to work.

## Performance
Performance for screen space decals will be determined by the complexity of the shader and how big the decal volume is. Overlapping decals will have a negative effect to overall performance as you are re-evaluating fragments. That said decals can be rendered very fast and making good use of the batching system also helps a lot. Some benchmarks done with RenderDoc have shown the decals in Tempest to take around 10 - 20 microseconds for each decal draw call so it's pretty fast. The omni light and AO decals will be a bit more expensive but not by much if there size is comparable to other decals.