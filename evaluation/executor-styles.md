# Executor styles and Send bounds

One key aspect of async fn in traits has to do with how to communicate `Send` bounds needed to spawn tasks. The key question is roughly "what send bounds are required to safely spawn a task?"

* A **single threaded** executor runs all tasks on a single thread. 
    * In this scenario, nothing has to be `Send`.
    * Example: [tokio::spawn_local](https://docs.rs/tokio/1.11.0/tokio/task/fn.spawn_local.html)
* A **thread per core** executor selects a thread to run a task when the task is spawned, but never migrates tasks between threads. This can be very efficient because the runtimes never need to communicate across threads except to spawn new tasks. 
    * In this scenario, the "initial state" must be `Send` but not the future once it begins executing.
    * Example: [glommio::spawn](https://docs.rs/glommio/0.5.1/glommio/struct.LocalExecutorBuilder.html#method.spawn)
* A **work-stealing** executor can move tasks between threads even mid-execution. 
    * In this scenario, the future must be `Send` at all times (or we have to rule out the ability to have leaks of data out from the future, which we don't have yet).
    * Example: [tokio::spawn](https://docs.rs/glommio/0.5.1/glommio/struct.LocalExecutorBuilder.html#method.spawn)
