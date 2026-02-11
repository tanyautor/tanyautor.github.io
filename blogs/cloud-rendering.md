---
layout: post
title: Attempting Volumetric Clouds from Scratch
date: 2025-01-24
---


Guerilla's Horizon series, is a great example for realistic Clouds in video games. They pretty much define the state of the art for realistic, real-time, Clouds. Their atmospheric lead, Andrew Schneider, held a presentation at [Siggraph](https://www.guerrilla-games.com/read/nubis-evolved) about how they rendered their clouds. Lo and behold, the way they render their clouds is through volumetric Ray Marching.

## Ray Marching vs. Ray Tracing
Ray Tracing has become a kind of a buzzword around video games, since NVidia has been pushing their RTX hardware. Even though they sound similar, the ideologies between these two techniques are different. The goal of Ray Tracing is to find surface intersections, which is precisely why we hear about it so much in discussion about general rendering. The discussion is then Rasterization vs. Ray Tracing, since Ray Tracing is mostly about the surface you hit. And projecting that hit on the pixel you’re processing. Essentially emulating rays of light.
The goal of Ray Marching is usually to sample a lot of points in a scene. Instead of Tracing a Line (Ray) to a surface hit, you’d step from a starting point in a direction, marching a Line (Ray), as you go. The reason for taking multiple steps, is that it allows you to sample at each and everyone, checking for intersections, distances, etc.

![alt](assets/img/blogs/raymarching_graphic.webp)
*[https://barradeau.com/blog/?p=575](https://barradeau.com/blog/?p=575) (23.1.2025)*

## Approach
The whole reason why we chose Ray Marching over Tracing, was to sample density, not surfaces. Since the Marching algorithm is already based on a step-by-step approach, it makes only sense that we use this to sample density at every step.
To sample our density, we first need a volume from which to sample from. A usual approach to Ray Marching is Signed Distance Functions (SDFs). These come in a variety of shapes and they’re how most people get to learn about Ray Marching in the first place.
But I didn’t want to be limited to the shapes of primitives of SDFSs. The volume I chose to sample density from, was a Grid. The Grid is nothing more than a 3D Texture. This allows us to sample density at UV- coordinates, that we derive from the local position of our ray.
Within the Texture we layer data in color channels. The reason for using the Grid is to be able to shape the our Clouds within the Volume, since each Texel is occupied by a unique value. For now we will occupy the red channel with a Dimensional Profile and the green channel with a Coverage value.

## Generating the Dimensional Profile
Before we start ray marching, we need some useful data to sample. So, we want to generate Volume. It makes sense to do this only once on start up. Doing this on the CPU is just fine as well, that way we can rearrange out buffer however we want, before sending them to the GPU.

Since we only need two color channels we can start with a simple vec2 array. I am using GLM for this, but any two floating points will do.

Since it is sort of a confusing term we can think of the dimensional profile as the shape of the Cloud itself. For now we want generate this Profile around the center of the Grid. To do this we need three gradients, a top gradient increasing in value going upwards, a bottom gradient increasing in value going downwards, and an edge gradient increasing in value towards the center. This is going to be important later on.

```glsl
const float density = 1.f;
const float coverage = 1.f;

for (int z = 0; z < grid_size.z; z++)
{
    for (int y = 0; y < grid_size.y; y++)
    {
        for (int x = 0; x < grid_size.x; x++)
        {
            vec2 texel;
            vec3 grid_pos = vec3(x, 0.f, z);

            float dist_from_center = 1.f - length(grid_pos - grid_size / 2) / grid_size;
            float height_fraction = (float)y / (float)grid_size.y;

            // generate dimensional profile
            float gradient_top        =       height_fraction;
            float gradient_bottom     = 1.f - height_fraction;
            float gradient_edge       = 1.f - dist_from_center;
            float dimensional_profile = gradient_top * gradient_bottom * gradient_edge; 
            dimensional_profile *= pow(2.f, 3.f); // augment profile with the num of gradients used
            dimensional_profile *= density; 

            // dimensional profile in the red channel
            texel.r = dimensional_profile;
            // coverage in the green channel
            texel.g = coverage;
        }
    }
}
```

The input values Coverage and Density are going to help shape the Cloud later on. Right now, they shouldn’t disrupt anything though, so let’s keep them at 1.0. The loop is going to generate a very simple shape, but it is an okay starting point that I will expand on later.

## Sampling Density
Ray Marching isn’t restricted to any processor or platform, neither is it dependent on any special hardware. You could write a Ray Marching algorithm on the CPU even. However, for overall simplicity and familiarity, I am goin to write this in a compute shader for OpenGL.

Setting up the shader involves a little leg work. Firstly, we need two uniforms, one cloud data Texture to read our Dimensional Profile and Coverage from. And since this isn’t a fragment shader, we need a 2D Image to render our output onto as well.

```glsl
layout (binding = 0) uniform image2D output;
layout (location = 0) uniform sampler3D cloud_data;
```

The output image we simply bind to a Texture ID on our host program. We don’t really need to write or read from it manually from here, we only need its ID to later render the finalized Texture on screen.

```cpp
unsigned int OutputTextureId = 0;
glActiveTexture(GL_TEXTURE0);
glGenTextures(1, &OutputTextureId);
glBindTexture(GL_TEXTURE_2D, OutputTextureId);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA32F, screen_size.x, screen_size.y, 0, GL_RGBA, GL_FLOAT, 0);
glBindImageTexture(0, OutputTextureId, 0, GL_FALSE, 0, GL_READ_ONLY, GL_RGBA32F);
```

The cloud data Texture we simply supply with our generated Grid from before. After that we just bind it whenever we dispatch our Ray Marching shader.

```cpp
unsigned int CloudSamplerId = 0;
glActiveTexture(GL_TEXTURE0);
glGenTextures(1, &CloudSamplerId);
glBindTexture(GL_TEXTURE_3D, CloudSamplerId);
glTexImage3D(GL_TEXTURE_3D, 0, GL_RG32F, grid_size.x, grid_size.y, grid_size.z, 0, GL_RG, GL_FLOAT, cloud_data);
```

In our shaders main function, we want to calculate for every pixel a ray; if necessary adjust the ray distance to start inside the volume, and then step through the volume, while sampling density along the way. Lastly we just store the accumulated density in our output image.

```glsl
void main()
{
    // Get current pixel coord
    ivec2 pixel_coord = gl_GlobalInvocationID.xy;
    vec4 pixel = vec4(0);
    // Set up primary ray
    vec2 screen_uv = gl_GlobalInvocationID.xy / gl_NumWorkGroups.xy;
    Ray p_ray = GetPrimaryRay(screen_uv);

    // Step to the volume, if necessarry
    if (!CubeContains(p_ray))
    {
        p_ray.m_t = RayCubeIntersect(p_ray);
    }
    // Step through the volume, sampling density along the way
    if (p_ray.m_t >= 0.0/*ray is valid*/)
    {
        pixel = SampleStepping(p_ray);
    }

    // Write to output image
    imageStore(raym_output, pixel_coord, pixel);
}
```

Generating the primary Ray through UV-coordinates and intersecting with the Volume is pretty straight forward. Since we can assume our Volume is a cube with dimensions of 1x1x1. The only important thing we need, is for the ray to have an Origin, a Direction, and a Distance as member variables.

Sampling density is pretty simple too. Since we can assume our Volume is located at position [0,0,0] and isn’t moving, we can just use our ray position as UV-coordinates to read from the cloud data Texture. This reads the previously generated Dimensional Profile and Coverage.

```glsl
vec4 SampleStepping(inout Ray _ray)
{
    vec4 value = vec4(0);
    int num_samples = 0;
    int max_steps = 500;
    float step_size = 1.0 / max_steps;
    while(++num_samples < max_steps && val.a < 1.0)
    {
        // step ray forward
        _ray.m_t += step_size;
        if(!CubeContains(_ray)) break; // stepped out of cube -> stop

        // assuming cube position is 0,0,0 -> ray_pos is in local space
        vec3 ray_pos = _ray.origin + (_ray.direction * _ray.distance);

        // using red = dimensional profile, and green = coverage to calculate density
        vec2 cloud_profile = texture(cloud_data, ray_pos);
        float noise_composite = 1.0;
        float density = saturate(noise_composite - (1.0 - cloud_profile.r)) * cloud_profile.g;

        if(density > 0.0)
        {   
            vec4 color = vec4(mix(vec3(1.0, 1.0, 1.0), vec3(0.0, 0.0, 0.0), density), density);
            color.rgb *= color.a;

            value += color * (1.0 - val.a);
        }
    }
    return value;
}
```

The output is looking a little strange, but this isn’t unexpected. Since we don’t have a noise composite yet and neither do we sample light.

![alt](assets/img/blogs/first_sample.webp)

Now to shape this into volume into a cloud, a common approach is using noise. The code above is already working with a noise composite, it just needs to be supplied with some useful data.

Perlin noise is not a perfect solution for everything. But it is an easy thing to come across and there are a lot of libraries that generate coherent 3D noise, so I wouldn’t recommend generating it yourself. However, GLM seems to already support Perlin noise, so the only thing we need is another 3D Texture to read it from.

```glsl
layout (binding = 0) uniform image2D output;
layout (location = 0) uniform sampler3D cloud_data;
layout (location = 1) uniform sampler3D noise_data;
```

And we sample the noise composite the same way we do our cloud data.

```glsl
vec2 cloud_profile = texture(cloud_data, ray_pos);
float noise_composite = texture(noise_data, ray_pos);
```

![alt](assets/img/blogs/perlin_sample.webp)
*This looks very Perlin like. You can try generating a composite out of multiple types of noise.*

The output looks a lot more cloud shaped, but very bright all over. This means, the one last thing to take into account is lighting. An easy implementation for this I’ve gotten from [Maxime Heckel’s](https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/) blog about volumetric clouds. It’s using the directional derivative method, from [Inigo Quilez](https://iquilezles.org/). Which means we need to sample density once more in the direction of our light source, to calculate a “diffuse” value. This is solution is far from perfect since it is meant to be meant to approximate surface normals on SDFs. But Maxime’s version works on density, so it is quite easy to implement in our existing function. The results, will probably struggle with any concave shape, but they should look believable enough for now.

This makes the complete sampling function looking like this:

```glsl
vec4 SampleStepping(inout Ray _ray)
{
    vec3 light_direction = normalize(vec3(0,0,0) - vec3(1,1,-1));

    vec4 value = vec4(0);
    int num_samples = 0;
    int max_steps = 500;
    float step_size = 1.0 / max_steps;
    while(++num_samples < max_steps && val.a < 1.0)
    {
        // step ray forward
        _ray.m_t += step_size;
        if(!CubeContains(_ray)) break; // stepped out of cube -> stop

        // assuming cube position is 0,0,0 -> ray_pos is in local space
        vec3 ray_pos = _ray.origin + (_ray.direction * _ray.distance);

        // using red = dimensional profile, and green = coverage to calculate density
        vec2 cloud_profile = texture(cloud_data, ray_pos);
        float noise_composite = texture(noise_data, ray_pos);
        float density = saturate(noise_composite - (1.0 - cloud_profile.r)) * cloud_profile.g;

        if(density > 0.0)
        {
            // sampling density in light direction
            vec3 d_ray_pos = ray_pos + vec3(0.1) * vec3(light_direction);
            vec2 d_cloud_profile = texture(cloud_data, d_ray_pos);
            float d_noise_composite = texture(noise_data, d_ray_pos);
            float d_density = saturate(d_noise_composite - (1.0 - d_cloud_profile.r)) * d_cloud_profile.g;

            float dif = saturate((_density - density2) / 0.1);

            vec3 light_color = vec3(1.0, 0.0, 1.0);
            vec3 cloud_color = vec3(0.6, 0.6, 0.6);
            vec3 lin = cloud_color * 0.75 + light_color * 0.25 * dif;
            vec4 color = vec4(mix(vec3(1.0, 1.0, 1.0), vec3(0.0, 0.0, 0.0), density), density);
            color.rgb *= lin;
            color.rgb *= color.a;

            value += color * (1.0 - val.a);
        }
    }
    return value;
}
```


![alt](assets/img/blogs/sdf_lighting.webp)
*The purple color is adjustable of course, look for a light_color variable ;)*

## Adjusting the Cloud Shape
The general shape seems very scattered, more like Smoke and less like an actual cloud laying on the Troposphere. Basing the dimensional profile around the center alone makes the result also seem a bit rigid. An interesting concept I found in [Guerillas Siggraph Presentation](https://www.guerrilla-games.com/read/nubis-evolved), was their Vertical Profile.

They used two look up Textures to generate essentially a vertical overlay to shape the Cloud. They sample these Textures through a Cloud Type on the x-axis, and a gradient on the y-axis, specifically the bottom and top gradients, we used to calculate the Dimensional Profile.

![alt](assets/img/blogs/dimensional_profile.webp)

Thinking back on how we generated the Dimensional Profile and Coverage, we simply set the Coverage to 1.0. If we set the Coverage value to the Vertical Profile, it should make our cloud look more like it’s laying on the Troposphere, since we’re overlaying it onto every slice of the Grid vertically and cutting the bottom of the cloud with a “sharp” Texture.

The loop for generating cloud data should look like this now:

```glsl
const float density = 1.f;
const float cloud_type = 1.f;

for (int z = 0; z < grid_size.z; z++)
{
    for (int y = 0; y < grid_size.y; y++)
    {
        for (int x = 0; x < grid_size.x; x++)
        {
            vec2 texel;
            vec3 grid_pos = vec3(x, 0.f, z);

            float dist_from_center = 1.f - length(grid_pos - grid_size / 2) / grid_size;
            float height_fraction = (float)y / (float)grid_size.y;

            // generate dimensional profile
            float gradient_top        =       height_fraction;
            float gradient_bottom     = 1.f - height_fraction;
            float gradient_edge       = 1.f - dist_from_center;
            float dimensional_profile = gradient_top * gradient_bottom * gradient_edge; 
            dimensional_profile *= pow(2.f, 3.f); // augment profile with the num of gradients used
            dimensional_profile *= density; 

            // dimensional profile in the red channel
            texel.r = dimensional_profile;

            // calculate vertical profile from look up textures  
            loat cloud_max = cloud_lookup[cloud_type][gradient_bottom].x;
            loat cloud_min = cloud_lookup[cloud_type][gradient_top].y;

            // using vertical profile as coverage "overlay"
            const float vertical_profile = cloud_min * cloud_max;
            const float coverage = vertical_profile;

            // coverage in the green channel
            texel.g = coverage;
        }
    }
}
```

And now our code should generate something like this:

![alt](assets/img/blogs/final_shape.webp)

This cloud looks a little different in shape, but it’s still quite untamed. Note that we replaced our previous constant value, Coverage, with the Cloud Type used for the look up Textures. So, if this value samples our look up Textures, then changing it should shape the cloud we sample at run time too.

![alt](assets/img/blogs/shape_adjusted.webp)

Evidently, calculating the Vertical Profile at different position along the x-axis, changes our Cloud shape drastically. You can think of the Vertical Profile as essentially cutting the Cloud off, at different heights in the Troposphere.

## Moving Clouds
Before calling this finished I’d like to add some Wind. In it’s current state we’re rendering a still image, every frame. Besides being wasteful, it’s also boring. We need a way to obscure our clouds at run time and make it look like the Density we are sampling is moving, in a complex, yet coherent manner.

This sounds very complex and difficult to implement, but really we are already disturbing the “clean” Density we read from our cloud Texture with Perlin noise. Remember this:

```glsl
float density = saturate(noise_composite - (1.0 - cloud_profile.r)) * cloud_profile.g;
```

This noise composite, is disturbing the shape of the cloud and transforms it into what ends up on screen. And we sample it at the same position as we do our density.

```glsl
float noise_composite = texture(noise_data, ray_pos);
```

If we move the sample position to an adjacent value, we should get a different value, that is not too far from what we originally sampled and shaped our density with. All we need is an animated offset to our UV-coordinate, in this case the rays position.

```glsl
layout (location = 2) uniform vec3 movement;
```

So we start by adding the offset vec3, that we then simply add onto the UV-coordinate.

```glsl
float noise_composite = texture(noise_data, ray_pos + movement);
```

Now we simply need to manage the offset in our host program. To move this value, we simply multiply a Wind Direction with the Delta Time of the frame and send it to the uniform.

```cpp
static vec3 wind_direction = vec3(0.35f, 0.f, 0.35f);
static vec3 location += wind_direction * delta_time;
glUniform3f(2, location.x, location.y, location.z);
```

![alt](assets/img/blogs/cloud_rendering_thumbnail.gif)

Adding the movement wasn’t too big of a challenge, but it does make the Cloud look a lot more lively.

To be fair, I haven’t looked into the “state of the art” of moving clouds. Even if this is pseudo movement, the result looked quite believale for most applications I could think of.

## Conclusion
Overall I heard a number of people say, that this output does make a passable Cloud. But this implementation still leaves a lot of open ends. It isn’t very optimized nor is it pushing any limits conceptually. The actual sampling is kept simple, since we preprocess and reuse a lot of important information. This does leave room for further development, like adding a more fitting lighting model, for example with actual light scattering, or other performance heavy additions.

The lighting model implemented here, was meant for approximating surface normals of SDFs, which isn’t what we need for clouds. The biggest assumption that this makes, is that the Cloud itself is a convex body. However, it is incredibly likely that the Cloud it’s sampling, like the ones shown above, are in fact concave. And a single step into the lights direction doesn’t accurately check for direct occlusion. Nor does is even touch on the topic of light scattering inside the volume. So, this would definitely require more attention.

In addition the Clouds shape is still quite rigid and limited to the Dimensional Profile. A massive improvement would likely be a tool to edit the Clouds Density and Coverage within the Volume independently, or generate them based on 3D Meshes.

![alt](assets/img/blogs/BUas_logo.webp)
*his was a student project from BUas Games*