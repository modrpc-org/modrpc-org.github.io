# Motivating example - task manager

Imagine you have an app (or several) that manages many different kinds of long-running tasks. And you're a stickler for avoiding unnecessary communications, so you want the client to keep track of all pending tasks and only receive status updates, never poll. So you write a reusable `TaskManager` interface to encapsulate the common logic of spawning, tracking, and optionally canceling tasks.

```
interface TaskManager<SpawnPayload, State, Error> @(Client, Server) {
    objects {
        spawn: std.Request<
            SpawnRequest<SpawnPayload>,
            result<void, SpawnError<Error>>,
        > @(Client, Server),

        cancel: std.Request<
            TaskId,
            result<void, CancelError>,
        > @(Client, Server),

        // server -> client ordered streams (per-task) of status updates
        task_status_update: std.MultiStream<TaskStatus<State>> @(Client, Server),
    }

    impl @(Server) {
        // Each TaskManager Server must specify how to spawn its tasks.
        new_task: NewTask<SpawnPayload> -> void,
    }

    methods @(Server) {
        // The server's tasks can invoke this method to send status updates to clients.
        post_status: PostTaskStatus<State> -> void,
    }

    // Clients can request to spawn and cancel tasks.
    methods @(Client) {
        spawn: async SpawnPayload -> result<SpawnSuccess, SpawnError<Error>>,
        cancel: async TaskId -> result<void, CancelError>,
    }

    state {
        // Clients will be provided a list of all pending tasks when they first connect.
        pending_tasks: [TaskId],
    }
}

struct TaskId { id: u64 }

struct SpawnSuccess {
    task_id: TaskId,
    status_stream: std.MultiStreamId,
}

// ... some definitions omitted
```

One of your apps is a worker for a pipeline that downloads and transcodes cat videos:

```
interface CatVideoTranscoder @(Client, Server) {
    objects {
        cat_video_downloads:
            TaskManager<
                CatUrl, CatVideoDownloadState, CatDownloadSpawnError,
            > @(Client, Server),

        h264_to_h265_transcoding: TaskManager<
            CatVideoId, VideoTranscodingState, VideoTranscodingSpawnError,
        > @(Client, Server),
    }
}

// ... some definitions omitted
```
