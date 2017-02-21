---
title: Setting up SDL2 and SDL_gpu on Windows using CMake
layout: single
category: posts
tags:
 - sdl2
 - sdl_gpu
 - gamedev
 - cmake
 - clion
 - mingw
published: true
---

In this post I'll cover detailed explanation of setting up SDL2 along with SDL_gpu on Windows using CMake (CLion). I'll build both libraries locally with MinGW. I plan to develop my own simple 2D game engine on top of SDL2 and document my progress periodically.

There are a few guides about how to start using SDL2 for different operating systems and compilers. I thought it would be nice to add a complete guide for the people who want to use the same environment as I'm using.

**SDL** (Simple Directmedia Layer) is a cross-platform library providing low level access to audio, input and graphics hardware. There are alternatives to SDL, however none of them officially support Mobile platforms.

I'm going to use **CLion** on Windows. Instead of using SDL's renderer, initially, I'm planning to use **SDL_gpu** (<https://github.com/grimfang4/sdl-gpu>), a native OpenGL/OpenGLES renderer library for SDL/SDL2. Later on, I'll implement my own renderer.

Although there are available debug builds for SDL2 and SDL_gpu, I couldn't find a way to attach sources on CLion for libraries compiled elsewhere. Thus I'm going to compile them and generate libraries myself. Note that, before publishing, its important to replace debug libraries with release ones, expecially with the ones provided by the library owners because they probably 
optimized them better.

First, we should install necessary **MinGW** packages.
Grab the installer from <https://sourceforge.net/projects/mingw/files/> and install the base, developer-toolkit, gcc-g++ and msys-base packages.

![](../../assets/images/mingw_setup.png)

Download and install **CMake**.
<br/>
<https://cmake.org/download/>

Download **SDL2 development libraries** and source code.
<br/>
<http://libsdl.org/download-2.0.php>

Download **SDL_gpu (SDL2_gpu for SDL2) development libraries** and source code.
<br/>
<https://github.com/grimfang4/sdl-gpu/releases>

For both, extract the libraries and header files to the same folder for proper use (e.g. *C:/Development/Library*).

Now, let's configure our **CMakeList** file.

Here we have 2 choices. Either we explicitly include and link SDL2 and SDL_gpu by hardcoding the paths to CMakeList or use CMake's **find_package**. **find_package** will search and find the library files. CMake's **find** commands work by searching suffixes. If you put your libraries and header files in any folder named other than **include** or **lib** you have to add those paths explicitly to the module file.

Good folks out there have already written some module files for SDL2. Get **FindSDL2.cmake** and place it in a common path (*C:/Development/CMake/modules* in this case).
<br/>
<https://github.com/tcbrindle/sdl2-cmake-scripts>

I'll further explain the details on CMakeList file.

```cmake
set(CMAKE_VERBOSE_MAKEFILE ON)
cmake_minimum_required(VERSION 3.6)
project(SDL_Sandbox)

set(CMAKE_CXX_STANDARD 11)

# Put the FindSDL2.cmake in a folder and tell CMake to find it there
set(CMAKE_MODULE_PATH "C:/Development/CMake/modules")

# This path is used by FindSDL2 module. 
# It will find include and lib folders under this directory.
# This is because find command doesn't include any path for Windows.
# So it can't find anything using default paths.
# SDL2_PATH is already expected in module file. I'm using the same variable.
set(SDL2_PATH "C:/Development/Library")

# CMake parses module files by their names. It will use FindSDL2 in this case.
find_package(SDL2 REQUIRED)

# Explicitly specify paths for SDL_gpu. It's hardcoded and ugly but easy.
set(SDL2_GPU_LIBRARY_PATH "C:/Development/Library/lib/libSDL2_gpu.dll.a")
set(SDL2_GPU_INCLUDE_PATH "C:/Development/Library/include/SDL2_gpu")

# SDL2_INCLUDE_DIR variable set by the FindSDL2 module (if it finds directory).
include_directories(${SDL2_INCLUDE_DIR} ${SDL2_GPU_INCLUDE_PATH})

set(SOURCE_FILES src/main.cpp)
add_executable(SDL_Sandbox ${SOURCE_FILES})

# SDL2MAIN_LIBRARY is also specified within SDL2_LIBRARY by the FindSDL2 module.
# It's needed for Windows specific main function.
# If I don't use FindSDL2 module, I have to link it as well.
target_link_libraries(SDL_Sandbox
	#	${SDL2MAIN_LIBRARY}
        ${SDL2_LIBRARY}
        ${SDL2_GPU_LIBRARY_PATH})
```

At this point everything is ready. Fire up the CLion and create an empty project.

Edit CMakeLists and run CMake by reloading changes. As we are dynamically loading SDL2 and SDL_gpu, we need to put **SDL2.dll** and **libSDL2_gpu.dll** under *Windows/system32* or our project's build directory (right near to exe).

Test with a simple main.

```cpp
int main(int argc, char *argv[]) {
    GPU_Init(1024, 768, 1);
    return 0;
}
```

From now on we can use SDL2 and SDL_gpu, But, we can't debug (step into) any SDL2 or SDL_gpu code. This is very annoying. I constantly step through the execution of a program to find errors faster. And if I'm going to develop something relatively big, I need this ability. First, to debug any external library, it needs to be compiled with the debug flag (-g). Debug builds provide more info about the execution and allow you to step into code. Debug libraries contain source info. They have the relative path to their sources in the debug_info section. Both SDL2 and SDL_gpu official releases include debug builds however this isn't enough. CLion has no ability to attach sources (AFAIK) to external libraries. So, I need to compile these libraries locally.

### Compiling SDL2

Run the **msys.bat** under *.../MinGW/msys/1.0*

Extract SDL2 sources to *.../MinGW/msys/1.0/home/{USER NAME}* and cd.

There are various custom options for compiling SDL2. I'm not going to use any of them and compile a bare debug library.

**NOTE:** If you encounter the error below:

```
src/audio/winmm/SDL_winmm.c: In function 'DetectWaveOutDevs':
src/audio/winmm/SDL_winmm.c:57:33: error: unknown type name 'WAVEOUTCAPS2W' DETECT_DEV_IMPL(SDL_FALSE, Out, WAVEOUTCAPS)
```

Fix it by editing the **winmm.h** file according to official change at: <https://hg.libsdl.org/SDL/rev/7e06b0e4dbe0>

I could also make this via CMake-gui but this is easier in this case. Run configure then make like below.

```
./configure --prefix=/mingw CFLAGS='-g'
make
make install
```

Library files have been put under *.../MinGW/lib* and *build*. Grab those (don't forget the libSDL2main.dll) and put them under the common *lib* folder (*C:/Development/Library/lib/* for me).

That's it. Don't forget to remove old ones.

For more info about building SDL: <https://web.archive.org/web/20120820141724/http://blog.pantokrator.net/2006/08/08/setting-up-msysmingw-build-system-for-compiling-sdlopengl-applications/>

### Compiling SDL_gpu

Run the **msys.bat** under `.../MinGW/msys/1.0`

Extract SDL_gpu sources to `.../MinGW/msys/1.0/home/{USER NAME}` and cd.

Run commands below. 

**`After CMake, open CMakeCache.txt and ensure that SDL2 paths are set correctly.`**

```
CMake -G "MinGW Makefiles"
mingw32-make
```

Libraries compiled and placed under *src*. Take those and place them under the common *lib* folder (*C:/Development/Library/lib/* for me).

Don't forget to remove old libraries. And make sure CMake linking the new ones (may use cache). Go back to CLion and test that stepping into external library methods work.

That's all. Whole process shouldn't take more than an hour. Hope this saves someone a couple of hours.

**An example project is located at: <https://github.com/ccavusoglu/SDL_Sandbox>**
