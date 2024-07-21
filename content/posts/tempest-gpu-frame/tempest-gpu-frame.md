---
title: "Tempest Gpu Frame"
date: 2024-07-21T08:54:06-07:00
draft: false
toc: false
images:
tags: 
  - rendering
  - editor
---


* [Shadows Phase](#shadows-phase)
* [Depth Phase](#depth-phase)
* [PreLighting Phase](#prelighting-phase)
* [Lighting Phase](#lighting-phase)
* [ImGui Phase](#imgui-phase)

![Test Scene](/images/tempest-gpu-frame/TestScene.png "Test Scene")

This post will go over one of the rendering test scenes and we'll see the steps the Tempest engine takes on the GPU to render a frame. The engine has its own editor separate from the player executable, which is what is shipped to people. The editor does pretty much the same thing the player does plus a few editor only rendering tasks so the scene we will analyze is actually running in the editor. 

As mentioned in previous posts, the engine uses a deferred renderer with a gbuffer layout setup as shown in the screenshot below. The renderer uses several command lists and is broken up into several rendering phases. This can give opportunities to submit the command list once it is done recording on the CPU so we can keep the GPU fed over the course of the frame. 

![GBuffer Layout](/images/decals/GBufferLayout.png "GBuffer")

## Shadows Phase
This is the very first phase for the renderer. In this phase, we take care of rendering tasks that need to run as early as possible, for example, GPU skinning compute shaders run here to update vertex buffers before they are needed in subsequent passes. The main thing however, is rendering all the shadows needed for the frame. This means running cascade shadow maps pass and any local shadows required by the scene's local punctual light sources such as spot or point lights.

### Cascade Shadow Maps
CSM has a configurable number of cascades and caps out at a typical count of 4 cascades. If the renderer has to render more than one cascade then multithreading is used to record each cascade pass in its own command list. This means there are at most 4 command lists dedicated to rendering cascade shadows. Currently the architecture requires the first cascade to be rendered synchronously so as to make sure any required rendering resources such as PSOs are created and initialized before we can use multithreading to record the other cascades. This is not the most optimal solution but is simple and has worked well enough in practice. It is however, an area to improve in the future.

Different from a lot of other CSM algorithms, Tempest uses a technique based on [Sample Distribution Shadow Maps](https://www.intel.com/content/www/us/en/developer/articles/technical/sample-distribution-shadow-maps.html). Contrary to other techniques SDSM tries to automate the cascade configuration while trying to make the best out of the resolution of the individual cascade shadow map. It is a bit more involved to support as the camera's depth texture needs to be analyzed every frame and the result of that analysis read back to the cpu so it can generate the optimal partitioning for each cascade. This means that instead of worrying about stabilizing the cascades as the camera moves we can instead focus on making optimal use of the shadow map's resolution in such a way that we can get sub-pixel resolution when sampling said shadow map.

Tempest uses a texture 2D array to store each of the shadow map cascades. These days texture arrays are ubiquitous and I believe required for modern APIs. This simplifies setting up the rendering and sampling of shadow maps instead of using the age old trick of packing all cascades into one 2D texture. A preview of cascade 0 is below.

![Cascade0](/images/tempest-gpu-frame/Cascade0.png "Shadow Cascade 0")

### Local Shadows
Tempest supports directional, spot, and point light sources. Only one directional light source is supported though (sun or moon) and that will use the CSM method outlined previously. This limitation does not extend to the other supported light types which will instead use the local shadows system. This system will be stored in a 2D shadow map atlas whose resolution is determined by the user specified shadow quality setting. At its highest setting it will use a 4K resolution and goes all the way down to 1024x1024 in the lowest setting. In the editor the lights can specify what resolution to use for their shadow map or default to whatever the shadow quality setting will use. Lights can also specify the update frequency of their shadow map. This allows fully caching the shadow map from a light source to improve performance or we they can just render their shadow map every frame. These local shadows will only render if they can be stored in the shadow map atlas. If the atlas gets full while still having pending shadow map requests then it will just ignore those. Due to the limited space, the renderer will try to sort the shadow map requests based on distance to the camera rendering the scene so it tries to prioritize the shadow maps close by.

Unlike many other renderers, the point light shadow maps are not rendered into cube maps. Instead each face of the shadow cube map is rendered into the 2D shadow map atlas just like spot light shadow maps. This keeps the rendering code the same between the two light types and the only difference is in regards to sampling the shadow map. In the lighting pass, the shader needs to figure out what face to sample from in order to apply the correct shadow fragment. To do this it requires looking at the light's world space direction vector to figure out the cube map face. Each face has an id associated with it that the shader can then add to the light buffer's shadow index to get the correct shadow map. The renderer has a punctual light buffer that contains all the required data on a light source and also contains a shadow buffer for local light sources. The shadow buffer itself has all the needed information to sample the correct region in the shadow map atlas. The light buffer has a index into this shadow buffer so it can retrieve the shadow specific information for that light source. In the case of a point light, that index will point to the start of its shadow cube map, in other words, the positive X face of the cube map. It is useful to separate these two buffers since not all light sources will cast shadows so it makes sense to optimize memory usage.

```hlsl
#define CUBEMAPFACE_POSITIVE_X 0
#define CUBEMAPFACE_NEGATIVE_X 1
#define CUBEMAPFACE_POSITIVE_Y 2
#define CUBEMAPFACE_NEGATIVE_Y 3
#define CUBEMAPFACE_POSITIVE_Z 4
#define CUBEMAPFACE_NEGATIVE_Z 5

float CubeMapFaceID(float3 dir)
{
    float faceID;

    if (abs(dir.z) >= abs(dir.x) && abs(dir.z) >= abs(dir.y))
    {
        faceID = (dir.z < 0.0) ? CUBEMAPFACE_NEGATIVE_Z : CUBEMAPFACE_POSITIVE_Z;
    }
    else if (abs(dir.y) >= abs(dir.x))
    {
        faceID = (dir.y < 0.0) ? CUBEMAPFACE_NEGATIVE_Y : CUBEMAPFACE_POSITIVE_Y;
    }
    else
    {
        faceID = (dir.x < 0.0) ? CUBEMAPFACE_NEGATIVE_X : CUBEMAPFACE_POSITIVE_X;
    }

    return faceID;
}
```

Now that all shadows have been recorded, the command lists for them get submitted to the graphics queue and we move on to the next phase.

## Depth Phase
As the name suggests, this phase will render the scene's depth into a texture but will also do several other render passes that only depend on the depth texture. Examples of passes that run in this phase are ones like object velocities, a depth pyramid used for ray-marching, distortion maps, etc. I'll expand on some of them here.

### Prepass
Some argue the usefulness of a depth prepass in a deferred renderer but at least in Tempest there is a performance benefit to it so the renderer will always do it. Tempest uses a inverted depth range so fragments close to the camera appear white and the farthest fragments will be black. This is done to improve the depth texture accuracy. The renderer also uses an infinite projection matrix so there is technically no far plane. In this pass the depth test is set to **greatar than or equal** (due to reversed depth) and samples the albedo texture and alpha clip for masked geometry. Other rendering passes will set the depth test to **equal** and do not have to explicitly handle masked geometry as the depth test will take care of discarding fragments that would normally be handled by alpha clipping. By doing this fragment shaders do not need to have a clip function to discard alpha clipped fragments and in doing so the hardware depth testing optimizations will very likely be turned on since it does not have to evaluate the fragment before doing the depth test like it does with alpha clipped or blended geometry.

![Depth Texture](/images/tempest-gpu-frame/DepthTexture.png "Depth Texture")

This is also where the stencil values are populated. Each bit of the 8 bit stencil value is used for specific purposes. The big features this allows is creating projection channels for screen space decals so you can exclude some meshes from receiving specific decals and the other big one is lighting channels which does the same thing decal projection channels do but for lighting. One bit is used for excluding certain objects from specific rendering features and the last remaining bit is open for something in the future.

```hlsl
// Stencil layout during the decal and deferred lighting pass
//     BIT ID | USE
//     [0]    | custom bit(bit to be use by any rendering passes, but must be properly reset to 0 after using)
//     [1]    | unused
//     [2]    | Receive decals
//     [3]    | Receive decals
//     [4]    | Receive decals
//     [5]    | Lighting channels
//     [6]    | Lighting channels
//     [7]    | Lighting channels
```

### Distortion Map Pass
In this test scene there is a single distortion particle rendered around where the two spheres on the left are. This effect is done in multiple passes but first you have to create the distortion map itself. In essence what the distortion map is a fullscreen UV distortion effect. It creates it by rendering models and particles that contribute to the distortion into a fullscreen render target with a format of RG16F. It sets the blend mode to additive so any overlapping distortion models just add to the distortion strength. Most of the draw calls in this pass will rely on a normal map to create the distortion result.

![Distortion Pass](/images/tempest-gpu-frame/DistortionPass.png "Distortion Pass")

As you can see from the screenshot, the particle is rendered behind geometry into the distortion map. The result is a properly depth tested particle that creates a distortion based on a shockwave normal map texture that you can see on the right side of the image. After all draw calls are done you end up with a fullscreen texture containing UV distortion offsets that will be applied at a later time in the frame.

### Depth Moments
The last thing I'll point out here is the rendering passes to build a blurred depth moments pyramid. This is currently only used during the screen space global illumination pass. What this rendering pass will do is take the current scene depth texture rendered earlier in the frame and converts it to a depth moments texture with mips so that it can be linearly filtered later on when sampling the mip chain. This is more or less the same as what variance shadow maps do where a moment is a two channel texture with r = depth and g = depth*depth. This allows the shader to use Chebyshev's inequality to sample the depth moments texture and the fact it has mips can allow the shader to sample those during ray-marching for a performance boost.

## PreLighting Phase 
At this point the renderer can start building the data needed for lighting. Anyone familiar with a deferred renderer will know this to be the point where the gbuffer pass will be done. So this phase is essentially rendering the gbuffer, render passes that modify it, and any other rendering passes that contribute to the lighting. 

### GBuffer
This is where all the gbuffer data for the frame gets created. All the draw calls will set the depth test to **equal** with depth writes disabled and test against the depth texture generated in the previous phase. Not surprinsingly it uses MRT to output to 5 different render targets. The gbuffer layout image at the start of the article covers the 4 render targets that form the gbuffer data but there is a fifth render target that gets initialized to the emission. That dataset isn't explicitly stored in a gbuffer since there was no need to sample that data later on in the frame. As such it was decided to store that data in the render target that will contain the lighting of the frame and reduce memory usage that way.

Here's all the gbuffers rendered in the test scene, ordered in the same way as the gbuffer layout describes it to be.

![GBuffer](/images/tempest-gpu-frame/GBuffer.gif "GBuffers")

After rendering the gbuffer we would normally render any decals but this test scene does not contain any so for more info on that you can look at the previous post about decals in Tempest.

### Screen Space AO
So the gbuffer can store any material based AO that comes from a texture but that's often not enough. To complement that, Tempest has support for [Ground Truth AO](https://www.activision.com/cdn/research/Practical_Real_Time_Strategies_for_Accurate_Indirect_Occlusion_NEW%20VERSION_COLOR.pdf). At the time it was implemented it was the most modern SSAO implementation but these days it might be better to support something like [this](https://ar5iv.labs.arxiv.org/html/2301.11376) instead. As the name implies this will create a fullscreen ambient occlusion texture that can then be combined with the gbuffer AO values to get very good AO results. The highest quality version of this technique will also output a fullscreen bent normals texture that will be used for GI sampling. For other quality settings it will instead use the world normal stored in the gbuffer.

### GBuffer Normals Blur
The ground environment for the game being made with Tempest is made out of cubes. These cubes have very hard edges that for stylistic purposes I wanted the edges to look a little softer. One way to do that without explicitly modeling the geometry is to blur the gbuffer storing the normals. This is what is done in Tempest and the idea originally came from a game called [Teardown](https://www.teardowngame.com/). The developer of that game had done a youtube video interview at some point where he talked about different aspects of the renderer used in that game.

## Lighting Phase
Now the renderer can start putting together the lighting results for the frame. The first thing done in this phase is the deferred lighting pass which is done in a compute shader. This pass will use all the gbuffers, shadow maps, SSAO, and the irradiance map for the sky. So the deferred lighting applies direct lighting and the skylight but there is still more to do with the lighting. Here's what the scene looks like after applying deferred lighting.

![Deferred Lighting](/images/tempest-gpu-frame/DeferredLit.png "Deferred Lighting")

### Sky
Atmospheric scattering and physically based sky is computed and applied after deferred lighting. The sky is based on [Hillaire's sky](https://github.com/sebh/UnrealEngineSkyAtmosphere) which is the same solution in Unreal Engine but with more bells and whistles. That technique is the best one I've seen yet for both performance and the quality you get from it. To apply the sky to the scene lighting, a fullscreen triangle is used instead of a cube or sphere like many other solutions do. The trick is to render the fullscreen triangle at the far plane and enable depth testing. After that it's just a little bit of math to compute ray-marching parameters. 

That said the sky component in the engine also supports using a HDRI for the sky or go more stylized and create a sky based on a color gradient.

### SSR and SSGI
If SSR or SSGI is enabled, a mip chain is created from the current lit scene color where each mip is blurred. These mips are sampled based on material properties such as roughness where the higher the roughness the higher the mip is used. There's nothing special about the SSR implementation but the SSGI implementation is somewhat different from others. It is based on [SSVGI](https://github.com/Raikiri/LegitEngine) which comes from one of the senior programmers that works on Path of Exile. This implementation stood out to me as it is not one that is requiring a temporal solution to look good and I've been favoring techniques that do not rely on temporal filters to look good or run performantly. 

And this is what the scene looks like after applying the remaining GI to the lighting results.

![Lit with GI](/images/tempest-gpu-frame/DeferredLit2.png "Deferred Lit + GI")

### Volumetric Fog
This is the only effect in this frame that is allowed to have a temporal filter as it is quite important for performance. Nothing unique here as if you've seen one implementation then you've seen most. Only thing I'll mention here is that the volumetric fog has support for rendering particles that will contribute to the fog's density. This creates localized fog effects using the particle system in the engine. It is a very nice effect but can get expensive very fast as the particles need to be rasterized multiple times based on how big they are.

### Grid Floor and Icons (EditorOnly)
The grid floor most people are familiar with gets rendered with no depth testing so it is easy to see and any entity icons for light sources or other entities is also rendered. The engine can also toggle rendering these things and make it look like how it does in the game view.

### Transparency
Due to Tempest using a deferred renderer, transparency needs to have its own solution. There is support for dithered transparency which does allow rendering those models in the deferred renderer but the visual quality can be unacceptable. Other techniques like depth peeling are more expensive than they are worth so Tempest does what most other engines do in this scenario and just render transparency using a forward renderer. As of now it only supports lighting with the directional light source as local lights support has not been needed yet. Unlit transparent models get rendered right after the lit transparents do and this includes particle effects. 

One other transparency solution used is order independent transparency via [Moment-based OIT](https://cg.ivd.kit.edu/english/mboit.php). This is sparingly used however, as it is a multi-pass technique so it can get expensive rather quickly. Only unlit models are supported as of now and can be seen in the test scene as the left-most trio of cubes are rendered using this method. 

### Fullscreen Passes
Now that all geometry has been rendered, the only thing left are all the fullscreen rendering passes like SMAA and any other post process effect (bloom, exposure, etc.). 

![Final Lit](/images/tempest-gpu-frame/FinalLit.png "Final Lit")

### In-game UI
The in-game UI solves the age old problem of rendering many quads in a efficient manner. All the UI elements use the same shader and rely on a texture atlas big enough to store all the UI textures. Every frame a vertex buffer is built based on what is visible on screen at that time. Due to the UI system using the same vertex buffer for everything and using the same material means that it can all be rendered in one draw call. One cool thing with the UI is that we can reuse the bloom texture and blend with a UI element to create that frosted glass look for free essentially. This effect is visible on the UI menu.

![Final Lit UI](/images/tempest-gpu-frame/FinalLitUI.png "Final Lit UI")

## ImGui Phase
This phase will only run in the editor. After the scene rendering is done, the editor will render all the UI elements for the editor. The texture containing the scene lighting will also get rendered as a ImGui image. The aspect ratio of that scene rendering is optionally preserved so the user can choose to have the image take up all available space or just enough to maintain its aspect ratio.

All of this and more things that I didn't mention was able to be done in a laptop using a NVidia 2060 rendering the scene at 1440p while the editor rendered on a ultra wide screen monitor and it took around 19 milliseconds. The bottleneck being heavily skewed towards all the fullscreen effects like SSR and SSGI. That said a 2060 isn't really suited to render a PBR frame at 1440p 60 frames per second. All things considered I'm especially happy with the performance considering there are still things to do to optimize the engine that will eventually be done.