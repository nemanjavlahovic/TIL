# Nuke: Always Downsample with ImageProcessors.Resize

Nuke handles fetching and caching, but without telling it what size you actually need, it caches the **full-resolution decoded bitmap**. Adding `ImageProcessors.Resize` makes Nuke downsample *before* caching.

## The Problem

A 40px avatar loaded from a 400x400 source image:
- **Without Resize**: Nuke decodes and caches a 400x400x4 byte bitmap = **640 KB**
- **With Resize**: Nuke decodes, downsamples, and caches a 40x40x4 byte bitmap = **6.4 KB**

That's **100x more memory** than needed per avatar. In a feed with 20 cards, each with an avatar + 4 item images at full resolution, you can easily burn 100+ MB of decoded image memory for content that displays at a fraction of the size.

## The Fix

Pass `ImageProcessors.Resize` to every `LazyImage`:

```swift
struct RMAvatar: View {
    let url: String?
    var size: CGFloat = 40

    private var processors: [any ImageProcessing] {
        let targetSize = CGSize(
            width: size * UIScreen.main.scale,
            height: size * UIScreen.main.scale
        )
        return [ImageProcessors.Resize(size: targetSize, contentMode: .aspectFill)]
    }

    var body: some View {
        LazyImage(url: url.flatMap { URL(string: $0) }) { state in
            // ...
        }
        .processors(processors)
        .frame(width: size, height: size)
        .clipShape(Circle())
    }
}
```

For pipeline-level preloading (e.g. `RMOutfitImageGrid`), use `ImageRequest`:

```swift
var request = ImageRequest(url: url)
request.processors = [ImageProcessors.Resize(size: thumbnailSize, contentMode: .aspectFit)]
let response = try await pipeline.image(for: request)
```

## Key Details

- Always multiply by `UIScreen.main.scale` — you want **pixel** size, not point size
- `.aspectFill` for avatars/covers (will be clipped), `.aspectFit` for product images (need to see the whole thing)
- Nuke's memory cache keys include processors, so the same URL at different sizes caches separately — this is correct behavior
- The disk cache stores the original compressed data by default (`DataCache`), so the full-res source is still available if you need it at a different size later

## References

- [Nuke docs: Image Processing](https://kean.blog/nuke/guides/image-processing)
- [WWDC 2018: Images and Graphics Best Practices](https://developer.apple.com/videos/play/wwdc2018/219/)
