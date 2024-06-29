---
title: "Asset System"
date: 2024-06-26T08:40:45-07:00
draft: false
toc: false
images:
tags: 
  - engine
  - tools
  - assets
---

* [Assets](#assets)
* [AssetLifetime](#asset-lifetime)
* [AssetRegistry](#asset-registry)
* [Cooking](#cooking)
* [CookOnTheFly](#cook-on-the-fly)
* [OfflineCooking](#offline-cooking)


## Assets
The Tempest engine, like most other engines out there, has a concept of an asset along with an asset manager that can handle asset lifetimes as well as asset cooking. The engine contains a base class that all asset types derive from. A simplified version of that Asset base class is below.

```cpp
struct AssetHandle
{
  // This contains the unique path to the asset, stored as a string 
  // and a hash value of that path evaluated at compile time using constexpr.
  StringHash id;
  
  operator uint64() const { return id.GetHash(); }
  operator const char*() const { return id.ToString(); }
  operator fs::path() const { return id.ToString(); }
  operator uint64() { return id.GetHash(); }
  operator const char*() { return id.ToString(); }
};

class Asset
{
  protected:
    AssetHandle handle;
    uint64 typeId; // Hashed value based on the type of asset
    // some more things
  
  public:
    // helper functions
    const AssetHandle& GetHandle() const { return handle; }
    uint64 AssetId() const { return handle; }
    const char* AssetPath() const { return handle; }
};
```

Let's talk a little bit about the AssetHandle struct. This is the entryway to loading assets in the Tempest engine. They are essentially a wrapper aroung a hashed string with some helper methods for ease of use with certain APIs. Asset handles support serialization so when they get deserialized we can tell the asset manager to load the asset represented by the asset handle. This allows game scripts to save asset references so for example, a game script could have an asset handle that points to a particle system asset. When the script gets initialized it will also tell the asset manager to load the asset represented by the handle. It is important to note that the asset manager will not load the same asset twice but instead checks an internal cache for loaded assets and returns early if it has been loaded before otherwise the asset is loaded and the cache is updated. There is a slight caveat to that behavior that will be explained later when talking about asset lifetimes. 

By default assets get loaded synchronously but there is support for async loading. The reason async loading is not the default is due to certain asset types lacking threading support. The engine was upgraded to c++20 recently which gives us access to coroutines so that could be a possible extension to synchronous asset loading in the future that might be worth exploring. Something worth mentioning is that all engine specific source assets are stored in plain text as this makes it very easy to edit outside of the editor and makes diffing with your version control software of choice very easy. They are however stored in binary form after they are cooked.

Based on a recommendation found in the Game Engine Architecture book by Gregory, the string hash function used is FNV-1a. It's a pretty simple algorithm to implement with a really low collision rate so it should be rare the times a different string produces the same hash value as another string. That said it is still a good idea to have debug code check for collisions in the editor. If there are collisions the solution is just to rename the string that produced the collision.

```cpp
// Sample FNV-1a implementation
#define FNV_OFFSET 2166136261u
#define FNV_PRIME 16777619u

constexpr size_t CalculateFNV(const char* str)
{
  constexpr size_t prime = FNV_PRIME;
  size_t hash = FNV_OFFSET;
  while (*str != 0)
  {
    hash = (hash ^ (*str)) * prime;
    ++str;
  }
  
  return (hash ^ (*str)) * prime;
}
```

## Asset Lifetime
The Tempest engine doesn't have a garbage collection system like what you would find in Unreal Engine. Due to the design of the game being made with this engine, it is very clear what asset lifetimes will be so a complex garbage collection system is not needed. The design pretty much ties the asset lifetime to the lifetime of a level. With that knowledge we can assume that when a level is unloaded it is safe to delete any and all assets loaded for that level. For the sake of simplicity in the design of this asset lifetime management system, the engine does not try to be smart and check what sort of assets the level we are transitioning to has so that the engine may skip loading assets it has already loaded into memory. In practice this means we reload an asset that was already loaded in the previous level but it hasn't been a source of performance issues as level transitions are still very fast. This is something that may need to be considered in the future if level complexity gets high enough that load times suffer because of this.

So how does the asset manager keep loaded assets internally? Well it keeps a nested UnorderedMap similar to the one below. An UnorderedMap is the same as std::unordered_map but using custom allocators.

```cpp
using AssetPtr = std::unique_ptr<Asset>;
UnorderedMap<uint64, UnorderedMap<uint64, AssetPtr>> loadedAssets;
```

Each level loaded into memory keeps a map of the assets loaded for that specific level. The outer map uses a key of type uint64 that represents the hash of a level and the value is the map of loaded assets for that level. The inner map uses a uint64 type for a key as well which is the asset's hashed id and the values are unique pointers to the asset. Using smart pointers for the assets lets of release all the memory used by those assets once we clear the map, typically done during a level transition. This way of managing assets does lead to the potential of duplicating assets in memory if multiple levels load the same asset but so far in practice it hasn't been a source of concern.

This lifetime system has a couple of "special" kinds of levels that are useful for some unique purposes. One such level is referred to as the permanent scene id. This is useful for loading global assets, in other words, assets the game may need at any time while the game is running or if you want to preload certain common assets. Such assets are things like UI textures and fonts. It is much better to load these when the game starts instead of reloading the assets any time a UI menu is invoked. These global assets then get released when game exits so from a asset lifetime perspective, these assets are always safe to invoke while the game is running. Another special level type is for transient assets. These are assets that need to be loaded for a temporary task and you want to free the memory afterwards.

It is important to note that the asset manager returns a struct called AssetRef instead of a pointer to the asset being loaded. Then when the actual asset is needed that asset ref object can be used to get a temporary pointer to the asset. If the pointer is not null then we have a valid asset and can do whatever we need. The asset ref struct is simply two uint64 member variables. One for the level's hash that owns the asset and the other is the asset hash itself. The main reason for using this kind of indirection is so that the asset manager can internally do modifications to the asset without invaliding pointers to it. For example, hot reloading an asset will create a new asset pointer so the previous one becomes a dangling pointer but this is not a problem with the current level of indirection.

## Asset Registry
The asset registry is essentially one large map of cooked assets that is created and managed by the editor and the cooker. The map stores asset metadata that can be useful for editor features like moving a file to another location within the content system. Whenever an asset is cooked, metadata associated with that asset gets added to the registry and later serialized. In the beginning this registry had more uses outside of the editor but they have been slowly phased out as they've seen little use and instead is more an editor only feature these days. The metadata can include things like GUIDs, file paths, and other useful bits of info. The registry itself is stored in plain text for development purposes but can also be stored in binary form.

## Cooking
Like a lot of other engines, Tempest engine will ingest different kinds of assets authored through DCC tools and convert them to a custom engine specific binary format. The engine takes an approach similar to what Unity does when it imports an asset. Tempest has more or less a custom vtable for all the import functions required to support every type of asset in the engine. So part of the process in supporting a specific type of asset is to map an asset extension to an import function. So for example, a jpeg asset will invoke the importer function for a texture asset. When the import is done the cooked asset is stored in a folder for cooked assets for a given platform. However, the source asset is left unmodified in the content folder making it easy to do updates to the source asset. If no valid asset extension to import function is found then the asset is ignored and it is logged that it was skipped. It's also worth pointing out that some types of assets also create a metafile alongside the source asset similar to what Unity does. As of now only texture assets do this in order to store texture settings in that metafile.

Cooking is only allowed to happen in the editor or when the offline cooker is running. The player is only allowed to read cooked data and will error out if it tries to cook an asset it does not have the data for. There are technically two cooking modes, cook on the fly and offline cooking. Both will be expanded on later in this article.

## Cook on the fly
This is the mode the editor uses and what this means is that it will only cook an asset when a request to load that asset occurs. When loading an asset, the asset manager checks to see if there is a cooked version of that source asset. If there isn't then it proceeds to import the asset otherwise, it will check if the source asset has been updated since the last time it was cooked. If it has been updated since then it will get cooked. This is where the std::filesystem API comes in handy to check the last modification timestamp of both the source asset file and the cooked binary file. So if the last modification timestamp for the source asset file is newer then we know we should cook it. After cooking is done the asset registry is updated to include the new asset.

## Offline Cooking
The offline cooker is what the build pipeline uses to cook the assets. The cooker takes as input a list of levels to include in the build. It can also accept a list of assets to always cook regardless of whether or not they are referenced in a level. This is to accommodate the case where assets are loaded from scripts instead of being in the level's file. By default the offline cooker will use incremental cooking where it operates much like the cook on the fly method so only dirty assets get updated. It can also do a full recook of all the assets to make sure everything is up to date and this is the preferred method when running the build pipeline. How this tool figures out what to cook is fairly simple. Generally speaking, the assets are stored in text format and are only in binary after being cooked. Level assets follow this pattern. Any time the level has a reference to an asset it will have serialized data for an asset handle in text. So the tool opens a file handle to the level's text file and scans for those tokens. By the end it will have a list of all referenced assets in that level. It does the same for all levels that will be included in the build so it ends up with all statically referenced assets. The tool then adds the assets that should always be included in the build and finally we can start the actual cooking process. Once the cooking is done there is some diagnostic data reported at the end where things like number of assets cooked is reported, total disk memory used by the assets, total disk space used by asset type, etc.