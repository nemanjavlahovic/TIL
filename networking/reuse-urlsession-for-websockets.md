# Reuse URLSession for WebSocket Connections

Never create a new `URLSession` per WebSocket connection or reconnect attempt. `URLSession` is heavyweight — it creates a TCP/TLS stack, connection pools, and a background delegate queue.

## The Problem

```swift
// BAD: new session per connect
private func performConnect() async {
    let session = URLSession(configuration: .default)  // leaked!
    let task = session.webSocketTask(with: url)
    webSocketTask = task
    task.resume()
}
```

On the reconnection path (up to 10 retries with exponential backoff), each call creates a new session. The old session's `invalidateAndCancel()` is never called — `tearDown()` only cancels the task, not the session. These sessions accumulate in memory.

## The Fix

Store the session as a property, create it once:

```swift
final class ChatWebSocketManager {
    private let session = URLSession(configuration: .default)

    private func performConnect() async {
        let task = session.webSocketTask(with: url)
        webSocketTask = task
        task.resume()
    }
}
```

## Key Details

- `URLSession` manages its own connection pool — creating one per request defeats the purpose
- If you need to fully tear down, call `session.invalidateAndCancel()` — but then you can't reuse it
- For WebSocket reconnects, reusing the session is correct — only the task needs to be recreated
