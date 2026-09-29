# Claiming a minimized window

On Vulkan, `SDL_ClaimWindowForGPUDevice` returned `true` for a window that was minimized at claim
time, but did not actually claim it. Every later swapchain call then failed as if the window had
never been claimed, and the window stayed unusable after it was restored. This change makes such a
window claimed normally. It is independent of the queue family work on the same branch.

## How it happens

The first `ShowWindow` of a Windows process uses the show mode from the `STARTUPINFO` that the
launching process passed in. A game started from a shortcut set to "Run: Minimized", or by a tool
that starts processes minimized, therefore gets its first window created minimized, with a client
area of 0x0. A second window created later in the same process is shown normally, which makes the
problem look as if it were tied to the first window.

For a minimized window the NVIDIA driver reports a surface extent of 0x0, and swapchain creation
answers "try again later". `VULKAN_ClaimWindow` handled that answer by setting
`needsSwapchainRecreate` on its window data and returning `true`, but it never stored the window
data in the window's properties or in the renderer's list of claimed windows, and did not install
the resize watch. Everything that runs after the claim looks the window data up in those properties
and finds nothing:

```
SDL_GetGPUSwapchainTextureFormat  -> SDL_GPU_TEXTUREFORMAT_INVALID
SDL_SetGPUSwapchainParameters     -> "Cannot set swapchain parameters on unclaimed window!"
SDL_AcquireGPUSwapchainTexture    -> "Cannot acquire a swapchain texture from an unclaimed window!"
```

The flag that asked for a retry lived in window data that nothing referenced, so no retry ever
happened.

The same code is present in 3.2.4, 3.4.14, 3.4.16 and upstream `main` as of 2026-09-26.

## What changes

When swapchain creation answers "try again later" during a claim, the window is registered exactly
as on success, with `needsSwapchainRecreate` set. From then on it is handled like a window that was
minimized while running, which SDL already supports: acquiring a swapchain texture succeeds with a
`NULL` texture while the extent is zero, and the swapchain is created on the first acquire after the
window is restored.

## What was verified

A minimal program creates a window, a Vulkan device with `debug_mode` and the Khronos validation
layer, claims the window and acquires three swapchain textures, restoring the window after the
first. GeForce GTX 1050, driver 560.94:

| step | launched normally | launched minimized |
|---|---|---|
| window after creation | 320x240 | minimized, client 0x0 |
| claim, swapchain format | `true`, 12 | `true`, 12 (was `INVALID`) |
| `SDL_SetGPUSwapchainParameters` | `true` | `true` |
| acquire while minimized | 320x240 | `true`, `NULL` texture |
| acquire after restore | 320x240 | 320x240 |

The validation layer reported nothing in either run.
