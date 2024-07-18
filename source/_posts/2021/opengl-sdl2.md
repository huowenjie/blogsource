---
title: Opengl3.3+ 搭配 SDL2 框架渲染
date: 2021-04-03 21:38:00
tags: [Opengl, Programming]
categories:
  - 计算机图形学
---

学习 Modern Opengl 的第一步，Opengl 的教程参考 [LearnOpengl](https://learnopengl-cn.github.io/)

<!-- more -->
## 准备工作

1. 首先下载 Opengl 的第三方中间件 [glad](https://glad.dav1d.de/) 并编译；
2. 下载 [SDL2.0](https://github.com/libsdl-org) 并编译，现在有可能变成 SDL3.0，但是 API 调用方式应该大差不差；
3. 当前的工程需要链接 glad（或者直接在项目中包含其源码亦可） 和 SDL 库，glad 会动态加载底层设备厂商的实现，给我们提供较新版本的 Opengl API。

话不多说，代码如下（VS2015+ 建立）：

``` C
#include <stddef.h>
#include <SDL.h>
#include "glad/glad.h"
 
#pragma comment(lib, "SDL2.lib")
#pragma comment(lib, "SDL2main.lib")
 
#define SCREEN_WIDTH 800
#define SCREEN_HEIGHT 600

int main(int argc, char *argv[])
{
　　SDL_Window *window = NULL;
　　SDL_GLContext context = NULL;
　　SDL_Event event;
 
　　int ret = -1;
 
　　if (SDL_Init(SDL_INIT_VIDEO) < 0) {
　　　　return ret;
　　}
 
　　SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3);
　　SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3);
　　SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, SDL_GL_CONTEXT_PROFILE_CORE);
 
　　window = SDL_CreateWindow("SDL-OpenGL3.3",
　　　　SDL_WINDOWPOS_UNDEFINED,
　　　　SDL_WINDOWPOS_UNDEFINED,
　　　　SCREEN_WIDTH, SCREEN_HEIGHT,
　　　　SDL_WINDOW_OPENGL | SDL_WINDOW_SHOWN);
 
　　if (!window) {
　　　　goto end;
　　}
 
　　context = SDL_GL_CreateContext(window);
　　gladLoadGLLoader((GLADloadproc)SDL_GL_GetProcAddress);
 
　　glClearColor(1.0f, 1.0f, 0.5f, 1.0f);
 
　　while (1) {
　　　　while (SDL_PollEvent(&event)) {
　　　　　　if (event.type == SDL_QUIT) {
　　　　　　　　goto end;
　　　　　　};
　　　　}
 
　　　　glClear(GL_COLOR_BUFFER_BIT);
 
　　　　SDL_GL_SwapWindow(window);
　　　　SDL_Delay(100);
　　}
 
　　ret = 0;
end:
　　if (context) {
　　　　SDL_GL_DeleteContext(context);
　　}
 
　　if (window) {
　　　　SDL_DestroyWindow(window);
　　}
 
　　SDL_Quit();
　　return ret;
}
```

