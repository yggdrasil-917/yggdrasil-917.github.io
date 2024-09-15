---
title: "Lens Flare"
date: 2024-09-14T11:34:55-07:00
draft: true
toc: false
images:
tags: 
  - rendering
---


![LensFlareExample](/images/lens-flare/LensFlareExample.png "Lens Flare Example")


* [Overview](#overview)
* [Scriptable Object Asset](#scriptable-object-asset)
* [Lens Flare Profile](#lens-flare-profile)
* [Lens Flare Occlusion](#lens-flare-occlusion)
* [Rendering](#rendering)
* [Lens Flare Atlas](#lens-flare-atlas)


## Overview
A lens flare is a phenomenon caused by the lens of an eye or camera when looking at very bright lights. In the real world the shapes produced by a lens flare are determined by many factors such as the shape of the lens. The different hardware configurations with cameras is one reason lens flare seen by the human eye are pretty different from lens flares created by a camera lens. With camera lenses you can typically see hexagonal shapes or circular ghosts across the viewport unlike with the human eye.

The goal for Tempest was to develop a flexible lens flare rendering system that could be used to emulate the types of lens flares you see coming from both cameras and human eyes. It goes without saying but this system has to be efficient as well as it makes no sense to spend too much of your frame budget on this feature. To that end, a fast and efficient data-driven sprite based rendering method was used to implement lens flares in Tempest unlike other methods that rely on screen space procedural techniques as those tend to be slower and harder to configure its look.

## Scriptable Object Asset
First though, lets discuss what was used to make lens flare rendering data driven as I have yet to discuss what a scriptable object asset is within the context of Tempest. The scriptable object asset is a data only container where its serialized content can be customized from c++. This is achieved through a abstract base class that defines an interface to adhere to. For those familiar with scripting in Unity this concept should be familiar as the idea was pretty much lifted from that engine.

One big benefit of using an asset to store data is that it is memory efficient. Since the asset manager will never load the same asset multiple times you can be confident that you will only pay the memory cost for only one instance. Then any game entity seeking to access the same asset will get a reference to the already loaded asset. Finally, being loaded by the asset manager the scriptable object's lifetime is also managed by the asset manager which makes using scriptable objects very easy to use.

Here is what the scriptable object base class looks like.
```cpp
class TE_ECS_API ScriptableObject
{
protected:
    bool isValid = false;

    void CopyBaseMembers(ScriptableObject* other)
    {
        isValid = other->isValid;
    }

public:
    virtual ~ScriptableObject() { isValid = false; }

    virtual ScriptableObject* Clone() { return nullptr; }

    virtual void Copy(ScriptableObject* /*other*/) {}

    virtual bool Equals(ScriptableObject* /*other*/) { return false; }

    // This is useful when comparing against other instances of a scriptable object
    virtual uint64 GetContentHash() const { return 0; }

    // This gets called after deserialization is done
    virtual void OnInitialize() { isValid = true; }

    // This is usually implemented as a hash using the class name and used with a factory pattern for creation
    virtual uint64 GetTypeHash() { return 0; }

    // This is mainly used for creating editor menus
    virtual const char* GetTypeName() const { return "ScriptableObject"; }

    // This can be used to draw the ImGui UI in the editor
    // Returns true if object was modified, false otherwise
    virtual bool DrawEditorUI(const struct EditorHelperData& /*data*/) { return false; }

    // Describe how to serialize the asset
    virtual void PupAsset(class Pupper* /*p*/) {}

    template <class T>
    static T* Instantiate(T* original)
    {
        return original->Clone();
    }

    static ScriptableObject* Instantiate(ScriptableObject* original)
    {
        return original->Clone();
    }
};
```

## Lens Flare Profile
With scriptable objects discussed we can now move on talk about the lens flare profile class which itself derives from the scriptable object base class to store lens flare specific configuration data.

```cpp
struct LensFlareElement
{
    // config data here
};

class LensFlareProfile : public ScriptableObject
{
protected:
	uint8 structVersion = 1; // for serialization purposes

	DECLARE_SO_REGISTRANT(LensFlareProfile);

public:

	LensFlareProfile* Clone() override;
	void Copy(ScriptableObject* other) override;
	bool Equals(ScriptableObject* other) override;
	const char* GetTypeName() const override { return "LensFlareProfile"; }

	bool DrawEditorUI(const struct EditorHelperData& data) override;

	void PupAsset(class Pupper* p) override;

	Vector<LensFlareElement> elements;
};
```

As you can see, it is very brief like most other classes derived from the scriptable object base class. It generally boils down to implementing functions use for cloning, copying, and comparison. Then we can also specify how to serialize the data and how to draw the UI in the editor for easy editing of each of the properties inside the class. In theory, a lot of this could be automated if a robust reflection system was available in c++ but for now all of that is done manually.

In the sample code you will notice a struct called ```LensFlareElement``` and its responsibility is to store every possible setting specific to an individual lens flare. This can be anything from color properties, shape properties, and transform properties. The lens flare profile can keep track of any number of those lens flare elements to create intricate and complex looking lens flares. The engine uses a component system similar to what you find in engines like Unity and in that fashion Tempest has a lens flare component that can be added to any entity in a scene, regardless of whether it is a light source or not. It is this component that stores a reference to a user specified lens flare profile asset and in this way it is very easy for multiple entities in a scene with a lens flare component to share the same looking lens flare. On top of that the component itself also stores a few additional global parameters that can affect some of the local settings specified by the lens flare profile. This includes things a global intensity parameter to give an example.

Lets expand a little on some of the supported parameters available in a lens flare element. For color properties, the lens flare element exposes settings to control its local intensity. This allows making some of the individual lens flare elements brighter or dimmer than the rest. We are also able to tint them by a user specified color or if the lens flare component is attached to a light source then we can automatically tint it based on the color of the light source. You can see this in effect with the screenshot at the top of the article as the lens flare is being tinted by the color of the directional light source. Other color related features are under consideration but no plans to implement them yet. An example of such a feature is specifying an animation curve to change the color of a lens flare element based on how far away it is from the source. This would allow creating more interesting color transitions to the lens flare.

In regards to transform properties for lens flare elements, the system allows you to do many different kinds of positional transforms and distortions. You can easily do all the common forms of transforms to the lens flare elements so that means you can offset positions, scale them, and even rotate them. There are also some procedural functions that can introduce some amount of distortion to the lens flare element so we can make them look a bit more complex.

Finally, the system support various kinds of lens flare shapes that can be chosen from a dropdown in the editor. Using SDF functions we can render circular or polygonal shapes like hexagons. We can actually render a variable number edges in a polygon so we can go from a simple triangle to something that has too many edges making it look like a circle but of course in the latter case we offer a faster alternative in the form of a circle SDF. There are properties that can control the falloff or hardness of the SDF function used to create those shapes and we can even invert the SDF function results allowing the creation of halo like lens flare elements as well.

The final type of lens flare shape supported is one that is determined by a grayscale texture. This fills in the gaps that the circle and polygonal shapes can't create performantly. This type however, required a lot more work to properly support it alongside the other types and the details about it will be discussed later in the article.

## Lens Flare Occlusion
Supporting lens flare occlusion is an important feature to have and implementations have varied over the years. For a long time this feature used to be done with hardware occlusion queries but these days there are better ways. In Tempest, lens flare occlusion is done by stochastically sampling the scene depth texture multiple times to create an average visibility value. Among the configurable parameters inside the lens flare component is the ability to set the search radius in screen space units. The source of the lens flare is treated as a disk in screen space that will be randomly sampled with the sampled location doubling as texture coordinates to look up the scene depth. That scene depth is compared with the screen space depth of where the lens flare source is located. If the scene depth is closer to the camera than the lens flare source then we know that sample is occluded otherwise it is visible and gets accumulated to the final visibility average. This computation can be optionally skipped if the lens flare component is setup to do not do the occlusion pass for that instance.

Setting up the occlusion rendering pass is fairly simple. Tempest uses a texture 2D array to store the visibility results for each lens flare that will be rendered in a frame. The width and height of each slice is only one pixel and the number of slices in the array is set to the max number of lens flares Tempest can render in one frame. At the time of this writing that max number is set to 128 so the final dimensions of the texture 2D array is 1x1x128 with a format of R32 floating point so memory footprint is very small. Also the fact that we are only rasterizing one pixel to compute the visibility results make this very fast even while using a high number of stochastic samples to lookup the scene depth.

## Rendering
In order to render the lens flares, Tempest keeps track of a GPU constant buffer that contains the required data to instance render each lens flare visible in the frame. So in one draw call we can render all the lens flares. This means Tempest does one draw call to render all lens flares to create the visibility texture 2D array and one more draw call to render all the flares into the scene color render target. Each packed instance of a lens flare uses up 112 bytes with 128 total lens flares means the constant buffer is around 14KB in size. You can see what the packed struct looks like below and the constant buffer just has an array of those.

```cpp
struct LensFlareData
{
    float4 tint;
    float4 data0; // x: localCos0, y: localSin0, zw: PositionOffsetXY
    float4 data1; // x: OcclusionRadius, y: OcclusionSampleCount, z: ScreenPosZ, w: ScreenRatio
    float4 data2; // xy: ScreenPos, zw: FlareSize
    float4 data3; // x: Allow Offscreen, y: Edge Offset, z: Falloff, w: invSideCount
    float4 data4; // x: SDF Roundness, y: Poly Radius, z: PolyParam0, w: PolyParam1
    float4 data5; // x: occlusion index, y: LFEF flags, z: flare type
};
```

Hadn't mentioned it yet but the draw calls to render these lens flares do not use any sort of vertex or index buffer for each sprite. It instead uses a procedural draw call where the number of instances and number of verts are specified. In out case the number of verts is set to 4 as we want quads and the number of instances is set to the number of lens flare instances packed into the constant buffer. The vertex shader takes care of computing the vertex position based on the vertex ID before being transformed by the lens flare specific transforms and we use the instance ID to lookup the correct lens flare data for the instance being drawn. This is all achieved using ```SV_VertexID``` and ```SV_InstanceID``` inputs to the vertex shader. The vertex position is computed with the sample function below using the vertex ID as input.

```hlsl
float2 GetQuadVertexPosition(uint vertexId)
{
    float u = vertexId & 1;
    float v = (vertexId >> 1) & 1;
    
    return float2(u * 2 - 1, 1 - v * 2);
}
```

Another important detail when rendering into the texture 2D array for occlusion information is the fact the vertex shader needs to output the index of the slice in the array the draw is rendering into. This is done by outputting a result to ```SV_RenderTargetArrayIndex``` which just so happens to be the same value stored in ```data5.x``` so this becomes trivial to do.

## Lens Flare Atlas
There is one major feature I haven't talked about much yet and that is supporting lens flares that sample a user specified texture for the flare's shape. I wanted to support this feature while also still being able to render all lens flares in one draw call which is not the simplest thing to do when you want to allow user specified textures of any reasonable size. At first I had considered using a texture 2D array to store all these textures but the problem with that is that each slice of the array has to have the same resolution and that's a deal breaker. The only other option left is to dynamically pack those individual textures into one big atlas. To make this work I would also need to provide the information on where to sample that atlas to retrieve the correct texture for any given lens flare. In this case, we alias the memory of ```data4``` from the ```LensFlareData``` struct to store the scale and offset to properly sample the tile inside the atlas texture. This is safe to do since that variable is only used when the lens flare shape is set to be a procedural polygon type.

The first problem to solve is packing the textures of any resolution into the atlas. As far as I know that is an [NP-Complete](https://en.wikipedia.org/wiki/NP-completeness) problem which tends to mean there is no theoretically optimal solution. Though considering this is a fairly common problem in graphics there are several algorithms out there with simple to complex heuristics that can help solve the problem quickly. In my case, I do not need the best but good enough and fast enough will do. In that spirit, Tempest uses a greedy algorithm that sorts the textures to be packed based on area and starts adding the biggest textures into the atlas first and makes its way down to the smallest textures as long as they fit in the atlas. This does not produce the most efficient use of the space in the Atlas but it is very fast and simple to implement. It's also worth noting the atlas is allocated using the render target pool allocation system so it is treated as a transient texture. If no lens flares need to use this atlas then the texture is not allocated and if it was allocated previously then it will eventually be deallocated if nothing has requested to use it for several frames.

Here is an example of what the lens flare atlas looks like after packing two user specified grayscale textures.

![LensFlareAtlas](/images/lens-flare/FlareAtlas.png "Atlas")

Here is what the lens flares look like using the atlas.

![TexturedLensFlares](/images/lens-flare/TexturedFlares.png "Textured Flares")