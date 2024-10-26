# Unstructured Concurrency

Если **SC** - это явно упорядоченные отношения между родильской и дочерними задачами, то Unstructured Concurrency — полная противоположность.

Unstructured concurrency (**UC**), напротив, предоставляет разработчику больше гибкости: задачи могут существовать независимо друг от друга, не имеют явной иерархии и не связаны напрямую с родительской задачей. В определенных ситуациях не обойтись без **UC** и такое поведение также означает, что разработчик сам отвечает за корректное управление задачами, отмену и обработку ошибок.

> [!NOTE]
> Основное отличие Unstructured от Structured — отсуствие родительской задачи!

### На примере

В самом простом сценарии (когда мы не сможем управлять задачей), нужно использовать структуру `Task` или `Task.detached` передавая в замыкание код:

```swift
Task {…}

Task.detached {…}
```

Более продвинутым вариантом послужит явно объявить тип данных (можно не указавать, компилятор определит тип данных сам) с последующим управлением:

1. Отменой задачи
2. Получение значения
3. Получением результата
4. Индикация отмены `isCancelled`

```swift
let handleNameViaTask = Task<String, Never> {
    "Дорогой читатель"
}

// 1. Отмена задачи
// handleNameViaTask.cancel()

// 2. Получение значения
// print(await handleNameViaTask.value)

/*
3. Обработка результата с помощью switch паттерна или guard case

switch await handleNameViaTask.result {
    case let .success(userName): print("User name: ", userName)
    case let .failure(error): print(error.localizedDescription)
}

guard case let .success(userName) = await handleNameViaTask.result else { return }
*/

// 4. Индикация отмены
// handleNameViaTask.isCancelled
```

Помимо этого, задача может быть объявлена в качестве опционального свойства:

```swift
struct Worker {
	var task: Task<Void, Never>?
}
```

### Время жизни задачи

Когда задача в методе `execute` завершилась, замыкание оперативно освобождается. Удерживание задачи не приводит к постоянному циклу, потому что все ссылки, которые захвачены задачей, освобождаются после ее завершения. В результате задача редко нуждаются в захвате слабых ссылок.

В коде ниже нет необходимости захватывать актор как слабую ссылку, т.к. по завершении задачи, ссылка на актор будет освобождена, разрывая reference cycle между `Task` и актором.


```swift
struct EmptyResult {}

actor Worker {
	var workerTask: Task<Void, Never>?
	var result: EmptyResult?
	
	deinit {
		assert(workerTask != nil)
		print("Retain count = ", _getRetainCount(Worker.self))
		print("Weak Retain count = ", _getWeakRetainCount(Worker.self))
		print("deinit actor")
	}
	
	func execute() {
		workerTask = Task {
			print("начало работы задачи")
			try? await Task.sleep(for: .seconds(1))
			
			self.result = EmptyResult()
			print("конец работы, выход из области видимости")
		}
	}
}

await Worker().execute()
```

> [!NOTE]
> Обратите внимание, что актор удерживается только за счет использования `self` методом `execute()` и  метод `execute()` выполняется сразу, не ожидая завершения неструктурированной задачи. Когда задача завершается и ее замыкание уничтожается, сильная ссылка на актор также освобождается, позволяя актору корректно деинициализироваться, как и полагается.

```js
начало работы задачи
конец работы, выход из области видимости

Retain count =  1
Weak Retain count =  1

deinit actor
```