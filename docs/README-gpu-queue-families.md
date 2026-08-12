# GPU queue families

This branch lets a caller choose which queue a command buffer is submitted to, and makes the
Vulkan backend use up to three queue families — graphics, compute and transfer — instead of a
single unified one. It is a fork of `release-3.4.14`, published so the approach can be looked at
and argued with. **It is not a merge proposal**, and it has not been rebased onto `main`.

The public surface is two additions:

```c
typedef enum SDL_GPUQueueType
{
    SDL_GPU_QUEUETYPE_GRAPHICS,
    SDL_GPU_QUEUETYPE_COMPUTE,
    SDL_GPU_QUEUETYPE_TRANSFER,
    SDL_GPU_QUEUETYPE_COUNT
} SDL_GPUQueueType;

SDL_GPUCommandBuffer *SDL_AcquireGPUCommandBufferOnQueue(SDL_GPUDevice *device,
                                                         SDL_GPUQueueType queue_type);
```

`SDL_AcquireGPUCommandBuffer()` is unchanged in meaning: it now forwards to the above with
`SDL_GPU_QUEUETYPE_GRAPHICS`. Existing code needs no edits.

Note the name: this is one queue per *family*, not several queues per family. Three families are
requested at device creation, with `queueCount = 1` each.

## How it degrades

The queue types are named by **capability**, not by intended use. What matters to the
implementation is what a queue can execute; roles such as "upload" or "render" belong in the
caller's own abstraction, which maps them onto these.

Capabilities are strictly nested — `TRANSFER ⊂ COMPUTE ⊂ GRAPHICS` — and a role with no
dedicated family of its own falls onto the nearest wider one. Only dedicated families count: a
family advertising `VK_QUEUE_GRAPHICS_BIT` is the same hardware engine that renders, so offloading
onto it would buy nothing.

- compute — `COMPUTE` without `GRAPHICS` (the ACEs on AMD)
- transfer — `TRANSFER` without `GRAPHICS` and without `COMPUTE` (a dedicated DMA engine, SDMA on AMD)

The important consequence: with no dedicated DMA family but a compute family present, uploads land
on compute rather than on graphics. That is still a different engine, so the overlap survives.
"Compute yes, transfer no" is a common configuration.

The whole fallback is decided **once**, at device selection, by how the role → family table is
filled. Nothing downstream — role mapping, command pool key, submission — ever asks whether a
queue exists, and a caller never has to branch on availability. Requesting
`SDL_GPU_QUEUETYPE_TRANSFER` on a device with no separate transfer queue is not an error; the same
code is correct either way and only the amount of overlap changes.

D3D12 and Metal accept the parameter and ignore it. For Metal that is permanent — it has a single
`MTLCommandQueue` by design.

## Ordering

The "command buffers execute in submission order" guarantee documented on `SDL_GPUCommandBuffer`
holds only **within one queue**. Work recorded on `TRANSFER` and work recorded on `GRAPHICS` are
ordered against each other only by whatever the caller arranges — a fence, typically. This is the
one contract change a caller has to absorb.

## What is not done

This is the part worth reading before drawing conclusions from the diff size.

**The second queue carries buffers only.** No texture data goes through it. `VULKAN_UploadToTexture`
ends by transitioning the texture back to its default state, and for a sampler texture that state
is not expressible outside the graphics queue, so the barrier assert fires. The copy itself would
be legal anywhere; it is the return to the default state that is not. Only a texture whose sole
usage is `COMPUTE_STORAGE_*` can be uploaded off the graphics queue, and nothing does that today.

**Textures are `EXCLUSIVE`, each bound to one family.** Which family follows from the usage flags,
via expressibility of the default state: `SAMPLER`, graphics storage and render targets are
graphics-only, while a compute-storage-only texture is also expressible on compute. There is no
ownership transfer anywhere in the branch — every barrier uses `VK_QUEUE_FAMILY_IGNORED` — so a
texture has to stay in one family for its whole life.

One case still violates that: a texture carrying **both** `SAMPLER` and `COMPUTE_STORAGE_*`
defaults to `SAMPLER`, is therefore created on graphics, and compute reaches for it anyway.
Fixing it means `VK_SHARING_MODE_CONCURRENT` for exactly those textures. The rule is written down
at the creation site but deliberately not implemented — `CONCURRENT` on an image usually costs
hardware compression, every frame, even when the second family never touches it.

Buffers do not have that problem and are created `CONCURRENT` whenever more than one distinct
family exists: a buffer has neither layout nor compression, so sharing is essentially free, and
`VK_QUEUE_FAMILY_IGNORED` is then the correct value in barriers, which leaves the existing barrier
code valid as written.

**Mipmap generation and blits stay on graphics.** They are draws. Nothing in SDL routes them —
what stops them is the texture barrier assert, plus whatever type discipline the caller has.

**`submitLock` is a single lock across all queues.** `vkQueueSubmit` needs external synchronisation
per queue, but the same lock also covers the fence pool, the deferred-destroy lists and
defragmentation, none of which split per queue. Submits therefore serialise on the CPU; the
parallelism is on the GPU.

**Compute-storage textures are not exercised at all**, so the compute lane has been used for
dispatch and buffers only.

**D3D12 and Metal received the parameter and nothing else.** No second queue exists there and none
is planned in this branch.

## Barriers

Two changes make barriers valid off the graphics queue.

Buffer usage bits are masked by what the recording queue's family can express, because
`SetMemoryBarrierFlags` turns those bits into pipeline stages and a stage the queue does not have
cannot be named. The mask is a property of the pair *(resource, queue)*, never of the resource —
it must not reach buffer creation, or a buffer would lose `VERTEX` permanently. It is derived from
the family's real `queueFlags` rather than from the role's name, because under fallback a
`TRANSFER` role may sit on a compute family that expresses compute-storage bits perfectly well;
stripping them by name would drop half the make-visible side of the barrier and desynchronise
silently instead of failing.

An empty mask is not zero stages — a barrier naming no stage is invalid too. An empty side is
expressed explicitly: empty destination becomes `BOTTOM_OF_PIPE` (release), empty source becomes
`TOP_OF_PIPE` (acquire).

Textures are checked rather than masked. Buffer usage bits accumulate, so masking removes addends
and leaves the rest correct; a texture mode is a *memory layout* picked as the first match by
priority, so masking would change the answer wholesale and silently leave an image in
copy-destination form while SDL believed it ready to sample. The assert is `SDL_assert_release`,
so it is live in release builds too.

## What was verified

Routing is observable: in debug mode the backend logs the resolved families once, and then the
first submit of each *(thread, queue family)* pair. Per-pair rather than per-family, because under
a threaded pipeline several threads submit into the same family and a per-family log would only
show the first of them.

Create the device with `debug_mode` set and run with `SDL_LOGGING=info` in the environment; SDL's
default log priority is `ERROR`, so the lines are otherwise suppressed.

On an AMD Radeon RX 640 (AMD proprietary driver 20.40.16, Vulkan conformance 1.2.0.2) all three
families come out distinct and dedicated, and the caller's pipeline threads reach all three:

```
GPU queue families: graphics=0 [gfx] compute=1 [comp] (dedicated) transfer=2 [xfer] (dedicated) (3 distinct)
GPU first submit: thread 18920 -> queue family 0
GPU first submit: thread 18200 -> queue family 0
GPU first submit: thread 18200 -> queue family 2
GPU first submit: thread 17456 -> queue family 1
GPU first submit: thread 4728 -> queue family 0
```

The middle two lines are the reason the logging is per-pair: one thread feeds both the graphics
and the transfer family. A per-family log would have printed family 2 once and said nothing about
who was feeding it, which is exactly the question worth answering.

The mask class is printed next to each index so the line checks itself: the class is derived from
the family's flags, so `transfer=2 [xfer]` confirms family 2 really is a copy family, while
`transfer=2 [gfx]` would mean the derivation is wrong.

Throughput was measured and did improve, but no numbers are quoted: the measurement comes from a
single machine with switchable graphics and would not generalise. The claim here is that the work
genuinely overlaps, which the routing log and the caller's own frame instrumentation show
directly — not a factor.

## Reading the patch

The nine commits are meant to be read in order. The first is a prerequisite rather than a
refinement: without stage masking, the very first submit on a copy queue emits a barrier naming
`VERTEX_INPUT` and loses the device. The first two commits are behaviour-neutral on their own.
