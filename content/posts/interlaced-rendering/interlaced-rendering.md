---
title: "Interlaced Rendering"
date: 2024-07-26T11:20:44-07:00
draft: false
toc: false
images:
tags: 
  - rendering
---

* [What Is It](#what-is-it)
* [Why Use It](#why-use-it)
* [Implementation](#implementation)
* [Deinterlacing](#deinterlacing)
* [Performance](#performance)
* [Final Thoughts](#final-thoughts)

## What Is It
A more common name might be interlaced video. The idea originates decades ago in the TV industry where companies wanted to save on bandwidth. To quote Wikipedia, the interlaced signal contains two images of a video frame captured consecutively which enhances motion perception to the viewer, and reduces flicker by taking advantage of the characteristics of the human visual system. In other words, the images are rendered at half resolution in either the vertical or horizontal axis. Then the current image and the previous image are combined to reconstruct a new image that has twice the resolution of the individual images. [Here](https://en.wikipedia.org/wiki/Interlaced_video) is a nice gif animating the process of interlacing two video fields to construct a higher quality image.

## Why Use It
In realtime rendering, we can use the same idea as interlaced video to reap similar benefits. Assuming your overall performance is GPU bound then using lower bandwidth will yield better performance. Since Tempest uses a deferred renderer, being GPU bound is a common occurrence so implementing interlaced rendering will generally have a positive effect on performance. Using interlaced rendering implies using lower resolution render targets when rendering the 3D world. Any computationally intensive rendering passes will benefit from a lower resolution output. So adopting interlaced rendering can introduce performance benefits in various areas of the pipeline.

So if it can have these benefits then why aren't more engines using this type of rendering? Reconstructing the final image from two lower resolution images does introduce some rendering artifacts, especially when things are moving fast. The screenshot below shows an example of interlace artifacts after image reconstruction. The image is zoomed in to make the issue obvious and during normal gameplay it often doesn't stand out unless you know where to look. If you've never seen it, I recommend looking at the [presentation](https://www.slideshare.net/slideshow/dynamic-resolution-and-interlaced-rendering/253712465#68) that goes over interlaced rendering, which is also where the image below is from.

![Interlaced Artifacts](/images/interlaced-rendering/InterlacedArtifacts.png "Interlaced Artifacts")

Other non-temporal based image reconstruction techniques suffer from similar issues. If you're familiar with [Checkerboard Rendering](https://pixiogaming.com/blogs/latest/checkerboard-rendering-an-ingenious-approach-to-4k-gaming) techniques that were used when the PS4 Pro was released then you will know what I'm talking about. Eventually advanced spatio-temporal upscalers got good enough to replace interlaced and checkerboard rendering solutions and the industry hasn't really looked back.

## Implementation
In Tempest, the goal is to not use any temporal upscaler or TAA solution to construct the final image. This is only for stylistic reasons as I personally miss the sharp look frames used to have before these temporal solutions became ubiquitous. For this reason SMAA is the anti-aliasing technique of choice in Tempest. That said the engine does support FSR2 but it is mostly meant for comparing its results with other techniques in the engine. If by the end of the game's dev cycle the FSR solution is still working correctly then it might be available for users to enable through the graphics settings but the focus is to not rely on these techniques as much as possible.

Before going into the implementation details of interlaced rendering I've added a gif below of the rendering test scene. It has two numbered frames 1 and 2, respectively. One frame is rendered using the normal progressive solution and the other frame uses interlaced rendering to construct the final image using the current and previous frame. Do you think you can spot which frame is using interlaced rendering? 

![Interlaced Comparison](/images/interlaced-rendering/InterlacedRenderingComparison.gif "Interlaced Comparison")

If you look at the two frames carefully then you will likely be able to tell which frame is using interlaced rendering. However, while playing the game you're likely not thinking about that though so you won't even notice the artifacts most of the time. Without further ado the frame using interlaced rendering is the one marked as number one.

Since most rendering resolutions used today are widescreen, the interlaced rendering solution will halve the width of the render targets as that will yield the best performance. The solution could be extended to halve vertical or horizontal resolution based on the aspect ratio but to keep things simple it will always halve the horizontal resolution. So for example, if the selected rendering resolution is 2560x1440 then the render targets used for the 3D environment will be set to 1280x1440. 

### Deinterlacing
After all the geometry has been rendered, a deinterlace pass runs to reconstruct the image to the selected rendering resolution. In the example above, this would mean deinterlacing into a render target with the resolution of 2560x1440. The image is not yet presented to the backbuffer but instead now is when all fullscreen effects will run and the pipeline is the same as the normal progressive renderer. The presentation on interlaced rendering describes several different ways to deinterlace the image and I highly suggest you read the presentation for more details as I will only cover the deinterlace method used in Tempest.

The chosen deinterlace method is referred to as "Weave" in the linked interlaced rendering presentation, of which you can see a slide from it below. The basic idea is to insert the pixel columns of the new image in between every column of the previous image at which point you end up with an image that is twice the width of the previous images.

![Weave Slide](/images/interlaced-rendering/WeaveSlide.png)

Like the slide mentions, reconstructing static scenes works extremely well and most of the deinterlace methods described in the presentation are fast methods that can achieve that quality. Dynamic scenes can however, show artifacts as previously detailed in this post. In those cases a more expenside deinterlace method can minimize those issues. Such a method is the temporal deinterlaced method but supporting it requires keeping track of several additional render targets so memory and performance requirements are higher. Keeping with the theme of the Tempest engine, no temporal solution is used and I typically prefer simple solutions if the result is good enough.

To perform the deinterlace pass, a fullscreen draw call with a custom fragment shader is used. The renderer does not explicitly store the previous frame's interlaced rendering results in order to keep VRAM usage as low as it can be while doing interlaced rendering. It does however, persistently store the result of the final scene color in a render target and this is what is used to represent the previous frame in the deinterlace pass. We can't read and write to it in the fragment shader so what the renderer does is setup hardware blending using the typical transparent blend mode. So for the fragments that do not need to be updated in the draw call will simply return a fully black pixel, including alpha set to 0. This will leave those fragments unmodified after the draw call and the fragments that need to be updated will simply sample the interlaced scene color render target and returns that value. Below is a code snippet detailing what the fragment shader function does. The final piece of the puzzle is to jitter the projection matrix every frame similar to what is done with TAA but only jittered horizontally in this case. Then we alternate updating the odd pixel columns on one frame and the even ones on the next frame so the entire reconstructed image is updated.

```hlsl
// Deinterlace fragment shader
float4 MainPS(in float4 i_position : SV_Position, half2 i_texcoord : TEXCOORD0) : SV_Target
{
  float2 uv = i_texcoord * g_Data.viewportScale;
  float4 finalColor = float4(0,0,0,0);
  uint pixelX = (uint)i_position.x;
  bool oddPixel = (pixelX & 1) == 1;
  bool doOdd = g_Data.updateOddColumns != 0;
  if ((oddPixel && doOdd)
  	|| (!oddPixel && !doOdd))
  {
    finalColor = SceneColor.SampleLevel(s_LinearClampSampler, uv, 0);
  }
  
  return finalColor;
}
```

## Performance
The performance test was done with a rendering resolution of 2560x1440 so interlaced rendering uses a 1280x1440 resolution and it is running on a 2060 GPU. Using RenderDoc to time the GPU frame we can see that the rendering test scene can take around 10 - 11 milliseconds to render without interlaced rendering and can be as fast as 6 milliseconds to render with interlaced rendering. That is fairly close to a 2x speedup which is amazing considering that only the world is rendered using interlaced rendering while the rest of the pipeline uses the reconstructed image.

It is also important to mention that this interlaced rendering implementation makes good use of the 2x2 warp that fragment shaders typically execute as. If you search for other implementations of interlaced rendering reveals variants that use the stencil buffer to disallow rendering into odd or even pixel columns during the world rendering and then they complain about no performance improvements. This is absolutely not the way to implement interlaced rendering and should be avoided in favor of something like what this post has outlined.

## Final Thoughts
Interlaced rendering has so far provided many benefits to both runtime performance and memory usage while only introducing a handful of rendering artifacts. It seems to be really useful for the kind of game being made with Tempest where the camera is setup to offer a bird's eye view of the map. For games that require the camera to be up close to the environment may have some difficulties preserving model silhouettes with interlaced rendering. I suspect depth discontinuities would present themselves as flickering when a naive deinterlace method is used. The RE engine, used for Resident Evil titles, has demonstrated that interlace rendering can work well in those games. I suspect they probably use a temporal based interlaced rendering solution but I haven't done too much research on their implementation as there isn't any info about it that I could find.

Bottom line is that this technique is very useful and can be considered an alternative to modern spatio-temporal upscalers. It's also much simpler to implement and maintain than any upscaler, except for maybe FSR 1.0 since it is just a spatial upscaler. I do wish more developers would consider using and iterating on techniques like interlaced and checkerboard rendering as I think they can be improved even more. This can then offer more options to gamers as there is a camp that does not like the softness of images or ghosting artifacts produced by DLSS and its ilk while others have less tolerance for jaggies so they opt for upscalers.