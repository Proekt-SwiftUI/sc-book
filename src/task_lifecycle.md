# Время жизни замыкания Task

Задача инициализируется с помощью замыкания, в которое мы передаем исполняемый код.
После завершения выполнения, задача завершиться вернув результат или ошибку.

Retaining a task object doesn't indefinitely retain the closure,
<!-- потому что ссылка, которую задача удерживает, освобождается после завершения самой задачи. -->
because any references that a task holds are released
after the task completes.
Поэтому, в задачах редко нужно захватывать значения по слабой ссылке.
<!-- Consequently, tasks rarely need to capture weak references to values. -->

В примере ниже, нет необходимости захватывать актор по слабой ссылке, потому что при выходе из области видимости, задача не будет удерживать ссылку, разорвав цикл между задачей (Task) и актором.
<!-- For example, in the following snippet of code it is not necessary to capture the actor as `weak`,
because as the task completes it'll let go of the actor reference, breaking the
reference cycle between the Task and the actor holding it. -->

```swift
struct EmptyResult {}

actor Worker {
	var workerTask: Task<Void, Never>?
	var result: EmptyResult?
	
	deinit {
		assert(workerTask != nil)
        // задача все еще удерживается,
       // но как только она завершиться, задача не создаст reference cycle с актором

		print("Retain count = ", _getRetainCount(Worker.self))
		print("Weak Retain count = ", _getWeakRetainCount(Worker.self))
		print("deinit actor")
	}
	
	func execute() {
		workerTask = Task {
			print("начало работы задачи")
			try? await Task.sleep(for: .seconds(1))
			
			self.result = EmptyResult() // захватили self
			print("конец работы, выход из области видимости")
            // при выходе из области видимости ссылка освобождается
		}

        // сильная ссылка сохраняется к workerTask
	}
}
```

Запустить можно так:

```swift
await Worker().execute()
```

Note that there is nothing, other than the Task's use of `self` retaining the actor,
And that the start method immediately returns, without waiting for the unstructured `Task` to finish.
So once the task is completed and its closure is destroyed, the strong reference to the "self" of the actor is also released allowing the actor to deinitialize as expected.
Therefore, the above call will consistently result in the following output:

```text
start task work
completed task work
deinit actor
```

