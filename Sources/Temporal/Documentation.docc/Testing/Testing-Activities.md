# Testing activities

Test activities in isolation with the activity test environment, or against a
local Temporal test server.

## Overview

Activities run the side-effecting work in your Temporal application —
calls to external services, database operations, and other actions a
workflow can't do directly. This article shows you how to test activities
in isolation, verify heartbeat details, simulate cancellation, and run
activities as part of a full workflow test.

You can test activities in two ways:

1. **Unit tests** with `withActivityTestEnvironment`:
   fast tests that cover activity logic, heartbeats, and cancellation
   without starting a server.
2. **Integration tests** with `TemporalTestServer`: verify the full
   activity-workflow interaction, including serialization and retries.

> Tip: Add `TemporalTestKit` as a test dependency in your Package.swift to
> access the test environment and test server.

### Unit test activities with the test environment

Use `withActivityTestEnvironment` from the `TemporalTestKit` module to test
activities that depend on ``ActivityExecutionContext``. This sets up a context
without requiring a Temporal server.

``ActivityExecutionContext/Info`` has a convenience initializer that fills in
defaults for every field. Pass only the values your test cares about:

```swift
import Temporal
import TemporalTestKit
import Testing

@Test
func sayHelloReturnsGreeting() async throws {
    let info = try await ActivityExecutionContext.Info()

    try await withActivityTestEnvironment(info: info) {
        let result = GreetingActivities().sayHello(input: "World")
        #expect(result == "Hello, World!")
    }
}
```

To test an activity that reads its execution context — for example, one that
resumes from the last heartbeat on retry — pass specific values through the
initializer:

```swift
@Test
func activityResumesFromHeartbeat() async throws {
    let info = try await ActivityExecutionContext.Info(
        attempt: 2,
        heartbeatDetails: 42
    )

    try await withActivityTestEnvironment(info: info) {
        let context = try #require(ActivityExecutionContext.current)
        let lastProgress = try await context.info.heartbeatDetails(
            as: Int.self
        )
        #expect(lastProgress == 42)
    }
}
```

### Verify heartbeats

Use the `assertHeartbeatDetails` closure to inspect heartbeats recorded by
the activity. The closure receives a
`HeartbeatDetailsSequence` that yields each heartbeat as the activity
emits it:

```swift
@Test
func activityRecordsHeartbeats() async throws {
    let info = try await ActivityExecutionContext.Info()

    try await withActivityTestEnvironment(info: info) {
        let context = try #require(ActivityExecutionContext.current)
        context.heartbeat(details: "step-1")
        context.heartbeat(details: "step-2")
    } assertHeartbeatDetails: { heartbeats in
        var recorded: [String] = []
        for try await details in heartbeats {
            guard let values = details as? [String] else { continue }
            recorded.append(contentsOf: values)
        }
        #expect(recorded == ["step-1", "step-2"])
    }
}
```

### Test cancellation

Activity cancellation in Temporal propagates through Swift's built-in task
cancellation. To test that your activity handles cancellation correctly,
cancel the task from outside:

```swift
@Test
func activityHandlesCancellation() async throws {
    let info = try await ActivityExecutionContext.Info()

    try await withThrowingTaskGroup(of: Void.self) { group in
        group.addTask {
            try await withActivityTestEnvironment(info: info) {
                try await Task.sleep(for: .seconds(60))
            }
        }
        group.cancelAll()
    }
}
```

You can also pass a `cancellationReason` to make a specific reason available
through ``ActivityExecutionContext/cancellationReason``. See
``ActivityCancellationReason`` for the full list of reasons.

### Integration test activities with a workflow

To test against a real server, apply the `.temporalTestServer` test trait to
start a local Temporal server automatically. Then use
`TemporalTestServer.withConnectedWorker` and
`TemporalTestServer.withConnectedClient` to set up a worker and client:

```swift
import Logging
import Temporal
import TemporalTestKit
import Testing

@Suite(.temporalTestServer)
struct GreetingActivityTests {
    @Test
    func greetingWorkflowReturnsHello() async throws {
        let testServer = TemporalTestServer.testServer!
        let taskQueue = "test-\(UUID())"
        let logger = Logger(label: "test")

        let config = TemporalWorker.Configuration(
            namespace: "default",
            taskQueue: taskQueue,
            instrumentation: .init(serverHostname: "localhost")
        )

        try await testServer.withConnectedWorker(
            configuration: config,
            activities: GreetingActivities().allActivities,
            workflows: [GreetingWorkflow.self]
        ) { _ in
            try await testServer.withConnectedClient(
                logger: logger
            ) { client in
                let handle = try await client.startWorkflow(
                    type: GreetingWorkflow.self,
                    options: .init(
                        id: "wf-\(UUID())",
                        taskQueue: taskQueue
                    ),
                    input: "World"
                )
                let result = try await handle.result()
                #expect(result == "Hello, World!")
            }
        }
    }
}
```

> Important: The `.temporalTestServer` trait manages the server lifecycle.
> It starts before your tests run and shuts down when they complete.
