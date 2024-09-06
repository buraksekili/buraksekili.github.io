---
editPost:
  URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/thread-pooling-rs.md"
  Text: "Edit this page on "
author: "Burak Sekili"
title: "Thread Pooling in Rust "
date: "2024-09-06"
description: "Explore the implementation and need for thread pools in Rust"
tags: ["Rust", "concurrency"]
TocOpen: true
---

A thread pool is a software design pattern where a set of worker threads is created to execute tasks concurrently.
Instead of creating a new thread for each task, which can be resource-intensive, tasks are submitted to the pool and executed by available worker threads.

This blog post will go through a simple thread pool implementation - similar to the one in Rust book - with
a couple of simple enhancements.

Unlike Go's lightweight goroutines, Rust's threads map directly to OS threads, making thread pools widely used
technique for resource management and performance optimization.

At its core, a thread pool is a collection of worker threads that are ready to execute given tasks.
Instead of spawning a new thread for each task - potentially costly operation in Rust - tasks are
submitted to the pool and executed by available workers (threads).
This pattern is useful in various scenarios: web servers handling multiple client connections,
task scheduling systems processing queued jobs, parallel data processing applications crunching large datasets,
I/O-bound applications managing concurrent operations, and more.

There are various benefits of thread pools. They offer resource management by limiting the number of
active threads, preventing system resource exhaustion.
Performance may be optimized as the overhead of repeated thread creation and destruction is eliminated.
Thread pools also allow for predictable resource consumption, enabling developers to control and forecast the maximum thread usage.
Load balancing comes as a natural consequence, with tasks distributed evenly across available threads.

However, thread pools are not without their challenges.

- Introducing additional complexity to the codebase, requiring careful management of shared state
  and synchronization to avoid pitfalls like deadlocks and race conditions,
- For very short-lived tasks, the overhead of task submission might outweigh the benefits.
  - This means that sometimes if your task is not long-running or not compute expensive,
    sending this execution to another thread might be longer than the time spent on preparation
    of the task and sending it to a thread.
  - Let's say a task takes 1 ms to execute directly. If the overhead of submitting to and retrieving
    from the thread pool takes 10 ms, you're spending 11 ms total instead of just 1.
  - Determining the optimal pool size can be a tricky process as well.
  - Contention or starvation may happen. For instance, while workers are busy with long running tasks,
    shorter tasks are waiting in line for a long time.

In the context of Rust, additional language-specific considerations arise. The ownership system and
Rust's memory safety guarantees require careful implementation - that's why I wanted to implement
the pooling from scratch. It helped me to practice concurrency practices in Rust while considering
ownership model.

Now, let's start implementing a thread pool in Rust.

## Implementation

> Please read Rust book's thread pool section if you haven't read before: https://rust-book.cs.brown.edu/ch20-02-multithreaded.html

Our thread pool, as explained above, will have already initialized threads which are ready to execute
a task. Here, we will use `Worker` to represents an individual thread in our pool.

### Worker

Each worker runs in its own thread, continuously polling for jobs. Its purpose is to process the given
jobs.

You may ask who gives the task to worker. Worker will poll the jobs from a queue which will be
shared among all other workers. If no jobs are available, the worker needs to wait until another task
is submitted to queue.

```rust
struct Worker {
    // id corresponds to the arbitrary id for the thread
    // useful while debugging :)
    id: usize,
    // thread is the actual thread which is going
    // to execute a real task.
    thread: Option<thread::JoinHandle<()>>,
}
```

So, the requirements that we expect from the worker:

- It will execute the task in its thread,
  - Thread is part of Worker struct. So, we need to somehow implement a logic to get the task.
  - We cannot store the task in Worker struct as the same thread (or Worker) can be reused
    after executing its task - this is the idea of the thread pooling.
- Thread safe queue to pull the tasks.
  - It will be part of the constructor of the Worker. So that, when we spawn the thread,
    we also start listening this queue.
  - The queue needs to be shared among all workers. That's why it needs to be thread-safe.

While creating a Worker, we will use `std::thread` and spawn the thread.
In the closure of the thread, we will run our logic to pull tasks from the queue.

If there is a task on the queue, the thread will execute the task, and then waits
for a next task to be scheduled for itself. As Worker needs to re-run tasks after
executing a single one, it needs to run continous loop in order not to miss any
task pushed into queue.

If no task is provided, the queue will return `None`. In that case, the thread
needs to wait for next task to be scheduled. Therefore, thread needs to wait within
the loop.

There are various way to do that but most simple one is sleeping.
However, sleeping leads to busy-waiting which wastes CPU cycles. Also,
if there are new tasks registered while the thread is sleeping, the worker
will run the thread after a duration of sleep which causes a latency.

Instead of sleeping, we will use `Condvar` which is a synchronisation primitive
to allow threads to wait for some condition to become true. It's often used in
conjunction with a `Mutex` to provide a way for threads to efficiently wait for
changes in shared state. So, we will use `Condvar` to wait for new tasks to be
available as it allows immediate response when the job is available which
improves thread utilization.

```rust
impl Worker {
    fn new(
        id: usize,
        job_queue: Arc<SegQueue<Job>>,
        job_signal: Arc<(Mutex<bool>, Condvar)>,
        running: Arc<AtomicBool>,
    ) -> Worker {
        let thread = thread::spawn(move || loop {
            match job_queue.pop() {
                Some(Job::Task(task)) => if let Err(_) = task() {},
                Some(Job::Shutdown) => {
                    break;
                }
                None => {
                    let (lock, cvar) = &*job_signal;
                    let mut job_available = lock.lock().unwrap();
                    while !*job_available && running.load(Ordering::Relaxed) {
                        job_available = cvar
                            .wait_timeout(job_available, Duration::from_millis(100))
                            .unwrap()
                            .0;
                    }
                    *job_available = false;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

Our queue depends on `crossbeam`'s `crossbeam_queue::SegQueue` which is a concurrent
queue implementation. It's designed for high-performance concurrent scenarios. Also,
it is lock-free which is an important aspect of `SegQueue`.

Our queue is actually a linked list of `Job` which will look like following:

```rust
pub enum Job {
    Task(Box<dyn FnOnce() -> Result<(), Box<dyn std::error::Error>> + Send + 'static>),
    Shutdown,
}
```

Job has two possible values, `Task` and `Shutdown`.

The `Shutdown` job type allows us to break the loop and terminate the thread gracefully.
If a thread receives a `Shutdown` from the queue, it will stop executing.

Okay, now check the `Task`. I know it looks too complicated at first glance, it still
looks complicated to me. I'll try to break it down.

- `Box<dyn FnOnce() -> Result<(), Box<dyn std::error::Error>> + Send + 'static>`

Here we have two main parts; the `FnOnce()` and the return type of this closure,
`Box<dyn std::error::Error>> + Send + 'static`

- `FnOnce()`:
  - It is a closure without taking any arguments. It is same as `thread::spawn(|| {})`
    closure.
  - helps us to prevent accidentally call a task multiple times when
    it's not safe to do so.
  - `-> Result<(), Box<dyn std::error::Error>>` is the return type of this closure.
    On success, it returns `()` (unit, or void). On error, it returns a boxed trait object
    of `std::error::Error`.

> Trait object is one of the ways in Rust to write polymorphic code. Especially, dynamic
> dispatch uses trait objects to resolve generic function calls at runtime.

- `Box<dyn ...>`: Box is used for heap allocation. `dyn` indicates a trait object,
  allowing for dynamic dispatch.
- `Send`: is a trait which ensures the closure can be safely sent between threads.
- `'static`: is a lifetime bound ensuring the closure doesn't contain any
  non-static references. `'static` bound ensures a type is safe to use without
  lifetime constraints. Without `'static`, we might create closures that reference
  stack-local variables, leading to use-after-free bugs.

In the arguments of `Worker::new` method, the use of `Arc` allows safe sharing of the job queue
and signaling mechanism between threads.
This is crucial in Rust's ownership model for concurrent programming.
Otherwise, shared data cannot be used safely among multiple threads as it will cause lots of critical bugs.

### Thread Pool

```rust
pub struct ThreadPool {
    // workers keep track of all worker threads.
    workers: Vec<Worker>,
    // job_queue corresponds to a shared queue for distributing jobs to workers.
    job_queue: Arc<SegQueue<Job>>,
    // job_signal is notifier for workers when new jobs are available.
    job_signal: Arc<(Mutex<bool>, Condvar)>,
    // running indicates whether the threadpool is actively running or not.
    // it is mainly checked by worker threads to understand the status
    // of the pool.
    running: Arc<AtomicBool>,
}

impl ThreadPool {
       pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let job_queue = Arc::new(SegQueue::new());
        let job_signal = Arc::new((Mutex::new(false), Condvar::new()));
        let mut workers = Vec::with_capacity(size);
        let running = Arc::new(AtomicBool::new(true));

        for id in 0..size {
            workers.push(Worker::new(
                id,
                Arc::clone(&job_queue),
                Arc::clone(&job_signal),
                Arc::clone(&running),
            ));
        }

        ThreadPool {
            workers,
            job_queue,
            job_signal,
            running,
        }
    }

    pub fn execute<F>(&self, f: F) -> Result<(), ThreadPoolError>
    where
        F: FnOnce() -> Result<(), Box<dyn std::error::Error>> + Send + 'static,
    {
        // We create a new Job::Task, wrapping our closure 'f'
        let job = Job::Task(Box::new(f));
        // Push this job to our queue
        self.job_queue.push(job);
        // Signal that a new job is available
        let (lock, cvar) = &*self.job_signal;
        let mut job_available = lock.lock().unwrap();
        *job_available = true;
        cvar.notify_all();
        Ok(())
    }
}
```

`ThreadPool::new` creates a fixed number of workers,
each with shared access to the job queue and signaling mechanism.

`ThreadPool::execute` takes a generic parameter called `F` which implements
`FnOnce() -> Result<(), Box<dyn std::error::Error>> + Send + 'static`. This type
is the same type used for `Job::Task` - so that we can use argument `f` as `Job`.
The argument passed in `execute` is actually a task that needs to be run in any of the workers.

For example,

```rust
let x = 3;
pool.execute(move || {
    println!("the task is running with value {}", x);
    Ok(())
})
```

The `execute` will take closure function as argument and we wrap the user's closure
, which is argument `f: F` in `ThreadPool::execute` in `Job::Task` because:

- It allows us to send different types of jobs through the same queue. Workers can
  distinguish between actual tasks and shutdown signals.
- It provides a uniform type (or interface if you are familiar
  with interfaces in other languages) for our job queue.
  All items in the queue are of type `Job`, regardless of the closure they contain.

But, why do we use `Box`? The `Box` is crucial here for several reasons:

- We are using `dyn FnOnce()` which is a trait object.
  In Rust, trait objects must be behind a pointer, and `Box` provides this.
- Closures can capture variables from their environment - as we did in the example above -
  which makes their size unknown at compile time. Box puts the closure on the heap,
  giving it a known size (the size of a pointer) at compile time.
- Box allows us to take ownership of the closure and move it into the `Job` enum,
  which is necessary because the closure will be executed in a different thread.

This design actually allows us to:

- Handle different types of jobs (tasks and shutdown signals) uniformly.
- Move closures between threads safely, respecting Rust's ownership rules.
- Deal with closures of different sizes and types in a unified manner.

### Graceful Shutdown

```rust
impl ThreadPool {
    pub fn shutdown(&mut self, timeout: Duration) -> Result<(), ThreadPoolError> {
        let start = Instant::now();
        // Step 1: Signal all workers to stop
        self.running.store(false, Ordering::SeqCst);

        // Step 2: Wake up all waiting threads
        let (lock, cvar) = &*self.job_signal;
        match lock.try_lock() {
            Ok(mut job_available) => {
                *job_available = true;
                cvar.notify_all();
            }
            Err(_) => {
                // We couldn't acquire the lock, but we've set running to false,
                // so workers will eventually notice
                println!("Warning: Couldn't acquire lock to notify workers. They will exit on their next timeout check.");
            }
        }

        // Step 3: Wait for all workers to finish
        for worker in &mut self.workers {
            if let Some(thread) = worker.thread.take() {
                // Step 4: Calculate remaining time
                let remaining = timeout
                    .checked_sub(start.elapsed())
                    .unwrap_or(Duration::ZERO);

                // Step 5: Check if we've exceeded the timeout
                if remaining.is_zero() {
                    return Err(ThreadPoolError::ShutdownTimeout);
                }

                // Step 6: Wait for the worker to finish
                if thread.join().is_err() {
                    return Err(ThreadPoolError::ThreadJoinError(format!(
                        "Worker {} failed to join",
                        worker.id
                    )));
                }
            }
        }
        // Step 7: Final timeout check
        if start.elapsed() > timeout {
            Err(ThreadPoolError::ShutdownTimeout)
        } else {
            Ok(())
        }
    }
}
```

To notify workers polling the queue, we set the `running` flag to false.
This is an atomic operation that immediately signals all workers that a shutdown
is in progress.
Since all threads need to finish executing the task they assigned to or stop waiting
for next job (through `cvar.wait_timeout` method in `Worker::new` method) for proper shutdown,
`job_available` is set to true, which triggers all threads and we notify all threads waiting on the
condition variable.

We use `try_lock()` instead of `lock()` while notifying the threads.
This attempts to acquire the lock but returns immediately if it can't, rather than waiting until
acquiring the lock.

- If the lock is acquired, we proceed as before: set `job_available` to true and notify all waiting threads. By doing that, we can ensure that idle workers that were waiting
  on `cvar.wait_timeout` will wake up and notice the shutdown signal.

- If the lock is not acquire successfully, the shutdown process continues. As `running`
  is already set to false, which all workers check periodically and
  `wait_timeout` in `Worker`'s main loop will expire - so they'll wake up eventually
  and notice that `running` is false.

After sending signals to notify threads, then we iterate through all workers,
attempting to join each thread. Before joining each thread, the remaining time is calculated based on our timeout.
This ensures we respect the overall timeout even if a worker is stuck (e.g., in an infinite loop), the timeout ensures we don't wait forever.

To use shutdown method explicitly, the `Drop` can be implemented where the shutdown
method can be triggered whenver `ThreadPool` is dropped, such as ThreadPool variable
going out of scope.

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        if !self.workers.is_empty() {
            let _ = self.shutdown(Duration::from_secs(2));
        }
    }
}
```

Implementing a thread pool in Rust offers a great opportunity to practice Rust's concurrency and memory safety paradigms.
Throughout this blog post, I've tried to explain some concepts
such as atomic operations, condition variables, and Rust's ownership system in a
practical context.
While the implementation provides a solid foundation, there's always room for improvement and optimization - so, ofc not use it on anywhere :)
There are plenty of great crates including `crossbeam`'s. I just developed this thread pooling to practice concepts aforementioned.
Consider exploring advanced features like work stealing algorithms or dynamic pool sizing
to further enhance performance.

If you notice any mistakes or have feedback, feel free to reach out to me on
[Twitter](https://x.com/buraksekili), [LinkedIn](https://www.linkedin.com/in/sekili/), or GitHub.

## References

- https://doc.rust-lang.org/book/ch20-02-multithreaded.html
- https://github.com/crossbeam-rs/crossbeam
