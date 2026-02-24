# LazyVStack Resets @State When Cells Are Recycled

`LazyVStack` releases cells from memory when they scroll far offscreen and recreates them when they scroll back. This means `@State` is reset to its initial value on every recycle.

## The Problem

```swift
struct FeedCardAnimationWrapper<Content: View>: View {
    @State private var isVisible = false  // resets to false on recycle!

    var body: some View {
        content()
            .opacity(isVisible ? 1 : 0)
            .onAppear {
                withAnimation(.easeOut(duration: 0.4)) {
                    isVisible = true  // re-animates every time cell reappears
                }
            }
    }
}
```

When 20 cells simultaneously re-enter the viewport (scrolling back up), you get 20 parallel opacity+scale animations. This causes visible jank.

## The Fix

Track whether the animation has already played:

```swift
@State private var isVisible = false
@State private var hasAnimated = false

var body: some View {
    content()
        .opacity(isVisible ? 1 : 0)
        .onAppear {
            if hasAnimated {
                isVisible = true  // instant, no animation
                return
            }
            hasAnimated = true
            withAnimation(.easeOut(duration: 0.4)) {
                isVisible = true
            }
        }
}
```

## Key Detail

`@State` in `LazyVStack` children is **not** like `@State` in a regular `VStack`. Regular `VStack` keeps all children in memory, so `@State` persists. `LazyVStack` creates and destroys views as needed — the `@State` identity is tied to the view's lifetime, not the data's.
