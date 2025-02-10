---
title: "Reflections Using Oblique Projections"
date: 2025-02-10T08:22:09-08:00
draft: false
toc: false
images:
tags: 
  - rendering
---

* [Tempest Reflections](#tempest-reflections)
* [Oblique Projection](#oblique-projection)
* [Creating Reflections](#creating-reflections)
* [Rendering Layers](#rendering-layers)
* [Sampling Reflections](#sampling-reflections)
* [Optimization](#optimization)
* [Limitations](#limitations)

Before the arrival of realtime raytracing APIs, reflections in rasterized graphics remains a very difficult problem to solve generically from a memory and performance standpoint. Not to say realtime raytracing fully solves it but it provides a much better framework to build upon in my opinion. If you look at the landscape of reflection techniques in rasterized graphics you'll find a variety of solutions that have been developed over the years but many of them have some cons that may be too big to ignore based on your needs.

Today it is pretty common for games to default to using some type of screen space reflection solution as they have proven to fit a wide variety of frame time budgets while still providing a decent enough quality in a lot of situations. Add to that the fact that they can be pretty easy to turn on in your project if you are using something like Unreal or Unity. If that is not the case, they are still not that difficult to write your own implementation assuming your renderer already has many of the required inputs to it. All that said they however, fail at the usual stuff screen space techniques do which, in my opinion, can make SSR a less desirable feature to use. Another downside to them is using any form of temporal filtering of your SSR results as that introduces ghosting and handling transparents is not trivial.

## Tempest Reflections
Despite the issues with SSR mentioned above, Tempest does support them through the use of the scene volume component API where enabling them is just ticking a checkbox and as long as the graphics settings of the user allows them they will then have screen space reflections in the scene. The engine also supports another variant of SSR known as screen space planar reflections ([SSPR](https://remi-genin.github.io/posts/screen-space-planar-reflections-in-ghost-recon-wildlands/)), similar to what was implemented in Ghost Recon. With this technique you can get clean mirror like reflections without launching a lot of rays to compute reflections but again it is a screen space technique so it has many failure cases. This technique is also integrated into the volume component API and the user can select between using these or traditional SSR. Screen space planar reflections do require a little more setup in regards to placing the mirroring planes in the scene so the algorithm can compute the reflections. 

In my search for a good way to render reflections for my game, I wasan't satisfied with the results coming from any of the screen space techniques. With the camera placement in combat, it was very hard to see the reflections in water surfaces so it was hard to justify the amount of time spent rendering them when their contributions were minimal. So instead of going with a screen space technique I started to think back on other rasterized reflection systems. Things like [parallax corrected cubemaps](https://seblagarde.wordpress.com/2012/09/29/image-based-lighting-approaches-and-parallax-corrected-cubemap/) were considered but only briefly as many of the combat levels, while cube-like, would have a large variation in height so it made them impractical from memory standpoint and blending. 

So I went back, digging deeper into my memory to recall other techniques I've used for reflections. This is when I remembered the old school way of making reflections, back when even shaders didn't exist. If you've been around long enough then maybe you've heard of the NeHe rendering tutorials from maybe two decades ago. That was the first time I can remember implementing fake reflections, from [this](https://webcf.waybackmachine.org/web/20241013154308/http://nehe.gamedev.net/tutorial/clipping__reflections_using_the_stencil_buffer/17004/) wayback machine link to the tutorial I had done. Essentially it uses the stencil buffer combined with duplicate models that you want to include in the reflections. So the idea was you would duplicate and mirror the meshes you wanted to reflect. Then you would render them with stencil operations in a specific way to create something that looks like reflections. Anyways, remembering this jogged my memory and I remembered there is a more modern way of doing this but I feel like it isn't talked about much these days. This is where rendering the scene with a view that uses an oblique perspective projection matrix comes into play.

## Oblique Projection
So what exactly is an oblique projection matrix? In a nutshell, they can be used to represent a projection where the view is slanted or skewed instead of directly perpendicular to the projection plane which is what your typical perspective projection matrix does. An oblique projection can be useful when trying represent 3D objects on a 2D plane that's placed anywhere in the 3D environment. This just so happens to be our use case when combining an oblique projection with a planar surface to act as our mirror so think of a water surface or even an actual flat mirror as our 2D plane we will use to project the reflections on.

To put it into context, below is a screenshot comparing what your typical frustum looks with a normal perspective projection at the top and what the frustum could look like with an oblique projection in the bottom. You'll notice it is very skewed in comparision to your typical perspective projection matrix which has a symmetrical frustum along the view direction.

![Projection Comparison](/images/reflections-using-oblique-projections/Perspective_Oblique_Comparison.png)

So the next question is, how do you make an oblique perspective projection matrix?

In our case, we only need oblique projections for planar reflections. As such, you can create the matrix from an existing projection matrix combined with information about your reflection plane. We represent that plane using its normal and distance from the origin so we can neetly represent it with 4 floats. This is all done in a simple function like the one below.

```cpp
// The input projection matrices are for the main camera rendering the scene.
// Also the clipPlane is in view space for our implementation.
static Math::float4x4 CalculateObliqueMatrix(const Math::float4x4& projection, const Math::float4x4& projectionInv, Math::float4 clipPlane)
{
    Math::float4x4 oblique = projection;
    
    // Calculate the dot product of the clip plane with the column vectors of the inverse projection matrix
    float dotProduct = Math::Dot(clipPlane, Math::float4(projectionInv[3][0], projectionInv[3][1], projectionInv[3][2], projectionInv[3][3]));
    
    // After this the near plane of the projection matrix becomes the clip plane 
    // or in other words the mirror's surface is the near plane of this projection
    oblique[0][3] -= clipPlane.x * dotProduct;
    oblique[1][3] -= clipPlane.y * dotProduct;
    oblique[2][3] -= clipPlane.z * dotProduct;
    oblique[3][3] -= clipPlane.w * dotProduct;
    
    return oblique;
}
```

## Creating Reflections
Now we have an oblique projection matrix but we are not at the point where we can render reflections so lets go over the rest. Since we want to allow the mirroring surfaces to be anywhere in the environment, we need a way to tell the engine where they are and what their orientations are. In Tempest, we have an entity that contains a component specialized at rendering the environment independent of the main camera. Naturally these components default to having a transform component as we need a way to represent them in the environment. The transform component can give us the position and orientation of the reflection plane in the environment. Its forward vector will correlate to the normal of the plane and the distance is simply the dot product of the normal and the entity's position negated.

The next step is computing the so called reflection matrix based on the reflection plane computed above. Remember, we need a way to mirror the models when the planar reflections camera is rendering the environment. If we don't do this then the models will not have the correct orientation in the mirrored render target and then the illusion fails. Back in the NeHe tutorial days, this step would be done manually by setting the correct position and scale of the models you wanted to render in the reflections. A matrix multiplication is a lot cheaper these days than it was a couple decades ago so computing the reflection matrix is not going to be a problem. Anyone that has used Unity for a while now will likely be familiar with the function below that computes the reflection matrix.

```cpp
// This creates a matrix that will reflect points across the clipPlane
static Math::float4x4 CalculateReflectionMatrix(Math::float4 clipPlane)
{
	Math::float4x4 m;
	m[0][0] = (1.0f - 2.0f * clipPlane[0] * clipPlane[0]);
	m[0][1] = (-2.0f * clipPlane[0] * clipPlane[1]);
	m[0][2] = (-2.0f * clipPlane[0] * clipPlane[2]);
	m[0][3] = (-2.0f * clipPlane[3] * clipPlane[0]);

	m[1][0] = (-2.0f * clipPlane[1] * clipPlane[0]);
	m[1][1] = (1.0f - 2.0f * clipPlane[1] * clipPlane[1]);
	m[1][2] = (-2.0f * clipPlane[1] * clipPlane[2]);
	m[1][3] = (-2.0f * clipPlane[3] * clipPlane[1]);

	m[2][0] = (-2.0f * clipPlane[2] * clipPlane[0]);
	m[2][1] = (-2.0f * clipPlane[2] * clipPlane[1]);
	m[2][2] = (1.0f - 2.0f * clipPlane[2] * clipPlane[2]);
	m[2][3] = (-2.0f * clipPlane[3] * clipPlane[2]);

	m[3][0] = 0.0f;
	m[3][1] = 0.0f;
	m[3][2] = 0.0f;
	m[3][3] = 1.0f;
	return m;
}
```

With this reflection matrix, you can then apply this transform to the view matrix of your scene's main camera to apply the mirroring effect with respect to that specific camera. At this point we have the proper view and projection matrix to use for the reflection rendering. We decide to go ahead and test out rendering reflections. We set a reflection camera on the floor and point it straight up to the sky. Our mirror will be the floor and our main camera is looking straight ahead into the scene and perpendicular to the floor. So we get a result like the image below which is the gbuffer storing the normals for that reflection camera. 

![Bad Normals](/images/reflections-using-oblique-projections/NormalReflectionBad.png)

I chose this render target as it demonstrates the current problem pretty nicely. If you look at the models you can see that it looks like the front faces are being culled instead of the back faces which is what you want most of the time. What is happening here is that the reflection matrix inverts things in such a way that what would normally be considered front facing triangles become back facing instead. If you use the same cull mode convention as you do when rendering the scene from your main camera then you end up with a result like the image above. For comparison the stormtrooper model is actually not using any culling which is why it is unaffected in this case. That said, we definitely don't want to turn off culling for the entire scene rendering when we do reflections as that would be costly. What the solution ends up being is to flip your culling mode tests when rendering your reflections. So what typically uses back face culling would then use front face culling when rendering reflections. With that fixed we get proper normals as you can see below.

![Good Normals](/images/reflections-using-oblique-projections/NormalReflectionGood.png)

## Rendering Layers
One major thing that is key to having this all work correctly is the renderer supporting rendering layers at a per camera level. This allows culling certain models when rendered by a specific camera. If you consider the example above where the reflection camera is at the floor and the scene does have a floor mesh then normally that floor mesh would be blocking everything else in the scene. Then your reflection texture is the floor or parts of it if some things get culled during rasterization. By setting up rendering layers you can have the reflection camera skip rendering the floor so the reflection texture contains everything you would expect. You can take it a step further and exclude other models during the reflection pass in order to save on performance.

This can be implemented with something as simple as a bit mask. In the engine, anything that can be rendered has a **uint32** to store what layers it is in. Any entity that can render the scene also has its own bit mask. Then when iterating through the draw items, you can do a simple **logical and** operation between these two unsigned integers to determine if the renderer should submit a draw call.

## Sampling Reflections
Now that we have a proper reflection texture, we still need to have a way to bind that to the correct material so that we can use it in our main scene rendering. This is solved by the planar reflections component serializing a list of material assets that will want to use the reflection texture during normal scene rendering. This list is configured in the editor by the user so it is easy to use and should be fine most of the time as there likely won't be many material assets to configure in this way.

At the time of this writing the reflection texture is setup as an emissive texture for the material being rendered. The material constant buffer also has a bit flag that represents when it needs to use this emissive texture input as the reflections texture. When this flag is set the sampling behavior of the emissive texture is tweaked as the shaders use the screen position of the fragment to sample the reflection texture instead of the model's underlying uv coordinates.

## Performance
Tempest offers a few configurable features to help manage the performance impact from these types on entities. 

Firstly you can configure the resolution used for the reflection texture you render. To minimize bandwidth, the reflection texture can choose to use the light HDR floating render target that uses 32 bits per pixel or if accuracy is needed it can use 16 bits per channel RGBA floating point render target but at the cost of doubling the bits per pixel. 

You don't always want to or need to render reflections every frame. For those situations you can specify the update frequency that the engine will render the reflections. It defaults to rendering every frame but it can be set to something like every other frame or updated only when scripts tell it to refresh. This becomes very powerful as suddenly you can make a scheduler to manage these if you happen to have many of them in a scene.

Another useful feature is that you can turn off certain rendering features like shadows or fog to improve performance at the cost of visual fidelity. 

Lastly, I haven't seen anyone do this before but the reflections texture can be rendered using interlaced rendering. A [previous post](https://yggdrasil-917.github.io/posts/interlaced-rendering/interlaced-rendering/) outlined the interlaced rendering solution used in Tempest. There is no reason it couldn't be used to render secondary views and reap the performance benefits it introduces. Furthermore, you often use these reflection textures with other distorting effects so many quality issues become harder to see. For example, water reflections typically distort the reflections so aliasing and other artifacts are harder to see in these reflection textures. The example above with the gbuffer normals was rendered using interlaced rendering, which is why the aspect ratio looks a little weird and that's because their width is smaller than the image's height resolution. Right before the reflection rendering is finished a deinterlace pass is done to restore to the final width resolution.

## Limitations
After all these good things, there are still some issues that need to be handled carefully or at least design around them. First thing is the obvious issue of increasing your draw call count as the planar reflection component renders the scene again. It is important to note that it only goes so far as to render the environment but skips any unnecessary work like post processing. 

Another thing is you want to avoid designing levels that could potentially have reflections within reflections as that kind of recursive rendering can quickly become a performance problem and will have rendering artifacts as some reflections won't have the reflections from other surfaces.

The last limitation is more of a limitation with my current implementation. Reflection texture resolution cannot go higher than what the main scene is rendered at. This is mostly due to memory considerations and the fact that the renderer does not look at the resolution of these reflection textures when rendering the scene to decide if it needs to resize the underlying render targets used for the rendering.

## Example
Alright it is long overdue to show at least one screenshot of it in practice so lets go ahead and do that. Below is a screenshot of the test scene that generated the gbuffer screenshot from earlier in the article. The floor acts as the reflective surface so the planar reflection camera is rendering the environment from around that position and is looking up in the sky. Since this is not screenspace all the elements in the reflection will remain there despite not being visible in the main camera. An added bonus is that transparency does not require special handling so things like particles would also just work out of the box.

![Example](/images/reflections-using-oblique-projections/PlanaerReflectionExample.png)

And here is the reflection texture rendered by the planar reflection camera. This is what will get sampled using the screen position when the floor is rendered

![Reflection Texture](/images/reflections-using-oblique-projections/ReflectionTexture.png)