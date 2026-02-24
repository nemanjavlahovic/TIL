# Replace Polling Loops with AsyncStream

If you already have a push-based source (like `NWPathMonitor`), don't wrap it in a polling loop. Expose an `AsyncStream` instead.

## The Problem

```swift
// BAD: wakes every 1 second for the entire app lifetime
private func setupNetworkObserver() {
    networkObserverTask = Task {
        var wasConnected = NetworkMonitor.shared.isConnected
        while !Task.isCancelled {
            try? await Task.sleep(for: .seconds(1))  // pointless wake
            let nowConnected = NetworkMonitor.shared.isConnected
            if !wasConnected && nowConnected { /* reconnect */ }
            wasConnected = nowConnected
        }
    }
}
```

`NetworkMonitor` already uses `NWPathMonitor` which pushes updates. The polling wrapper defeats the entire purpose and wastes CPU/battery.

## The Fix

Add an `AsyncStream` to the monitor:

```swift
final class NetworkMonitor {
    private var continuations: [UUID: AsyncStream<Bool>.Continuation] = [:]

    func connectivityUpdates() -> AsyncStream<Bool> {
        let id = UUID()
        let (stream, continuation) = AsyncStream<Bool>.makeStream()
        continuations[id] = continuation
        continuation.onTermination = { [weak self] _ in
            self?.continuations.removeValue(forKey: id)
        }
        return stream
    }
}
```

Consume it with `for await`:

```swift
for await nowConnected in NetworkMonitor.shared.connectivityUpdates() {
    if !wasConnected && nowConnected { /* reconnect */ }
    wasConnected = nowConnected
}
```

Zero wakeups when nothing changes. Instant reaction when connectivity flips.
