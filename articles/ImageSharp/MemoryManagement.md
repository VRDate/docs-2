#Memory Management

### ImageSharp seems to retain ~300-400 MB of managed memory even after disposing all my images. Is this a memory leak?
By default, ImageSharp uses [ArrayPool-s](http://adamsitnik.com/Array-Pool/) for performance reasons, however this behavior is fully configurable. All large buffers are managed by the [](xref:SixLabors.Memory.MemoryAllocator?displayProperty=name) implementation associated to [](xref:SixLabors.ImageSharp.Configuration?displayProperty=name)-s [](xref:SixLabors.ImageSharp.Configuration.MemoryAllocator?displayProperty=name) property. We are using [](xref:SixLabors.Memory.ArrayPoolMemoryAllocator?displayProperty=name) by default, in order to utilize the benefits of array pooling:
- Less pressure on GC, because buffers are being reused most of the time
- Reduced [LOH fragmentation](https://blogs.msdn.microsoft.com/maoni/2016/05/31/large-object-heap-uncovered-from-an-old-msdn-article/)
- When working with unclean buffers is acceptable, we can spare on GC-s array cleaning behavior too

**Summary**: pooling helps us to *reduce CPU work and increase throughput for the cost of a larger memory footprint.*

### Working in memory constrained environments
Sometimes having ~300 MB memory footprint is not an option. Let's mention a few cases:
- When horizontal scaling is achieved by having multiple memory constrained containers in a cloud environment.
- Mobile applications.

Before scaling down pooling behavior because of unwanted `OutOfMemoryException`-s in a cloud or desktop environment:
- Keep in mind that image processing is a *memory intensive* application! This may affect your scaling strategy. We don't recommend using containers with 1 GB or smaller memory limit!
- Make sure that you are running your service in a **64 bit process**!

There are several pre-defined factory methods to create an [](xref:SixLabors.Memory.ArrayPoolMemoryAllocator?displayProperty=name) instance for memory constrained environments. For example [ArrayPoolMemoryAllocator.CreateWithModeratePooling](xref:SixLabors.Memory.ArrayPoolMemoryAllocator.CreateWithModeratePooling) might be suitable in most constrained situations:
```cs
Configuration.Default.MemoryAllocator = ArrayPoolMemoryAllocator.CreateWithModeratePooling();
```
Of course, you may also configure a MemoryAllocator on your own [](xref:SixLabors.ImageSharp.Configuration?displayProperty=name) instance.

You can find [benchmark results in the original PR](https://github.com/SixLabors/ImageSharp/pull/475) which may help to select you a configuration, but they are bit outdated, because our throughput got better since then!

### Releasing pools programatically 
If your application uses ImageSharp sporadically (eg. generating some images on startup, or on other non-frequent use-cases), you may want to release the retained pools using [ReleaseRetainedResources](xref:SixLabors.Memory.MemoryAllocator.ReleaseRetainedResources):
```cs
Configuration.Default.MemoryAllocator.ReleaseRetainedResources();
```

### Using multiple MemoryAllocator instances in the same process
You need to create and maintain your own [](xref:SixLabors.ImageSharp.Configuration?displayProperty=name) instance, setting a specific MemoryAllocator on it. It's possible to pass custom Configuration instances to methods accross our whole API.