---
layout: post
title: Offscreen rendering in Vulkan
---

In the <a href="https://bynmz.github.io/2024/02/05/simple-ui-with-dear-imgui/">last post</a> we looked at integrating a simple UI with Dear Imgui using my <a href="https://github.com/bynmz/mygraphicsengine" target="_blank">vulkan renderer</a>. Now we shall look at offscreen rendering. This can be used for various purposes such as post-processing effects, rendering to textures for later use, shadow mapping, reflections etc. In this example we shall look at offscreen rendering for reflections. After having a decent understanding of the Vulkan specification and having gone through the renderer sample code provided, you can follow along or use this as a general guide on how you can implement this in your own projects. The intention of this is not to serve as a tutorial for beginners but as a guide for someone with a good enough understanding of the Vulkan specification and some modern C++ knowledge.

To start, I encapsulated the following into a new `offScreen.cpp` class; 


In the constructor I initialize the offscreen renderer with a specified device and set the width and height of the offscreen framebuffer. This also calls the `init()` function to perform initialization tasks.
```cpp
OffScreen::OffScreen(Device &deviceRef)
: device{deviceRef}
{
    offscreenPass.width = FB_DIM;
    offscreenPass.height = FB_DIM;
    init();
}
```
The init() method calls various functions to create the necessary Vulkan objects for offscreen rendering, such as images, image views, samplers, depth resources, render passes and framebuffers. The descriptor is also updated with the image layout for later use in a descriptor set.
```cpp
void OffScreen::init()
{
    createImage();
    createImageView();
    createSampler();
    createDepthResources();
    createRenderPass();
    createFramebuffers();
    updateDescriptor();
}
```
The destructor destroys Vulkan objects created during initialization, such as samplers, image views, images, memory, framebuffers and render passes.
```cpp
OffScreen::~OffScreen()
{
  // Sampler
  vkDestroySampler(device.device(), offscreenPass.sampler, nullptr);

  // Color attachment
  vkDestroyImageView(device.device(), offscreenPass.color.view, nullptr);
  vkDestroyImage(device.device(), offscreenPass.color.image, nullptr);
  vkFreeMemory(device.device(), offscreenPass.color.mem, nullptr);

  // vkDestroyBuffer(device.device(), offscreenPass.color.staging, nullptr);
  vkDestroyFramebuffer(device.device(), offscreenPass.frameBuffer, nullptr);

  vkDestroyRenderPass(device.device(), offscreenPass.renderPass, nullptr);

  // Depth attachment
  vkDestroyImageView(device.device(), offscreenPass.depth.view, nullptr);
  vkDestroyImage(device.device(), offscreenPass.depth.image, nullptr);
  vkFreeMemory(device.device(), offscreenPass.depth.mem, nullptr);
}
```
The updateDescriptor() method fills a descriptor structure with information about the offscreen color image, including its layout and sampler.
```cpp
void OffScreen::updateDescriptor() {
  offscreenPass.descriptor.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
  offscreenPass.descriptor.imageView = offscreenPass.color.view;
  offscreenPass.descriptor.sampler = offscreenPass.sampler;
}
```
Next, a Vulkan image object representing the offscreen color attachment is created. The format, extend, usage and memory properties of the image are also specified.
```cpp
void OffScreen::createImage() {
  		// Color attachment
		VkImageCreateInfo image{};
    image.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
		image.imageType = VK_IMAGE_TYPE_2D;
		image.format = FB_COLOR_FORMAT;
		image.extent.width = offscreenPass.width;
		image.extent.height = offscreenPass.height;
		image.extent.depth = 1;
		image.mipLevels = 1;
		image.arrayLayers = 1;
		image.samples = VK_SAMPLE_COUNT_1_BIT;
		image.tiling = VK_IMAGE_TILING_OPTIMAL;
		// We will sample directly from the color attachment
		image.usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;

    device.createImageWithInfo(
        image,
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
        offscreenPass.color.image,
        offscreenPass.color.mem);
}
```
The Vulkan image view object is created for the offscreen color image. The format and subresource range of the image view is also specified.
```cpp
void OffScreen::createImageView() {
    VkImageViewCreateInfo viewInfo{};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = offscreenPass.color.image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = FB_COLOR_FORMAT;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    if (vkCreateImageView(device.device(), &viewInfo, nullptr, &offscreenPass.color.view) !=
        VK_SUCCESS) {
      throw std::runtime_error("failed to create texture image view!");
    }
}
```
Create a Vulkan sampler object for sampling the offscreen color image. Parameters such as filtering, addressing mode and mipmapping are also specified.
```cpp
void OffScreen::createSampler() {
      VkSamplerCreateInfo samplerInfo{};
      samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
      samplerInfo.magFilter = VK_FILTER_LINEAR;
      samplerInfo.minFilter = VK_FILTER_LINEAR;

      samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
      samplerInfo.addressModeV = samplerInfo.addressModeU;
      samplerInfo.addressModeW = samplerInfo.addressModeU;

      samplerInfo.anisotropyEnable = VK_TRUE;
      samplerInfo.maxAnisotropy = 1.0f;
      samplerInfo.borderColor = VK_BORDER_COLOR_FLOAT_OPAQUE_WHITE;
      samplerInfo.unnormalizedCoordinates = VK_FALSE;

      // these fields useful for percentage close filtering for shadow maps
      samplerInfo.compareEnable = VK_FALSE;
      samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;

      samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
      samplerInfo.mipLodBias = 0.0f;
      samplerInfo.minLod = 0.0f;
      samplerInfo.maxLod = 1.0f;

      if (vkCreateSampler(device.device(), &samplerInfo, nullptr, &offscreenPass.sampler) != VK_SUCCESS) {
          throw std::runtime_error("failed to create texture sampler!");
      }
}
```
A Vulkan render pass object is created for defining the sequence of rendering operations. Attachment descriptions for color and depth attachments are specified. Subpasses and dependencies between rendering operations are defined.
```cpp
void OffScreen::createRenderPass() {
  VkAttachmentDescription depthAttachment{};
  depthAttachment.format = findDepthFormat();
  depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
  depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
  depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
  depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
  depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
  depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
  depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

  VkAttachmentReference depthAttachmentRef{};
  depthAttachmentRef.attachment = 1;
  depthAttachmentRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

  VkAttachmentDescription colorAttachment = {};
  colorAttachment.format = FB_COLOR_FORMAT;
  colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
  colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
  colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
  colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
  colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
  colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
  colorAttachment.finalLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;

  VkAttachmentReference colorAttachmentRef = {};
  colorAttachmentRef.attachment = 0;
  colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

  VkSubpassDescription subpass = {};
  subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
  subpass.colorAttachmentCount = 1;
  subpass.pColorAttachments = &colorAttachmentRef;
  subpass.pDepthStencilAttachment = &depthAttachmentRef;

  // Use subpass dependencies for layout transitions
  std::array<VkSubpassDependency, 2> dependencies;

  dependencies[0].srcSubpass = VK_SUBPASS_EXTERNAL;
  dependencies[0].dstSubpass = 0;
  dependencies[0].srcStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
  dependencies[0].dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT;
  dependencies[0].srcAccessMask = VK_ACCESS_NONE_KHR;
  dependencies[0].dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
  dependencies[0].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

  dependencies[1].srcSubpass = 0;
  dependencies[1].dstSubpass = VK_SUBPASS_EXTERNAL;
  dependencies[1].srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT;
  dependencies[1].dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
  dependencies[1].srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
  dependencies[1].dstAccessMask = VK_ACCESS_MEMORY_READ_BIT;
  dependencies[1].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

  std::array<VkAttachmentDescription, 2> attachments = {colorAttachment, depthAttachment};
  VkRenderPassCreateInfo renderPassInfo = {};
  renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
  renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
  renderPassInfo.pAttachments = attachments.data();
  renderPassInfo.subpassCount = 1;
  renderPassInfo.pSubpasses = &subpass;
  renderPassInfo.dependencyCount = static_cast<uint32_t>(dependencies.size());
  renderPassInfo.pDependencies = dependencies.data();

  if (vkCreateRenderPass(device.device(), &renderPassInfo, nullptr, &offscreenPass.renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
  }
}
```
Create Vulkan framebuffer objects for rending into the offscreen color and depth attachments. This associates the framebuffers with the render pass.
```cpp
void OffScreen::createFramebuffers() {
  std::array<VkImageView, 2> attachments = {offscreenPass.color.view, offscreenPass.depth.view};

  VkFramebufferCreateInfo framebufferInfo = {};
  framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
  framebufferInfo.renderPass = offscreenPass.renderPass;
  framebufferInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
  framebufferInfo.pAttachments = attachments.data();
  framebufferInfo.width = offscreenPass.width;
  framebufferInfo.height = offscreenPass.height;
  framebufferInfo.layers = 1;

  if (vkCreateFramebuffer(
          device.device(),
          &framebufferInfo,
          nullptr,
          &offscreenPass.frameBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to create framebuffer!");
  }
}
```
Create a Vulkan image object representing the offscreen depth attachment, create a Vulkan image view object for the depth attachment and specifies the format, extent, usage and memory properties of the depth image.
```cpp
void OffScreen::createDepthResources() {
  VkFormat depthFormat = findDepthFormat();
    // Color attachment
    VkImageCreateInfo imageInfo{};
    imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    imageInfo.imageType = VK_IMAGE_TYPE_2D;
    imageInfo.extent.width = offscreenPass.width;
    imageInfo.extent.height = offscreenPass.height;
    imageInfo.extent.depth = 1;
    imageInfo.mipLevels = 1;
    imageInfo.arrayLayers = 1;
    imageInfo.format = depthFormat;
    imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
    imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageInfo.usage = VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT;
    imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;

    device.createImageWithInfo(
        imageInfo,
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
        offscreenPass.depth.image,
        offscreenPass.depth.mem);

    // Depth stencil attachment
    VkImageViewCreateInfo depthStencilView{};
    depthStencilView.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    depthStencilView.image = offscreenPass.depth.image;
    depthStencilView.viewType = VK_IMAGE_VIEW_TYPE_2D;
    depthStencilView.format = depthFormat;
    depthStencilView.subresourceRange.aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT;
		if (depthFormat >= VK_FORMAT_D16_UNORM_S8_UINT) {
			depthStencilView.subresourceRange.aspectMask |= VK_IMAGE_ASPECT_STENCIL_BIT;
		}
    depthStencilView.subresourceRange.baseMipLevel = 0;
    depthStencilView.subresourceRange.levelCount = 1;
    depthStencilView.subresourceRange.baseArrayLayer = 0;
    depthStencilView.subresourceRange.layerCount = 1;

    if (vkCreateImageView(device.device(), &depthStencilView, nullptr, &offscreenPass.depth.view) != VK_SUCCESS) {
      throw std::runtime_error("failed to create texture image view!");
    }
}

VkFormat OffScreen::findDepthFormat() {
  return device.findSupportedFormat(
      {VK_FORMAT_D32_SFLOAT, VK_FORMAT_D32_SFLOAT_S8_UINT, VK_FORMAT_D24_UNORM_S8_UINT},
      VK_IMAGE_TILING_OPTIMAL,
      VK_FORMAT_FEATURE_DEPTH_STENCIL_ATTACHMENT_BIT);
}
```
In the renderer class `renderer.cpp`, we can create an offscreen object.
{% raw %}
```cpp
void Renderer::createOffScreen() {
    OffScreen = std::make_unique<OffScreen>(device);
}
```
To begin and end the offscreen renderpass
```cpp
void Renderer::beginOffScreenRenderPass(VkCommandBuffer commandBuffer) {
  assert(isFrameStarted && "Can't call beginSwapChainRenderPass if frame is not in progress");
  assert(
      commandBuffer == getCurrentCommandBuffer() &&
      "Can't begin render pass on command buffer from a different frame");
  VkRenderPassBeginInfo renderPassInfo{};
  renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
  renderPassInfo.renderPass = OffScreen->offscreenPass.renderPass;
  renderPassInfo.framebuffer = OffScreen->offscreenPass.frameBuffer;
  renderPassInfo.renderArea.offset = {0, 0};
  renderPassInfo.renderArea.extent.width = OffScreen->offscreenPass.width;
  renderPassInfo.renderArea.extent.height = OffScreen->offscreenPass.height;
  clearValues[0].color = {0.01f, 0.01f, 0.01f, 1.0f};
  clearValues[1].depthStencil = {1.0f, 0};
  renderPassInfo.clearValueCount = static_cast<uint32_t>(clearValues.size());
  renderPassInfo.pClearValues = clearValues.data();

  vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);

  VkViewport viewport{};
  viewport.x = 0.0f;
  viewport.y = 0.0f;
  viewport.width = static_cast<float>(OffScreen->offscreenPass.width);
  viewport.height = static_cast<float>(OffScreen->offscreenPass.height);
  viewport.minDepth = 0.0f;
  viewport.maxDepth = 1.0f;
  VkRect2D scissor{{0, 0}, {OffScreen->offscreenPass.width, OffScreen->offscreenPass.height}};
  vkCmdSetViewport(commandBuffer, 0, 1, &viewport);
  vkCmdSetScissor(commandBuffer, 0, 1, &scissor);
}

void Renderer::endOffScreenRenderPass(VkCommandBuffer commandBuffer) {
  assert(isFrameStarted && "Can't call endOffScreenRenderPass if frame is not in progress");
  assert(
      commandBuffer == getCurrentCommandBuffer() &&
      "Can't end render pass on command buffer from a different frame");
  vkCmdEndRenderPass(commandBuffer);
}

```
{% endraw %}
Now we can call these methods in our app `App3D.cpp`. I use a global descriptor pool, descriptor set layout, and uniform buffer objects (UBOs) for each frame in flight. If you are not familiar with these concepts you can read more about them <a href="https://vulkan-tutorial.com/Uniform_buffers/Descriptor_pool_and_sets" target="_blank">here</a>.

In the interest of keeping this post short, the code for implementing them can be found <a href="https://gist.github.com/bynmz/35254c4a22aa832412c1fabefb71ebce" target="_blank">here</a>.

Back to our app `App3D.cpp`

Initialize a RenderSystem object for our pipeline creation and rendering. This expects a device, a swapchain/ offscreen renderpass and a global descriptor set layout
```cpp
void App3D::loop() {
  RenderSystem3D objMirrored{
      device,
      Renderer.getOffScreenRenderPass(),
      globalSetLayout->getDescriptorSetLayout()};
  RenderSystem3D objSystem{
      device,
      Renderer.getSwapChainRenderPass(),
      globalSetLayout->getDescriptorSetLayout()};
  PointLightSystem pointLightMirrored{
      device,
      Renderer.getOffScreenRenderPass(),
      globalSetLayout->getDescriptorSetLayout()};
  PointLightSystem pointLightSystem{
      device,
      Renderer.getSwapChainRenderPass(),
      globalSetLayout->getDescriptorSetLayout()};
  MirrorSystem mirrorSystem{
      device,
      Renderer.getSwapChainRenderPass(),
      globalSetLayout->getDescriptorSetLayout()
  };

    ...

  // Main rendering loop
  while (!Window.shouldClose()) {
    ...
      // Begin offscreen rendering pass for mirror effects.
      Renderer.beginOffScreenRenderPass(commandBuffer);
      objMirrored.renderGameObjects(frameInfo);
      pointLightMirrored.render(frameInfo);
      Renderer.endOffScreenRenderPass(commandBuffer);

      // Begin rendering to the swap chain.
      Renderer.beginSwapChainRenderPass(commandBuffer);

      // Render game objects and lights. order here matters
      objSystem.renderGameObjects(frameInfo);
      pointLightSystem.render(frameInfo);      

      // Render mirrored objects and mirror planes.
      mirrorSystem.renderMirrorPlane(frameInfo);
      
      // End rendering to the swap chain and end the fram and present it to the screen.
      Renderer.endSwapChainRenderPass(commandBuffer);
      Renderer.endFrame();
    ...
}
```
Use the global descriptor set layout in our app during the pipeline creation stage.

`mirror_system.cpp`
```cpp
void MirrorSystem::createPipelineLayout(VkDescriptorSetLayout offscreenSetLayout) {
  VkPushConstantRange pushConstantRange{};
  pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT;
  pushConstantRange.offset = 0;
  pushConstantRange.size = sizeof(MirrorPushConstants);

  renderSystemLayout = 
      DescriptorSetLayout::Builder(Device)
          .addBinding(
            0,
            VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
            VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT)
          .addBinding(1, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, VK_SHADER_STAGE_FRAGMENT_BIT)
          .build();

  std::vector<VkDescriptorSetLayout> descriptorSetLayouts{
    offscreenSetLayout,
    renderSystemLayout->getDescriptorSetLayout()};

  VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
  pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
  pipelineLayoutInfo.setLayoutCount = static_cast<uint32_t>(descriptorSetLayouts.size());
  pipelineLayoutInfo.pSetLayouts = descriptorSetLayouts.data();
  pipelineLayoutInfo.pushConstantRangeCount = 1;
  pipelineLayoutInfo.pPushConstantRanges = &pushConstantRange;
  if (vkCreatePipelineLayout(Device.device(), &pipelineLayoutInfo, nullptr, &pipelineLayout) !=
      VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
  }
}

void MirrorSystem::createPipeline(VkRenderPass renderPass) {
  assert(pipelineLayout != nullptr && "Cannot create pipeline before pipeline layout");

  PipelineConfigInfo pipelineConfig{};
  Pipeline::defaultPipelineConfigInfo(pipelineConfig);

  pipelineConfig.renderPass = renderPass;
  pipelineConfig.pipelineLayout = pipelineLayout;
  Pipeline = std::make_unique<Pipeline>(
      Device,
      "shaders/mirror.vert.spv",
      "shaders/mirror.frag.spv",
      pipelineConfig);
}
```
Shaders:
<a href="https://gist.github.com/bynmz/4b52af0f38d3adb532433aded186b32e" target="_blank">mirror.vert</a> /
<a href="https://gist.github.com/bynmz/60ed8c7408216b4c885a43c59da13dde" target="_blank">mirror.frag</a>

With this all in place we are now able to render reflections in our app with Vulkan 🎉.

![](/images/reflections.png)