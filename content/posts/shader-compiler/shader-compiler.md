---
title: "Shader Compiler"
date: 2024-06-29T09:27:15-07:00
draft: false
toc: false
images:
tags: 
  - shaders
  - tools
---

* [Editor Integration](#editor-integration)
* [Shader Cache](#shader-cache)
* [Hot Reloading](#hot-reloading)
* [Tempest Shader](#tempest-shader)

These days every game engine out there is going to need to compile shaders but how they go about it tends to be somewhat different depending on the engine's needs. At the time of this writing Tempest only supports the Windows platform and supports DirectX 12 and Vulkan. The compiler uses the [dxc executable](https://github.com/microsoft/DirectXShaderCompiler) under the hood to compile for both DX12 and Vulkan.

The Tempest shader compiler is a separate executable similar to what other engines do where the editor will automatically launch it as a child process. When the shader compiler is launched this way it uses inter process communication (IPC) through local sockets to send information to the editor and listen to packets from the editor. It can also run in standalone mode where it compiles all dirty shaders or optionally recompiles everything in the engine. This is the mode the build pipeline uses.

## Editor Integration
One important thing to mention about the Tempest engine tool ecosystem is that for tools that need to communicate with other Tempest tools requires the use of IPC. This is achieved through a daemon that serves as a hub of sorts for interpreting packet types that come in and internally figures out what data to send and to what tools to send that data to. This is referred to as the Tempest Daemon. This setup is currently only used with the editor as the player does not need to talk to any other applications at the moment. On editor startup, it will configure the IPC setup by first starting up the daemon and will wait until it establishes connection with it before then starting the shader compiler. The shader compiler gets launched but not in standalone mode. It instead will be required to connect to the daemon as well before the compiler proceeds to compile any dirty shaders. Once the compiler is done an acknowledgement packet is sent to the editor if everything finished successfully otherwise, it should exit due to engine level shader errors. To finish setting up IPC, the editor spawns a new thread in order to handle all the network related IPC tasks and not block the main thread while doing so. This thread will keep running until the editor is closed. 

Another key thing the editor will do is start a thread to handle file watching for any changes. This system is used for several things but one of them is to know when a shader file is modified while the editor is running. As soon as the thread detects a change to a shader file a network packet representing what file was changed is prepared and marshalled off to the shader compiler. The packet boils down to a string containing several tokens delimited by a semicolon. Below is the function used internally to prepare the network packet with the required information for the shader compiler to handle the compilation job. It also shows the function the shader compiler uses to send a packet to the editor when it successfully compiles a shader.

```cpp
void Packet::PrepareShaderCompilationJobPacket(const char* networkAddress, const char* filepath, const char* shaderStage, const char* entryPoint, const char* defines, const char* api)
{
  data.type = PacketType::BroadcastTo;
  char payload[Packet::maxMessageSize] = {};
  const int32 charsWritten = strlen(defines) == 0 ? snprintf(payload, sizeof(payload), "%s;%s;%s;%s", filepath, api, shaderStage, entryPoint)
                                                  : snprintf(payload, sizeof(payload), "%s;%s;%s;%s;%s", filepath, api, shaderStage, entryPoint, defines);
  if (charsWritten < 0)
    Warnf("Local payload buffer is too small to fit all shader compilation options");
  
  strcpy(data.u.broadcastPacket.ip, networkAddress);
  strcpy(data.u.broadcastPacket.msg, payload);
}

// Used when all shader variant compilation succeeds
void Packet::PrepareShaderCompilationResultPacket(const char* networkAddress, const String& result)
{
  data.type = PacketType::BroadcastTo;
  char payload[Packet::maxMessageSize]{};
  const int32 charsWritten = snprintf(payload, sizeof(payload), "%s", result.c_str());
  if (charsWritten < 0)
    Warnf("Local payload buffer is too small to fit all shader compilation result");
  Debugf("CompilationResultPacket sending '%s' to '%s'", payload, networkAddress);
  
  strcpy(data.u.broadcastPacket.ip, networkAddress);
  strcpy(data.u.broadcastPacket.msg, payload);
}
```

## Shader Cache
The engine has support for a shader cache that gets shipped with the executable. Only the editor and the shader compiler are able to create the shader cache. The cache itself is composed of multiple individual files instead of one large file containing all compiled shaders. This is mainly done to keep shader loading code simple but the more optimal approach is certainly to coalesce all shader caches into one large file. Each file that forms the cache is essentially binary blobs containing identifying information for all the shader variants in the blob as well as the bytecode generated from dxc for each shader variant. The shader cache's data is specific to the rendering backend API it belongs to so for example, there is one cache for DX12 and a different one for Vulkan.

```cpp
// Sample code to pack shader variants into one binary blob after compilation succeeds
for (const std::pair<const String, Vector<BlobEntry>>& it : g_ShaderBlobs)
{
  fs::path outputPath = it.first; 
  FileHandle handle = File::Open(outputPath.generic_string().c_str(), FileMode::FM_Write, true);
  if (!handle)
  {
    Errorf("Error writing permutation shader file");
    continue;
  }
  defer(File::Close(handle););
  const char* signature = "NVSP"; // 4 byte magic number used by NVRHI
  File::Write(tag, 1, 4, handle);
  
  const auto& entries = g_ShaderBlobs[it.first];
  for (auto& entry : entries)
  {
    String inputFilename = entry.compiledPermutationFile.generic_string().c_str();
    BinaryBlob inputFileData;
    if (!File::Load(inputFilename.c_str(), inputFileData))
    {
      Errorf("Failed to open file %s", inputFilename.c_str());
      return;
    }
    
    ShaderBlobEntry binaryEntry;
    binaryEntry.permutationSize = (uint32)entry.permutation.size();
    binaryEntry.dataSize = inputFileData.size;
    
    File::Write(&binaryEntry, 1, sizeof(binaryEntry), handle);
    File::Write(entry.permutation.data(), 1, entry.permutation.size(), handle);
    File::Write(inputFileData.data, 1, inputFileData.size, handle);
  }
}

// Function signature for loading shader from the cache
ShaderHandle ShaderFactory::CreateShader(const char* fileName
                                       , const char* entryName
                                       , const Vector<ShaderMacro>& defines
                                       , nvrhi::ShaderType shaderType
                                       , bool forceCreate);
```

As noted above with the CreateShader function, in order to create a shader we need the obvious things like filename and shader type but we also need any of the defines used to compile the shader. This is so we can get the correct shader variant from the compiled shader blob as it contains all the shader variants. For now we do a simple linear search inside that binary blob to find the correct shader permutation. It's simple and works well but it is definitely not a scalable solution given the problem with shader permutation explosion. If it ever does become a performance problem then a better solution is likely to add some sort of table of content built into the binary blob so searches can be done a lot quicker. 

The game being worked on with this engine is attempting to make a variety of shaders using dynamic branching instead of the usual static branching which creates a new shader variant for each static branch. The main benefit of limiting the use of static branching is that it keeps the number of shader permutations low which in turn keeps build times very low as these days the bottleneck in a build is usually compiling all the shader permutations in the game. Downside of course, is individual shader performance can suffer but it is important to know that there are different kinds of dynamic branching with their performance impact. The shaders in Tempest tend to use dynamic branching based on a value passed to the shader from a constant buffer which means all threads take the same branch during shader execution. Branch performance has certainly gotten better over the years so people should not be afraid of using them but it goes without saying that profiling should also be done to make sure no serious issue is introduced.

## Hot Reloading
Every engine should make it a priority to support hot reloading of assets and shaders are no different. It makes iteration so much better and you'll be all the happier seeing your changes take effect quickly. In Tempest, there are two extensions for shader files. One is .hlsli for include files and the other is .hlsl which is the shader file with the entry points for whatever shader type it is. Also worth pointing that a hlsl file can have a vertex shader and a fragment shader in the same file and still compile correctly. I point this setup out because of how hot reloading is currently implemented in the engine. When the editor is running and a modification is done to a hlsl file the hot reloading logic will kick in. The editor sends a packet to the shader compiler so it takes care of compiling all the shader variants for that file. If all the shaders compiled successfully then the shader compiler will send a packet to the editor telling it that it can attempt to hot reload the shader file path contained in the packet. If even one shader variant failed to compile then the editor is never notified to hot reload the shader so things continue on unchanged until another edit is made to the shader. The compile errors are logged in the console as well as a log file for later viewing so you can see what file failed and where it failed. 

The hot reloading logic is not executed if a hlsli file is edited. This is mainly due to not being implemented at the moment. To properly do this the shader compiler would need to store some sort of dependency graph somewhere so it can find what hlsl files depend on that modified header file and force a recompile. I also do like not having this feature as during development I mainly want to see the changes on the hlsl file I'm actively developing instead of having any shaders dependent on that header file recompile when they aren't needed. There are some header changes that are too invasive that I can't follow this workflow and have to recompile most shaders to ensure everything is working but those are few and far between. Think for example, changing the layout or size of a constant buffer. After I'm happy with the changes to a hlsli file I always make sure to recompile all the shaders in the engine to ensure I didn't miss anything. This is as simple as deleting the shader cache folder and relaunching the editor or run the shader compiler in standalone mode. Compilation is multi threaded with the game having around 1000 shaders or so last I looked so the shader compiler gets through all compilation tasks in less than 10 seconds on my laptop. That includes building the shader cache for DirectX 12 and Vulkan.

## Tempest Shader
Let's go over a simple shader example. Below is a blit shader from the Tempest engine that can support 2D texture or texture2DArray as input
```cpp
// Simple copy shader
// $fragment_variants TEXTURE_ARRAY

#if TEXTURE_ARRAY
Texture2DArray tex : register(t0);
#else
Texture2D tex : register(t0);
#endif
SamplerState samp : register(s0);

void MainPS(
    in float4 pos : SV_Position,
    in float2 uv : TEXCOORD0,
    out float4 o_rgba : SV_Target)
{
#if TEXTURE_ARRAY
    uint layer = 0; // only need the first layer as of now
    o_rgba = tex.Sample(samp, float3(uv, layer));
#else
    o_rgba = tex.Sample(samp, uv);
#endif
}
```

As you can see at the very top of the shader there is a comment line that starts with a $ followed by a keyword and a value. For anyone familiar with hlsl authoring in Unity, that line should make sense. For those unfamiliar with it, this line specifically tells the shader compiler that this shader file has two variants, one with TEXTURE_ARRAY defined and another one without it. You'll also notice that it is specifying that only the fragment shader has the variant so the other shader stages will not be affected by this shader variant. This keeps shader compilation fast and keeps memory to a minimum. At this moment the compiler supports four of these special pragma statements that I will outline below.
* **vertex_variants** - Only the vertex shader produces variants specified by the list of values preceding this keyword
* **fragment_variants** - Only the fragment shader produces variants specified by the list of values preceding this keyword
* **variants** - All shader stages or compute shader will produce variants specified by the list of values preceding this keyword
* **with_debug_symbols** - This tells the compiler to include debug symbols for the shader file being compiled

A limitation of sorts is that the entry point function for each shader stage has to have a certain name in order for the shader compiler to find. For instance, a vertex shader needs to have a MainVS as the entry point function, for fragment shader it's MainPS, and for a compute shader it is MainCS. This is again to keep the code simple and it has worked out fine so far so this will likely not change anytime soon. Shaders can also output to multiple render targets without any issue as long as they follow the requirements needed for MRT. There's a few render passes in the engine that uses that but I will cover that in a later post. For now just know that the engine uses deferred rendering so the gbuffer pass is one such pass that uses MRT for its output. 

Another limitation of the compiler is that it does not do any reflection on the compiled shader. Due to this it is pretty common to have to write new c++ code to support a brand new shader. In a way this is intentional as it forces me to rely on existing shaders in the engine which can help keep the shader permutation problem under control. That said, at times you just have to make a new shader and live with that. I am still considering how I might implement this in the future.