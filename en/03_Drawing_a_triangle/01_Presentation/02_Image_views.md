To use any `VkImage`, including those in the swap chain, in the render pipeline
we have to create a `VkImageView` object. An image view is quite literally a
view into an image. It describes how to access the image and which part of the
image to access, for example, if it should be treated as a 2D texture depth
texture without any mipmapping levels.

In this chapter we'll write a `createImageViews` function that creates a basic
image view for every image in the swap chain so that we can use them as color
targets later on.

First add a class member to store the image views in:

```c++
std::vector<vk::raii::ImageView> swapChainImageViews;
```

Create the `createImageViews` function and call it right after swap chain
creation.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

The parameters for image view creation are specified in a
`VkImageViewCreateInfo` structure. The first few parameters are the flags, 
this isn't needed in our case, we'll add the images in the upcoming for loop.
Next, we specify that we're rendering to a 2d screen.  If we were wanting 
to render to a 3d screen or cube map, those are also options as is a 1d 
screen.  As you can probably guess, we'd want a 2d render target in most 
cases when we're rendering to a screen.

Next, we specify the image format; this is how the colorspace 
components are configured so you get the right color format in your 
renders. Next, components aren't needed for our swap chain, we'll talk 
about them in a bit though. The last variable is the SubResource range, 
which is necessary, and we'll talk about shortly.

```c++
void createImageViews() {
    vk::ImageViewCreateInfo imageViewCreateInfo( {}, {}, vk::ImageViewType::e2D, swapChainImageFormat, {}, {} );
}
```

The `components` field allows you to swizzle the color channels around. For
example, you can map all the channels to the red channel for a monochrome
texture. You can also map constant values of `0` and `1` to a channel. In our
case we'll stick to the default mapping by accepting the constructed 
defaults, but here's how to explicitly do it:

```c++
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

The `subresourceRange` field describes what the image's purpose is and which
part of the image should be accessed. Our images will be used as color targets
without any mipmapping levels or multiple layers.

```c++
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

An easy way to specify all of that is this single line:

```c++
vk::ImageViewCreateInfo imageViewCreateInfo( {}, {}, vk::ImageViewType::e2D, swapChainImageFormat, {}, { vk::ImageAspectFlagBits::eColor, 0, 1, 0, 1 } );
```

If you were working on a stereographic 3D application, then you would create a
swap chain with multiple layers. You could then create multiple image views for
each image representing the views for the left and right eyes by accessing
different layers.

The maximum number of multiple image views you should expect graphics cards 
to handle is 16. This configuration covers most standard use cases, VR/AR 
headsets typically require no more than four simultaneous views. However, 
lightfield displays and CAVE displays might require a different solution as 
their view requirements can number in the thousands for simultaneous views.  
Those exotic requirements for rendering are beyond the scope of this 
tutorial, but even those use cases can be rendered to with these same 
structures and techniques we describe here.

Next, set up the loop that iterates over all the swap chain images and add
them to our structure.

```c++
for (auto image : swapChainImages) {
    imageViewCreateInfo.image = image;
}
```

Creating the image view is now a matter of calling `vkCreateImageView`:

```c++
for (auto image : swapChainImages) {
    imageViewCreateInfo.image = image;
    swapChainImageViews.emplace_back( *device, imageViewCreateInfo );
}
```

An image view is sufficient to start using an image as a texture, but it's not
quite ready to be used as a render target just yet. That requires one more 
step, known as a framebuffer. But first, we'll have to set up the
graphics pipeline.

[C++ code](/code/07_image_views.cpp)
