---
layout: post
title: Graphics in Unreal Engine 5, what I wish I knew
date: 2025-10-31
---

![alt](assets/img/projects/fluid-rendering/fluid_rendering_thumbnail.gif)

For the past 7 weeks I have been working with a friend and fellow student
Robin Heijmans on a school project. We set out to create a plugin for Unreal Engine 5, that features particle-based fluid simulation and real time ray marched rendering. If this sounds familiar, it’s because we got the idea from Sebastian Lague. He undoubtedly, developed a much more sophisticated approach in Unity. Please do check out his repository and videos on Fluid Simulation, they’re really great and he actually goes into the how and why of developing physics and rendering. However, my goal wasn’t focused on developing yet another ray marching algorithm, but more to simply implement one.

## My challenge
Having only ever worked on gameplay code in Unreal, I have never gotten a chance to actually work on anything graphics related, nor have I had the opportunity to write actual C++ in engine either, regardless I started making a loose plan.

I wanted to take a couple of fixed particle positions within a cubed volume, generate a density map based on those positions, and then render those densities by marching rays through the volume. Rendering an actual fluid on screen wasn’t a concern at the moment, as I just wanted to set up a framework for raymarching the volume, so that most of the graphical improvements could be handled inside the shader itself.

With that in mind, I could safely assume that I needed to set up a couple of things in engine:

- __Shaders__, separate ones for updating the density map and rendering
- A __3D texture__, for the density map
- A __Buffer__ with stored particle positions
- An __Output texture__, to overlay onto the screen

### Shaders… which ones?
Rasterization is mostly ever concerning itself with using vertex- and fragment shaders (or pixel shader in UE), there are a lot more stages to the pipeline itself, but for the most part those two shaders are what we as programmers interact with, when it comes to graphics.

![alt](assets/img/blogs/fluid_rendering/rasterization.webp)
*[https://learnopengl.com/Getting-started/Hello-Triangle](https://learnopengl.com/Getting-started/Hello-Triangle) (31.10.2025)*

However, in my application I simply do not need to concern myself with vertices. All I need to do is cast rays from screen space into the world. To do so in a more minimalistic setting, I choose to use a compute shader instead, since this would also mean I could dispatch a single shader and wouldn’t have to go through any of the pipeline stages mentioned above.

## But how do you make a compute shader in Unreal Engine?
Most of the resources that I could find for this were more examples in other projects than documentation. To be honest, UEs API is greatly documented, and I really don’t mind learning from other people’s implementation, personally I find it more captivating than reading dry, technical jargon. One thing I can only recommend anyone do is read the source code. If anything is ever unclear, or if you’re just looking for a certain function, skimming through header files helped me understand everything a lot more than documentation at times.

Now there are a few types of shaders in Unreal, even some that can be created and compiled at runtime through the Material Editor. But as far as I am aware there are no other shader types accessible when using C++, other than Global Shaders. These will be compiled with your project on startup.

Now, just creating a Global Shader isn’t really an issue, so let’s start with a new header file.

```cpp
// in Shader.hpp

class FMyShader : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyShader);
    SHADER_USE_PARAMETER_STRUCT(FMyShader, FGlobalShader);

    using FParameters = FMyShaderParams;

    static void ModifyCompilationEnvironment(const FGlobalShaderPermutationParameters& Parameters, FShaderCompilerEnvironment& OutEnvironment)
    {
        OutEnvironment.SetDefine(TEXT("THREADS_X"), 8);
        OutEnvironment.SetDefine(TEXT("THREADS_Y"), 8);
        OutEnvironment.SetDefine(TEXT("THREADS_Z"), 1);
    }
};
```

Most of the heavy lifting is hidden away in these macros, but one important thing to note for later is the parameter struct.

```cpp
// in Shader.hpp

BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParams, )
    SHADER_PARAMETER_STRUCT_REF(FMyVolumeStruct, Volume)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, Output)
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture3D<float>, DensityMap)
END_SHADER_PARAMETER_STRUCT()

//...
using FParameters = FMyShaderParams;
```

This essentially declares the environment parameters within the shader itself.

Now before we can even get to dispatching the shader, we still have to implement it. Adding this somewhere on the top of our source file is all we have to do

```cpp
IMPLEMENT_GLOBAL_SHADER(FMyShader, "/Shaders/Compute/MyShader.usf", "Compute", SF_Compute);
```

Note that Unreal likes to use virutal paths for most of everything. That means we still need to setup the virtual path to this shader file and link it to our real asset folder, but I will come back to that later.

For now let’s look at the shader file

```cpp
struct FMyVolumeStruct{
    float3 Position;
    float3 Size;
};

ConstantBuffer<FMyVolumeStruct> Volume;
RWTexture2D<float3> Output;
Texture3D<float> DensityMap;

[numthreads(THREADS_X, THREADS_Y, THREADS_Z)]
void Compute(
    uint3 DispatchThreadId : SV_DispatchThreadID,
    uint GroupIndex : SV_GroupIndex)
{
    //...    
}
```

We can see the our environment variables that we declared in our header file before. The Volume struct, output texture, and density map. Another familiar variable are the macros used in numthreads(), remember we declared these in the shader class itself. These are just the number of threads per dispatch.

```cpp
// from our shader class FMyShader
static void ModifyCompilationEnvironment(const FGlobalShaderPermutationParameters& Parameters, ShaderCompilerEnvironment& OutEnvironment)
{
    OutEnvironment.SetDefine(TEXT("THREADS_X"), 8);
    OutEnvironment.SetDefine(TEXT("THREADS_Y"), 8);
    OutEnvironment.SetDefine(TEXT("THREADS_Z"), 1);
}
```

```hlsl
// translates to 
#define THREADS_X 8
#define THREADS_Y 8
#define THREADS_Z 1

// ... 
[numthreads(THREADS_X, THREADS_Y, THREADS_Z)]
void Compute(
    uint3 DispatchThreadId : SV_DispatchThreadID,
    uint GroupIndex : SV_GroupIndex)
{
    // ...
}
```

## Uniform Buffers
There are a couple of ways to declare and use Unifrom Buffers, you can even declare them globally, that way you won’t have to add the struct definition in the shader yourself. But since this is a little easier to understand and use, I implemted my UB like this.

```cpp
// in Shader.hpp

// Custom struct
BEGIN_UNIFORM_BUFFER_STRUCT(FMyVolumeStruct, )
    SHADER_PARAMETER(FVector3f, Position)
    SHADER_PARAMETER(FVector3f, Size)
END_UNIFORM_BUFFER_STRUCT()

// ...

BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParams, )
    SHADER_PARAMETER_STRUCT_REF(FMyVolumeStruct, Volume) // <- reference to custom struct
    // ...
END_SHADER_PARAMETER_STRUCT()

// in Shader.cpp

IMPLEMENT_GLOBAL_SHADER(FMyShader, "/Shaders/Compute/MyShader.usf", "Compute", SF_Compute);
IMPLEMENT_UNIFORM_BUFFER_STRUCT(FMyVolumeStruct, "Volume");
```

This follows the same playbook as the shader and its environment struct. And you probably already noticed that the string literal passed to the macro in the source file is the same as the volume parameter in the struct.
But beware that this UB isn’t static, so we still need to create it manually. More on this later however. For now…

## Using Shaders
I am going to simplify this greatly, but shaders can essentially be dispatched in 2 different ways. Within the Unreals render pipeline, or independently.

### Independently
There is a yet another macro in Unreal that allows you to queue render commands independently from the actual render pipeline. Just keep in mind that these are lambda functions meaning they will not run immediately.

```cpp
ENQUEUE_RENDER_COMMAND(ParticleBufferInit)(
[this](FRHICommandListImmediate& RHICmdList) 
{
    // Generate particle buffers, density map texture, etc.
});
```

This has its use cases and can come in super handy, in this case to create buffers and textures to use in our shaders later on. But I don’t generally recommend this for actual rendering, like when you want to add something like a post processing effect. A better way to handle the post processing shader I had to make would be with a

### Custom Scene View Extension
I like this a lot better, since it hooks into the rendering pipeline unreal is already calling every frame (it also lets you avoid lambda functions).

To use Unreals Scene View Extension is through inheritance. When createing your own extension from the base class you will still need to add it to a engine subsystem. This might sound intimidating, but it just means that we need to create another class and inherit from the engine subsystem base. The two important functions here are Initialize and Deinitialize, here we can actually instantiate and destroy our extension.

```cpp
// Subsytem.hpp
UCLASS()
class UMySubsystem : public UEngineSubsystem {
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

private:
    TSharedPtr<class FMyExtension, ESPMode::ThreadSafe> MyExtension;
};

// Subsytem.cpp
void UMySubsystem::Initialize(FSubsystemCollectionBase& Collection) {
    MyExtension = FSceneViewExtensions::NewExtension<FMyExtension>();
}

void UMySubsystem::Deinitialize() {
{
    MyExtension.Reset();
    MyExtension= nullptr;
}
```

Now through the base class, the extension comes with a lot of virtual functions you can override to implement your own rendering code, updating buffers, dispatching shaders and so on.

```cpp
// Extension.hpp
class FMyExtension : public FSceneViewExtensionBase 
{
public:
    FMyExtension(const FAutoRegister& AutoRegister);

    virtual int32 GetPriority() const { return 1 << 16; };
    virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override {};
    virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {};

    /* All the rendering happens in here. */
    virtual void PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& InView, const FPostProcessingInputs& Inputs) override;
};
```

This extension is being called on every frame. Not the setup functions, but more importantly for us the RenderThreads from the base class. In there we can finally use the GraphBuilder to dispatch shaders.

This brings me to the main difference to note between using this scene view extension or using the enqueue macro to add render commands independently, which is ownership over the graph builder. Inside the independent render command we are given a FRHICommandListImmediate. With that we usually create the FRDGBuilder seen in the extensions post process function as an argument. The extension however doesn’t have ownership of the graph builder, since it never created it, it’s not responsible to execute it unlike the enqueued render command. Meaning the proper way to use the graph builder is either this

```cpp
ENQUEUE_RENDER_COMMAND(ParticleBufferInit)(
[this](FRHICommandListImmediate& RHICmdList) 
{
    FRDGBuilder GraphBuilder(RHICmdList);

    FComputeShaderUtils::AddPass(
            GraphBuilder,
            RDG_EVENT_NAME("MyShader Dispatch"),
            ERDGPassFlags::Compute,
            ComputeShader,
            PassParameters,
            DispatchCount);

    // End Render Commands with this
    GraphBuilder.Execute();
});
```

or this

```cpp
void FMyExtension::PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& InView, const FPostProcessingInputs& Inputs) 
{
    // GraphBuilder already exists, we can just use it
    FComputeShaderUtils::AddPass(
            GraphBuilder,
            RDG_EVENT_NAME("MyShader Dispatch"),
            ERDGPassFlags::Compute,
            ComputeShader,
            PassParameters,
            DispatchCount);
    // no ownership of the GraphBuilder no need to call Execute()
}
```

## Persistent GPU Resources
Most of the resources you create using the render graph builder have a life time limited to the frame you instantiate them in. But there are ways to externalize them and make them persistent throughout multiple frames.

### Textures
One obvious texture I would want to be have an extended life time is the 3D density map, since it would otherwise be a sizeable chunk of memory to allocate every frame and we only ever read and write from it within shader anyhow, there is no need to interact with it on the CPU side other than to create it on start up.

Making resources persistent means to pool them. In the case of the density map we are looking into creating a IPooledRenderTarget that we can later use to create references to SRVs and UAVs to give read or write access to this texture to shaders.

```cpp
TRefCountPtr<IPooledRenderTarget> DensityMap;

FRHITextureCreateDesc DensityMapDesc = FRHITextureCreateDesc::Create3D(TEXT("DensityMap Desc"))
   .SetExtent(DensityMapSize.X, DensityMapSize.Y)
   .SetDepth(DensityMapSize.Z)
   .SetFormat(EPixelFormat::PF_R8)
   .SetFlags(ETextureCreateFlags::UAV | ETextureCreateFlags::ShaderResource)
   .SetInitialState(ERHIAccess::SRVCompute);
   FTextureRHIRef DensityMapRHI = RHICreateTexture(DensityMapDesc);
   DensityMap = CreateRenderTarget(DensityMapRHI, TEXT("DensityMap"));
```

So when we dispatch all we need to do is register the pooled texture, to get the reference and finally make UAV parameter that we can pass to the shader right before dispatching it.

```cpp
// Before the shader dispatch
FRDGTextureRef DensityMapRef = GraphBuilder.RegisterExternalTexture(DensityMap);
MyShaderParams.DensityMap = GraphBuilder.CreateUAV(FRDGTextureUAVDesc(DensityMapRef));
```

### Buffers
The process for Buffers is a little different, but follows the same principle. First we allocate the buffer and pool it on start up, so that we only need to create SRVs and UAVs for our shaders later on.

```cpp
FRDGBufferDesc PositionsDesc = FRDGBufferDesc::CreateStructuredDesc(sizeof(FVector3f), NumPositions);
FRDGBufferRef TempPositionsRef = GraphBuilder.CreateBuffer(PositionsDesc, TEXT("Positions"));

TRefCountPtr<FRDGPooledBuffer> Positions = GraphBuilder.ConvertToExternalBuffer(TempPositionsRef);
```

Since our buffer is now pooled the same way we did the density map, we can also treat it the same way when passing it to shaders.

```cpp
// Before the shader dispatch
FRDGBufferRef PositionsRef = GraphBuilder.RegisterExternalBuffer(Positions, TEXT("Positions"));
MyShader.Positions = GraphBuilder.CreateUAV(PositionsRef);
```

## Drawing to the Screen
Now that we have everything set up for the shaders writing to the output texture isn’t a complicated issue, the only thing were still missing is actually rendering it onto the screen. The easiest way to do so, that I can only recommend, is to copy our output texture to the screen. If we have access to the current scene color texture, we can simply do so with a copy texture pass.

The post processing function that came with the scene view extension comes with neat side effect, the post process inputs, which contains the current scene color texture.

```cpp
void FMyExtension::PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& InView, const FPostProcessingInputs& Inputs) 
{
    // Get scene color from post process input
    FRDGTexture* SceneColor = Inputs.SceneTextures->GetContents()->SceneColorTexture;

    // Create output texture to render to in MyShader
    FRDGTextureDesc OutputDesc = SceneColor->Desc;
    FRDGTextureRef OutputTexture = GraphBuilder.CreateTexture(OutputDesc, TEXT("MyShader Output"));
    MyShaderParams.Output = GraphBuilder.CreateUAV(FRDGTextureUAVDesc(OutputTexture));

    // Dispatch MyShader -> render to output texture ...

    // Done rendering to our output texture -> copy it onto the screen
    AddCopyTexturePass(GraphBuilder, OutputTexture, SceneColor);
}
```

Beware that this will simply overwrite the entire screen, so we do still add it the color we write to the output texture in our shader.

```cpp
[numthreads(THREADS_X, THREADS_Y, THREADS_Z)]
void Compute(
 uint3 DispatchThreadId : SV_DispatchThreadID,
 uint GroupIndex : SV_GroupIndex )
{
    // add onto output color the scene color
    float3 OutputColor = SceneColor[DispatchThreadId.xy];

    // add onto output color the raymarched output 
    // ...

    // write scene + raymarched output to texture
    Target[DispatchThreadId.xy] = OutputColor;
}
```

Other Resources
I skimped over a lot of steps here, since I didn’t really want for this to be a tutorial. There are lot of other resources and examples to actually learn these things within actual projects. I simply wanted to point out some things that weren’t explicitly said anywhere and I had to painstakingly track down myself.

Some great resources I found while learning all this:
- [A blog from another BUas student](https://karolinamot.github.io/HLSL-and-ue5-blog/#global-shaders-and-sceneviewextension), [Karolina Motužytė](https://github.com/KarolinaMot)
- [Boids example using Niagara](https://github.com/Shadertech/UE5NiagaraComputeShaderIntegration/tree/master)
- [Volumetric Cloud plugin by another BUas student](https://github.com/mxcop/vapor), [Max](https://m4xc.dev/)
- [UE4 Shader Introduction](https://logins.github.io/graphics/2021/03/31/UE4ShadersIntroduction.html)

Please do check them out they cover a lot more ground than I did here and go into a lot more detail. Still I hope that you could learn at least something from what I wrote here.

Oh and my partner on this also wrote a blog about the physics side of our project please do check it out as well if you’re interested.
[Real-Time Particle Based Fluid Simulation Plugin for Unreal Engine](https://medium.com/@robinheijmans/real-time-particle-based-fluid-simulation-plugin-for-unreal-engine-b38294f6a507)

![alt](assets/img/Logo_BUas.png)
*this was a student project from BUas Games*