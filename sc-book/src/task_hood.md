# Заглядываем внутрь Task

Для начала, каждая задача имеет четыре состояния. За состояние (статус) отвечает класс `TaskStatusRecord`:

1. Suspended: приостановлена и не выполняется
2. Enqueued: ожидает выполнения
3. Running: выполняется
4. Completed: выполнена

За переключение состояний отвечают четыре метода. Возможны следующие переходы из одного состояния в другое:

```swift
// suspended -> enqueued
// suspended -> running
// enqueued -> running
// running -> suspended
// running -> completed
// running -> enqueued
```

## Метаданные

Как понять, в каком состоянии задача приостановила свою работу ? А в каком должна возобновить ?
Для ответа на эти вопросы, существуют методы с помощью которых можно передать контекст для приостановления и возобновления задачи.

1. `AsyncContext`: данный контекст управляет состоянием задачи, которая ожидает завершения в дальнейшем. Когда задача приостанавливается, текущее состояние и контекст сохраняются, чтобы ее (задачу) можно было возобновить позже.
2. `Continuation` (TaskContinuationFunction): с помощью `continuation` возобновляется выполнение задачи.

```cpp
// Функция для возобновления выполнения AsyncTask.
TaskContinuationFunction * __ptrauth_swift_task_resume_function ResumeTask;
```

3. Класс `TaskOptionRecord` отвечает за запись параметров созданной задачи, включая приоритет и другие детали, помогающие исполнителю эффективно и корректно управлять задачами.
В обычном Swift хранится указатель метаданных и диспетчирезация происходит в witness table.
Во встроенном (embedded) Swift у нас отсутствует witness table в рантайме, поэтому диспетчеризация вызывается напрямую (direct).

<!-- /// Describes type information and offers value methods for an arbitrary concrete
/// type in a way that's compatible with regular Swift and embedded Swift. In
/// regular Swift, just holds a Metadata pointer and dispatches to the value
/// witness table. In embedded Swift, because we do not have any value witness
/// tables present at runtime, the witnesses are stored and referenced directly.
///
/// This structure is created from swift_task_create, where in regular Swift, the
/// compiler provides the Metadata pointer, and in embedded Swift, a
/// TaskOptionRecord is used to provide the witnesses. -->

## Планирование, приостановка и возобновление задачи

Исполнитель (executor) отвечает за планирование и запуск задач.
<!-- Executors: An executor is responsible for scheduling and running tasks. ExecutorRef manages references to executors, ensuring that tasks are resumed on the correct executor, which can be a global concurrent queue or a custom executor.
Job Interface: The Job interface represents a unit of work that an executor can run. Both tasks and continuations are implemented as jobs, allowing them to be scheduled and executed efficiently. -->

#### Приостановка задачи:

При использовании ключевого слова `await`, выполняемая задача приостанавливается.
Текущее состояние, включая приоритет, контекст и локальные переменные сохраняются.
В рантайме создается `ContinuationAsyncContext` для управления состоянием задачи, гарантируя возобновление задачи позже.

Хранение состояния для возобновления:

`Continuation` хранит данные таким образом, чтобы исполнитель или рантайм имели доступ к задаче во время приостановки. Обычно эти данные используются при добавлении задачи в очередь и последующего возобновления.

#### Возобновление задачи:

После завершения приостановки `await`, задача попадает в очередь соответствующего исполнителя, который планирует возобновление задачи.
Runtime восстанавливает состояние задачи, позволяя возобновить выполнение `async` функции с того места, где она (функция) была приостановлена.

### Handling Concurrency Safely:

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
