# Заглядываем внутрь Task

Для начала, каждая задача имеет четыре состояния. За это отвечает класс `TaskStatusRecord`:

1. Suspended: In this state, a task is considered not runnable
2. Enqueued: In this state, a task is considered runnable
3. Running
4. Completed

За переключение состояний отвечают четыре метода. Возможны следующие переходы из одного состояния в другое:

```swift
// suspended -> enqueued
// suspended -> running
// enqueued -> running
// running -> suspended
// running -> completed
// running -> enqueued
```

#### Метаданные

Как понять, в каком состоянии задача приостановила свою работу ? А в каком должна возобновить ?
Для ответа на эти вопросы, существуют методы с помощью которых можно передать контекст для приостановления и возобновления задачи.

**AsyncContext**: данный контекст управляет состоянием задачи, которая ожидает завершения в дальнейшем. Когда задача приостанавливается, текущее состояние и контекст сохраняются, чтобы ее (задачу) можно было возобновить позже.
**Continuation** (TaskContinuationFunction): с помощью `continuation` возобновляется выполнение задачи.

```cpp
// Функция для возобновления выполнения AsyncTask.
TaskContinuationFunction * __ptrauth_swift_task_resume_function ResumeTask;
```

/// Describes type information and offers value methods for an arbitrary concrete
/// type in a way that's compatible with regular Swift and embedded Swift. In
/// regular Swift, just holds a Metadata pointer and dispatches to the value
/// witness table. In embedded Swift, because we do not have any value witness
/// tables present at runtime, the witnesses are stored and referenced directly.
///
/// This structure is created from swift_task_create, where in regular Swift, the
/// compiler provides the Metadata pointer, and in embedded Swift, a
/// TaskOptionRecord is used to provide the witnesses.

TaskOptionRecord: This records task creation options, including priority and other execution details, which help the executor manage tasks efficiently.

#### Планирование Task и Executors:

Executors: An executor is responsible for scheduling and running tasks. ExecutorRef manages references to executors, ensuring that tasks are resumed on the correct executor, which can be a global concurrent queue or a custom executor.
Job Interface: The Job interface represents a unit of work that an executor can run. Both tasks and continuations are implemented as jobs, allowing them to be scheduled and executed efficiently.


Safe and Efficient Suspension and Resumption Process
Suspending a Task:

When an async function encounters an await keyword, it suspends the current task. The task’s current state, including its execution context and any local variables, is saved in a continuation.
The runtime creates a TaskFutureWaitAsyncContext to manage the suspension state, ensuring the task can be resumed later.
Storing the Continuation:

The continuation is stored in a way that the executor or the runtime can access it when the awaited operation completes. This typically involves placing the continuation in a queue or a future’s waiting list.
Resuming a Task:

Once the awaited operation completes, the continuation is enqueued on the appropriate executor. The executor then schedules the continuation as a job to be executed.
The runtime restores the task’s state from the continuation, allowing the async function to resume execution from the point where it was suspended.
Handling Concurrency Safely:

Swift ensures thread safety by using synchronization primitives and careful state management in TaskStatusRecord and other concurrency constructs. This prevents race conditions and ensures that tasks are resumed correctly and safely.
Executors manage task execution and ensure that resumption occurs on the correct thread or queue, maintaining the correctness of concurrent execution.


Here is a simplified example illustrating task suspension and resumption in Swift:

```swift
func fetchData() async throws -> Data {
    let url = URL(string: "https://example.com/data")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}

Task {
    do {
        let data = try await fetchData()
        print("Data received: \(data)")
    } catch {
        print("Failed to fetch data: \(error)")
    }
}
```

`fetchData` is an asynchronous function that suspends when awaiting the network response.
The await keyword suspends the task, and the state of fetchData is saved in a continuation.
Once the network call completes, the continuation is enqueued and the task is resumed, printing the data or handling an error.