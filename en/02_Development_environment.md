In this chapter, we'll set up your environment for developing Vulkan 
applications and install some useful libraries. All the tools we'll use, 
except for the compiler, are compatible with Windows, Linux and macOS, but the
steps for installing them differ a bit, which is why they're described
separately here.

First, some common considerations for all platforms:

### Vulkan SDK

The most important part you'll need for developing Vulkan applications is
the SDK. It includes headers, standard validation layers, debugging tools
and a loader for the Vulkan functions. The loader looks up the functions in the
driver at runtime, similarly to GLEW for OpenGL—if you're familiar with that.

The SDK can be downloaded from [the LunarG website](https://vulkan.lunarg.com/)
using the buttons at the bottom of the page. You don't have to create an
account, but it will give you access to some additional documentation that may
be useful to you.

![](/images/vulkan_sdk_download_buttons.png)

Proceed through the installation and pay attention to the installation
location of the SDK. The first thing we'll do is verify that your graphics
card and driver properly support Vulkan. Go to the directory where you
installed the SDK, open the `bin` directory and run the `vkcube` demo.
You should see the following:

![](/images/cube_demo.png)

There is another program in this directory that will be useful for
development. The `glslangValidator` and `glslc` programs will be
used to compile shaders from the human-readable [GLSL](https://en.wikipedia.
org/wiki/OpenGL_Shading_Language) to bytecode. We'll cover this in depth in
the [shader modules](!en/Drawing_a_triangle/Graphics_pipeline_basics
/Shader_modules) chapter. The `bin` directory also contains the binaries of
the Vulkan loader and the validation layers, while the `lib` directory
contains the libraries.

Lastly, there's the `include` directory that contains the Vulkan headers.
Feel free to explore the other files, but we won't need them for this tutorial.

To automatically set the environment variables up that VulkanSDK will use to 
make life easier with the CMake project configuration and various other 
tooling, We recommend using the `setup-env` script. This can be added to 
your auto-start for your terminal and IDE setup such that those environment 
variables work everywhere.

If you receive an error message, then ensure that your drivers are up to date,
include the Vulkan runtime and that your graphics card is supported. See the
[introduction chapter](!en/Introduction) for links to drivers from the major
vendors.

### CMake
For all the warts of working in cross-platform projects, CMake has become an 
industry-wide staple. It allows developers to create a project wide build 
description file which takes care of setting up and configuring all the 
support tools required to create any project.
Other build systems that achieve similar capabilities exist such as bazel, 
however, none are as widely used and accepted as CMake is.
A full discription of how to use CMake is beyond the scope of this tutorial, 
however, further details can be found at [CMake](http://www.cmake.org)

Vulkan SDK has support for using find_package. To use it with your project, 
you can add the search path for the *-config.cmake to the `HINTS` portion of 
the find_package config calls: i.e. find_package(Slang CONFIG HINTS "$ENV
{VULKAN_SDK}/lib/cmake").

In the future, FindVulkan.cmake might migrate to the *-config.cmake standard,
however at the time of writing it is recommended to grab FindVulkan.cmake 
from VulkanSamples, as the one from Kitware is both deprecated and has bugs 
in the macOS build.  You will find it in the code directory [FindVulkan.
cmake](!../code/CMake/FindVulkan.cmake)

Using FindVulkan.cmake is a project-specific file, you can take it and make 
changes as necessary to work well in your build environment, and can craft 
it further to your needs.  The one Khronos distributes in VulkanSamples is 
well tested and is a good starting point.

To use it, add it to your CMAKE_MODULE_PATH like this:
`list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/CMake")`

This will allow other projects that distribute via Find*.cmake to be placed 
in that same folder. See the accompanying [CMakelists.txt](!..
/code/CMakeLists.txt) for an example of a working project.

Vulkan has support for C++ modules which became available with c++20. A 
large advantage of C++ modules is they give all the benefits of C++ without  
the overhead long compile times. To do this, the .cppm file must be compiled 
for your target device. This tutorial serves as an example of taking 
advantage of C++ modules. The CMakeLists.txt in our tutorial has all the 
instructions needed for building the module automatically:

```cmake
find_package (Vulkan REQUIRED)

# set up Vulkan C++ module
add_library(VulkanCppModule)
add_library(Vulkan::cppm ALIAS VulkanCppModule)

target_compile_definitions(VulkanCppModule
        PUBLIC VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1
)
target_include_directories(VulkanCppModule
        PRIVATE
        "${Vulkan_INCLUDE_DIR}"
)
target_link_libraries(VulkanCppModule
        PUBLIC
        Vulkan::Vulkan
)

set_target_properties(VulkanCppModule PROPERTIES CXX_STANDARD 20)

target_sources(VulkanCppModule
        PUBLIC
        FILE_SET cxx_modules TYPE CXX_MODULES
        BASE_DIRS
        "${Vulkan_INCLUDE_DIR}"
        FILES
        "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
)
```

The VulkanCppModule target only needs to be defined once, then add it to the 
dependency of your consuming project and it will be built automatically, and 
you won't need to also add Vulkan::Vulkan to your project.

```cmake
target_link_libraries (${PROJECT_NAME} Vulkan::cppm)
```

That is all that is required to add Vulkan to any project.

## Window Management
As mentioned before, Vulkan by itself is a platform-agnostic API and does not
include tools for creating a window to display the rendered results. To benefit
from the cross-platform advantages of Vulkan and to avoid the horrors of Win32,
we'll use the [GLFW library](http://www.glfw.org/) to create a window, which
supports Windows, Linux and macOS. There are other libraries available for this
purpose, like [SDL](https://www.libsdl.org/), but the advantage of GLFW is that
it also abstracts away some of the other platform-specific things in Vulkan
besides just window creation.  An unfortunate disadvantage is GLFW doesn't 
work in Android or iOS; it is a desktop-only solution.  SDL does offer 
mobile support; however, mobile windowing support is best done by 
interfacing with the Operating system such as using the JNI in Android.
While mobile is beyond the scope of this initial tutorial, plans exist to 
eventually cover it in detail and [Google has excellent documentation]
(https://developer.android.com/ndk/guides/graphics/getting-started).

### GLM

Unlike DirectX 12, Vulkan does not include a library for linear algebra
operations, so we'll have to download one. [GLM](http://glm.g-truc.net/) is a
nice library that is designed for use with graphics APIs and is also commonly
used with OpenGL.

### Texturing library

Vulkan by itself has no support for reading various texture resources such 
as png, jpeg, or ktx files. However, as this is a large topic it is beyond 
the scope of this tutorial to fully dive into all the various formats.  For 
this tutorial, we will use stb as a dependency for loading up textures.  We 
do recommend investigating ktx to gain full advantage of a texture format 
that is designed for graphics applications in mind.

### Modeling library

Model formats are numerous and expose a lot of details everywhere.  In 
general, with Vulkan and other graphical APIs, the most important things to 
know are vertex information, texture coordinates, and potentially diffuse 
color details.  GLTF is an advanced feature-full model format with easy to 
support features available in a cross platform library.  However, for this 
tutorial, we're going to use tinyobjloader for its pure simplicity.  We 
recommend tinyobjlader library only for small not complex projects.

## Windows

Development in Windows is easiest with Visual Studio. CLion works well with 
Windows as does Android Studio, however, Visual Studio is very popular and 
well-supported, so we'll discuss getting dependencies there. For complete 
C++20 support, you need to use any version greater than 2019. The steps 
outlined below were written for VS 2022.

## Package management
For all platforms, we recommend using a platform management tool. Windows
natively doesn't depend upon package management, so this is a foreign concept.
However, Microsoft has introduced a fantastic package management tool which
does work cross-platform.  VCPkg also includes setting up all required CMake 
settings.  We recommend  following the excellent documentation [here]
(https://learn.microsoft.com/en-us/vcpkg/get_started/get-started?pivots=shell-powershell)
for details on how to use CMake in Windows projects.
This setup allows Windows developers to natively work in Visual Studio using 
CMake and the integration is rather quite good.

Alternatively, [CLion](http://jetbrains.com) natively supports CMakeLists.txt 
projects on all platforms and works/functions exactly like Android Studio. 
It is also a free IDE.

### GLFW

We recommend using vcpkg as mentioned before to install packages, to do that,
 run this from the command line: `vcpkg install glfw3`

If you desire to install without vcpkg, you can find the latest release of 
GLFW on the [official website](http://www.glfw.org/download.html).

In this tutorial, we'll be using the 64-bit binaries, but you can of course also
choose to build in 32-bit mode. In that case make sure to link with the Vulkan
SDK binaries in the `Lib32` directory instead of `Lib`. After downloading it, extract the archive
to a convenient location. I've chosen to create a `Libraries` directory in the
Visual Studio directory under documents.

![](/images/glfw_directory.png)

### GLM

GLM can also be installed with vcpkg like so: vcpkg install glm

Alternatively, GLM is a header-only library, so download the [latest version]
(https://github.com/g-truc/glm/releases) and store it in a convenient 
location. You should have a directory structure similar to the following now:

![](/images/library_directory.png)

### tinyobjloader

Tinyobjloader can be installed with vcpkg like so: vcpkg install tinyobjloader

### Setting up Visual Studio

Now that you've installed all the dependencies, we can set up a basic Visual
Studio project for Vulkan and write a little bit of code to make sure that
everything works.

Start Visual Studio and create a new `Windows Desktop Wizard` project by 
entering a name and pressing `OK`.

![](/images/vs_new_cpp_project.png)

Make sure that `Console Application (.exe)` is selected as an application 
type so that we have a place to print debug messages to, and check `Empty 
Project` to prevent Visual Studio from adding boilerplate code.

![](/images/vs_application_settings.png)

Press `OK` to create the project and add a C++ source file. You should
already know how to do that, but the steps are included here for completeness.

![](/images/vs_new_item.png)

![](/images/vs_new_source_file.png)

Now add the following code to the file. Don't worry about trying to
understand it right now; we're just making sure that you can compile and run
Vulkan applications. We'll start from scratch in the next chapter.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

Let's now configure the project to get rid of the errors. Open the project
properties dialog and ensure that `All Configurations` is selected, because most
of the settings apply to both `Debug` and `Release` mode.

![](/images/vs_open_project_properties.png)

![](/images/vs_all_configs.png)

Go to `C++ -> General -> Additional Include Directories` and press `<Edit...>`
in the dropdown box.

![](/images/vs_cpp_general.png)

Add the header directories for Vulkan, GLFW and GLM:

![](/images/vs_include_dirs.png)

Next, open the editor for library directories under `Linker -> General`:

![](/images/vs_link_settings.png)

And add the locations of the object files for Vulkan and GLFW:

![](/images/vs_link_dirs.png)

Go to `Linker -> Input` and press `<Edit...>` in the `Additional Dependencies`
dropdown box.

![](/images/vs_link_input.png)

Enter the names of the Vulkan and GLFW object files:

![](/images/vs_dependencies.png)

And finally, change the compiler to support C++17 features:

![](/images/vs_cpp17.png)

You can now close the project properties dialog. If you did everything right, 
then you should no longer see any more errors being highlighted in the code.

Finally, ensure that you are actually compiling in 64-bit mode:

![](/images/vs_build_mode.png)

Press `F5` to compile and run the project, and you should see a command prompt
and a window pop up like this:

![](/images/vs_test_window.png)

The number of extensions should be non-zero. Congratulations, you're all set for
[playing with Vulkan](!en/Drawing_a_triangle/Setup/Base_code)!

## Linux

These instructions will be aimed at Ubuntu, Fedora and Arch Linux users, 
but you may be able to follow along by changing the package 
manager-specific commands to the ones that are appropriate for you. You 
should have a compiler that supports C++20 (GCC 7+ or Clang 5+). You'll 
also need `cmake`.  Most of this can be installed via larger packages such 
as build-essentials.

### Vulkan Packages

The most important parts you'll need for developing Vulkan applications on 
Linux are the Vulkan loader, validation layers, and a couple of command-line 
utilities to test whether your machine is Vulkan-capable:

Download the VulkanSDK tarball from [LunarG](https://vulkan.lunarg.com/).  
Place the uncompressed VulkanSDK in a convenient path, and create a symbolic 
link to the latest on like so:

```shell
pushd vulkansdk
tar -xzf vulkansdk-linux-x86_64-1.4.304.1.tgz
ln -s 1.4.304.1 default
```

Then add the following to your ~/.bashrc file so Vulkan's environment 
variables are enabled everywhere:

```shell
source ~/vulkanSDK/default/setup-env.sh
```

If installation  was successful, you should be all set with the Vulkan 
portion. Remember to run `vkcube` and ensure you see the following pop up 
in a window:

![](/images/cube_demo_nowindow.png)

If you receive an error message, then ensure that your drivers are up to date,
include the Vulkan runtime and that your graphics card is supported. See the
[introduction chapter](!en/Introduction) for links to drivers from the major
vendors.

### Ninja
Ninja is a rapid build system that CMake has support for in all 
platforms.  We recommend installing it with `sudo apt install ninja`

### X Window System and XFree86-VidModeExtension
It is possible that these libraries are not on the system, if not, you can 
install them using the following commands:
* `sudo apt install libxxf86vm-dev` or `dnf install libXxf86vm-devel`: 
  Provides an interface to the XFree86-VidModeExtension.
* `sudo apt install libxi-dev` or `dnf install libXi-devel`: Provides an X 
  Window System client interface to the XINPUT extension.

### GLFW

We'll be installing GLFW from the following command:

```bash
sudo apt install libglfw3-dev
```
or
```bash
sudo dnf install glfw-devel
```
or
```bash
sudo pacman -S glfw-wayland # glfw-x11 for X11 users
```

### GLM

It is a header-only library that can be installed from the `libglm-dev` or
`glm-devel` package:

```bash
sudo apt install libglm-dev
```
or
```bash
sudo dnf install glm-devel
```
or
```bash
sudo pacman -S glm
```

### Setting up a CMake project

Now that you have installed all the dependencies, we can set up a basic
CMake project for Vulkan and write a little bit of code to make sure that
everything works.

I will assume that you already have some basic experience with CMake, like 
how variables and rules work. If not, you can get up to speed very quickly with [this tutorial](https://cmake.org/cmake/help/book/mastering-cmake/cmake/Help/guide/tutorial/).

You can now use the code directory in this tutorial as a template for your 
Vulkan projects. Make a copy, rename it to something like `HelloTriangle` 
and remove all the code in `main.cpp`.

You are now all set for [the real adventure](!en/Drawing_a_triangle/Setup/Base_code).

## MacOS

These instructions will assume you are using Xcode and the [Homebrew 
package manager](https://brew.sh/). Also, keep in mind that you will need 
at least macOS version 10.11, and your device needs to support the [Metal 
API](https://en.wikipedia.org/wiki/Metal_(API)#Supported_GPUs).

### Vulkan SDK

The SDK can be downloaded from [the LunarG website](https://vulkan.lunarg.
com/) using the buttons at the bottom of the page. You don't have to create 
an account, but it will give you access to some additional documentation 
that may be useful to you.

![](/images/vulkan_sdk_download_buttons.png)

The SDK version for macOS internally uses [MoltenVK](https://moltengl.com/).
There is no native support for Vulkan on macOS, so what MoltenVK does is 
actually act as a layer that translates Vulkan API calls to Apple's Metal 
graphics framework. With this, you can take advantage of debugging and 
performance benefits of Apple's Metal framework.

After downloading it, extract the contents to a folder of your choice (keep 
in mind you will need to reference it when creating your projects on Xcode).
Inside the extracted folder, in the `Applications` folder you should have 
some executable files that will run a few demos using the SDK. Run the 
`vkcube` executable and you will see the following:

![](/images/cube_demo_mac.png)

### GLFW

As mentioned before, Vulkan by itself is a platform-agnostic API and does 
not include tools for creation a window to display the rendered results. 
We'll use the [GLFW library](http://www.glfw.org/) to create a window, 
which supports Windows, Linux and macOS. There are other libraries 
available for this purpose, like [SDL](https://www.libsdl.org/), but the 
advantage of GLFW is that it also abstracts away some of the other 
platform-specific things in Vulkan besides just window creation.

To install GLFW on macOS, we will use the Homebrew package manager to get 
the `glfw` package:

```bash
brew install glfw
```

### GLM

Vulkan does not include a library for linear algebra operations, so we'll 
have to download one. [GLM](http://glm.g-truc.net/) is a nice library that 
is designed for use with graphics APIs and is also commonly used with OpenGL.

It is a header-only library that can be installed from the `glm` package:

```bash
brew install glm
```

### Setting up Xcode

Now that all the dependencies are installed, we can set up a basic Xcode 
project for Vulkan. Most of the instructions here are essentially a lot of 
"plumbing" so we can get all the dependencies linked to the project. Also, 
keep in mind that during the following instructions whenever we mention the 
folder `vulkansdk` we are referring to the folder where you extracted the 
Vulkan SDK.

We recommend using CMake everywhere, and Apple is no different. An example 
of how to use CMake for Apple can be found [here](https://medium.com/practical-coding/migrating-to-cmake-in-c-and-getting-it-working-with-xcode-50b7bb80ae3d)
We also have documentation for using a cmake project in Apple environments 
at the VulkanSamples project.  It targets both iOS and Desktop Apple.

Once you use CMake with the XCode generator, open the resulting xcode 
project.  If you use the code directory of this tutorial, you can do this 
from the ocmmand line:

```shell
cd code
cmake -G XCode
```

Start Xcode and open the newly created project. On the window that will open 
select Application > Command Line Tool.

![](/images/xcode_new_project.png)

Select `Next`, write a name for the project and for `Language` select `C++`.

![](/images/xcode_new_project_2.png)

Press `Next` and the project should have been created. Now, let's change 
the code in the generated `main.cpp` file to the following code:

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

Keep in mind you are not required to understand all this code is doing yet, 
we are just setting up some API calls to make sure everything is working.

Xcode should already be showing some errors such as libraries it cannot 
find. We will now start configuring the project to get rid of those errors. 
On the *Project Navigator* panel select your project. Open the *Build 
Settings* tab and then:

* Find the **Header Search Paths** field and add a link to 
  `/usr/local/include` (this is where Homebrew installs headers, so the glm 
  and glfw3 header files should be there) and a link to 
  `vulkansdk/macOS/include` for the Vulkan headers.
* Find the **Library Search Paths** field and add a link to 
  `/usr/local/lib` (again, this is where Homebrew installs libraries, so 
  the glm and glfw3 lib files should be there) and a link to 
  `vulkansdk/macOS/lib`.

It should look like so (paths will be different depending on 
where you placed your files):

![](/images/xcode_paths.png)

Now, in the *Build Phases* tab, on **Link Binary With Libraries** we will 
add both the `glfw3` and the `vulkan` frameworks. To make things easier, we 
will be adding the dynamic libraries in the project (you can check the 
documentation of these libraries if you want to use the static frameworks).

* For glfw open the folder `/usr/local/lib` and there you will find a file 
  name like `libglfw.3.x.dylib` ("x" is the library's version number, it 
  might be different depending on when you downloaded the package from 
  Homebrew). Drag that file to the Linked Frameworks and Libraries tab on Xcode.
* For vulkan, go to `vulkansdk/macOS/lib`. Do the same for the both files 
  `libvulkan.1.dylib` and `libvulkan.1.x.xx.dylib` (where "x" will be the 
  version number of the SDK you downloaded).

After adding those libraries, in the same tab on **Copy Files** change 
`Destination` to "Frameworks," clear the sub-path and deselect "Copy only 
when installing." Click on the "+" sign and add all those three frameworks 
here as well.

Your Xcode configuration should look like:

![](/images/xcode_frameworks.png)

The last thing you need to set up is a couple of environment variables. On 
Xcode toolbar go to `Product` > `Scheme` > `Edit Scheme...`, and in the 
`Arguments` tab add the two following environment variables:

* VK_ICD_FILENAMES = `vulkansdk/macOS/share/vulkan/icd.d/MoltenVK_icd.json`
* VK_LAYER_PATH = `vulkansdk/macOS/share/vulkan/explicit_layer.d`

It should look like so:

![](/images/xcode_variables.png)

Finally, you should be all set! Now if you run the project (remembering to 
setting the build configuration to Debug or Release depending on the 
configuration you chose), you should see the following:

![](/images/xcode_output.png)

The number of extensions should be non-zero. The other logs are from the 
libraries, you might get different messages from those depending on your 
configuration.

You are now all set for [the real thing](!en/Drawing_a_triangle/Setup/Base_code).
