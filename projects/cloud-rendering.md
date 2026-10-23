---
layout: post
title: Real Time Cloud Rendering
date: 2025-10-31
---

**My Year 2 self-study project at BUas.**

In my second year studying at BUas, I worked for 8 weeks on a self-study project. The assignment we were given gave us complete freedom in our choice of topic, and only limited us in time and complexity. After deliberating for a while on what topic to explore, I chose to try myself at graphics programming with Volumetric Cloud Rendering. 

## My Work

Personally, I had very little experience in graphics programming prior to this project. I only made a simple 3D OpenGL Renderer, and a simple voxel ray-tracer, in my 1st year at BUas before looking into this, but I desperately wanted to broaden my horizons. 

## Inspiration

The main inspiration for this project, was Guerilla's Volumetric Clouds from Horizon: Zero Dawn and Horizon: Forbidden West and a very nifty article on the system the developed for these games: [Nubis³](https://www.guerrilla-games.com/read/nubis-cubed).

I dove more into the implementation and design of this this Blog Post: [Attempting Volumetric Clouds from Scratch](/blogs/cloud-rendering).

## Approach

The approach I took, I heavily based on Nubis³. Choosing this this project to focus on a study of ray-marching. I also chose to do this entirely in a custom engine, to not get side tracked by getting acquainted with a new work environment or game-engine (since I haven't had any custom graphics experience in Unreal or Unity). 

My starting point was a light weight custom engine written in C++, that I had previously used in my time at BUas. 