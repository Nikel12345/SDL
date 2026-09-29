# Vulkan version requirement

This change makes the Vulkan API version declared through `SDL_GPUVulkanOptions` a requirement
for the physical device, not just a ceiling. It is independent of the queue family work on the
same branch.

## The problem

`vulkan_api_version` ends up only in `VkApplicationInfo::apiVersion`. For the instance that is an
upper bound: the version a device actually runs under is `min(declared, device)`. Upstream never
reads the device's own `apiVersion`, so a device below the declared version is selected like any
other and quietly works under a lower contract. Two things break for it without an error:

- shaders compiled for a newer SPIR-V version are not valid on it;
- the `VkPhysicalDeviceVulkan1xFeatures` structs that SDL chains into `vkCreateDevice` whenever
  1.2 or later is declared are unknown to it.

## What changes

During device selection a device whose `apiVersion` is below the declared one is skipped, the same
way an unsuitable device is. Only major and minor are compared; patch and variant carry no
capabilities.

The check is active only when the caller passes `SDL_GPUVulkanOptions`. Without them SDL declares
1.0 and selection behaves exactly as upstream.

On a machine with two GPUs, an integrated one below the requirement is now excluded outright, and
the discrete one is picked because it qualifies rather than because it happens to rank higher.

## When no device qualifies

If every device is rejected for its version, the one with the highest version is named once. With
1.9 declared on a GeForce GTX 1050 whose driver (560.94) reports 1.3:

```
Vulkan 1.9 required, but 'NVIDIA GeForce GTX 1050' supports only 1.3 (update the GPU driver)
```

This goes to `SDL_LogError` in the `GPU` category instead of `SDL_SetError`: once every backend
has declined, `SDL_CreateGPUDevice` replaces the error with a generic "No supported SDL_GPU
backend found!", and the reason would be lost.
