#### Hop to MainActor

Функция `hopToMainActor(_:)` с помощью `unsafeBitCast` обходит проверку компилятора на выполнение замыкания с `@MainActor` на главном потоке, преобразуя его в обычное замыкание `() -> ()`.
Это позволяет выполнить замыкание в любом потоке, нарушая акторную изоляцию и потенциально создавая проблемы с потокобезопасностью.

```swift
func hopToMainActor(_ x: @escaping @MainActor () -> ()) {
	typealias Func = () -> ()
	let x2 = unsafeBitCast(x, to: Func.self)
	x2()
}

@MainActor
func another() async {
	hopToMainActor { }
}
```

> [!NOTE]
> unsafeBitCast — это низкоуровневое преобразование типов, которое позволяет преобразовать один тип в другой без проверки их совместимости.

#### Текущий приоритет задачи

```swift
func executedAt(priority: TaskPriority) async {
    print("START 🔦:")
    let currentPrioprity = Task.currentPriority
    
    while (priority != currentPrioprity) {
        print("Task priority = \(currentPrioprity) != fn \(priority)")
        try? await Task.sleep(for: .seconds(1))
    }
}

await executedAt(priority: .background)
```

<details>
  <summary>Вывод</summary>

```rust
START 🔦:
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
Task priority = TaskPriority.high != fn TaskPriority.background
```

</details>

#### Task init

```swift
func download10MB(id: Int) async throws -> Data {
    let readmeURL = URL(string: "https://raw.githubusercontent.com/wmorgue/wmorgue/main/README.md")!
    let startDate = Date()

    print("Task #\(id) started downloading.")
	
	defer {
		let duration = Date().timeIntervalSince(startDate)
        print("Task #\(id) completed in \(duration) seconds.")
	}
	
	return try await URLSession.shared.data(from: readmeURL).0
}

for r in 0...10 {
	Task {
		try await download10MB(id: r)
	}
}
```

<details>
  <summary>Вывод</summary>

```js
Task #1 started downloading.
Task #2 started downloading.
Task #7 started downloading.
Task #0 started downloading.
Task #4 started downloading.
Task #6 started downloading.
Task #5 started downloading.
Task #3 started downloading.
Task #10 started downloading.
Task #9 started downloading.
Task #8 started downloading.
Task #5 completed in 1.651481032371521 seconds.
Task #2 completed in 1.6585689783096313 seconds.
Task #3 completed in 1.6518199443817139 seconds.
Task #6 completed in 1.651737928390503 seconds.
Task #4 completed in 1.651810884475708 seconds.
Task #7 completed in 1.65175199508667 seconds.
Task #0 completed in 1.6590250730514526 seconds.
Task #1 completed in 1.658919095993042 seconds.
Task #10 completed in 1.5991131067276 seconds.
Task #8 completed in 1.5986690521240234 seconds.
Task #9 completed in 1.5986000299453735 seconds.
```

</details>

Но более правильным вариантом будет использование `withThrowingTaskGroup`:

```swift
await withThrowingTaskGroup(of: Data.self) { group in
	for r in 0...10 {
		try await group.addTask { try await download10MB(id: r) }
	}
}

<details>
  <summary>Вывод</summary>

Task #3 started downloading.
Task #2 started downloading.
Task #1 started downloading.
Task #6 started downloading.
Task #5 started downloading.
Task #0 started downloading.
Task #4 started downloading.
Task #7 started downloading.
Task #10 started downloading.
Task #8 started downloading.
Task #9 started downloading.
Task #0 completed in 1.0400439500808716 seconds.
Task #6 completed in 1.0401649475097656 seconds.
Task #1 completed in 1.0402040481567383 seconds.
Task #8 completed in 1.0201719999313354 seconds.
Task #10 completed in 1.0204299688339233 seconds.
Task #9 completed in 1.0203959941864014 seconds.
Task #7 completed in 1.040037989616394 seconds.
Task #2 completed in 1.0401289463043213 seconds.
Task #3 completed in 1.0400739908218384 seconds.
Task #4 completed in 1.0400439500808716 seconds.
Task #5 completed in 1.0400769710540771 seconds.
```

</details>

#### Работа с приоритетами

```swift
let taskPriorities: [TaskPriority] = [.userInitiated, .background, .low]

func makeEachTask(with priority: TaskPriority, fn: () async -> Void) async {
	print("Init task with \(priority) priority")
	
	await fn()
}

for prior in taskPriorities {
	Task {
		await makeEachTask(with: prior) {
			print("Another job via \(prior.description)")
		}
	}
}
```


#### Проверка отмены у группы

```swift
func test_detach_cancel_taskGroup() async {
  print(#function) // CHECK: test_detach_cancel_taskGroup

  await withTaskGroup(of: Void.self) { group in
    group.cancelAll() // immediately cancel the group
    print("group.cancel()") // CHECK: group.cancel()

    group.addTask {
      // immediately cancelled child task...
      await withTaskCancellationHandler {
        print("child: operation, was cancelled: \(Task.isCancelled)")
      } onCancel: {
        print("child: onCancel, was cancelled: \(Task.isCancelled)")
      }
    }
    // CHECK: child: onCancel, was cancelled: true
    // CHECK: child: operation, was cancelled: true
  }

  print("done") // CHECK: done
}

await test_detach_cancel_taskGroup()
```

#### Worker

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
```

#### Явный yeild задач

```swift
protocol Start: Actor {
	func start(times: Int) async -> Int
}

extension Start {
	func start(times: Int) async -> Int {
		for i in 0...times {
			print("actor \(Self.self): \(#function) \(i)")
			await Task.yield()
		}
		
		return times
	}
}

actor One: Start {}
actor Two: Start {}

func yielding() async {
	let one = One()
	let two = Two()
	
	await withTaskGroup(of: Int.self) { group in
		group.addTask {
			await one.start(times: 100)
		}
		
		group.addTask {
			await two.start(times: 100)
		}
	}
}

await yielding()
```

<details>
  <summary>Вывод</summary>

```bash
actor One: start(times:) 0
actor Two: start(times:) 0
actor One: start(times:) 1
actor Two: start(times:) 1
actor Two: start(times:) 2
actor One: start(times:) 2
actor One: start(times:) 3
actor Two: start(times:) 3
actor One: start(times:) 4
actor One: start(times:) 5
actor One: start(times:) 6
actor One: start(times:) 7
actor One: start(times:) 8
actor Two: start(times:) 4
actor Two: start(times:) 5
actor One: start(times:) 9
actor Two: start(times:) 6
actor Two: start(times:) 7
actor One: start(times:) 10
actor Two: start(times:) 8
actor One: start(times:) 11
actor One: start(times:) 12
actor One: start(times:) 13
actor Two: start(times:) 9
actor Two: start(times:) 10
actor One: start(times:) 14
actor Two: start(times:) 11
actor Two: start(times:) 12
actor Two: start(times:) 13
actor Two: start(times:) 14
actor Two: start(times:) 15
actor Two: start(times:) 16
actor One: start(times:) 15
actor One: start(times:) 16
actor Two: start(times:) 17
actor One: start(times:) 17
actor One: start(times:) 18
actor Two: start(times:) 18
actor One: start(times:) 19
actor Two: start(times:) 19
actor One: start(times:) 20
actor Two: start(times:) 20
actor One: start(times:) 21
actor Two: start(times:) 21
actor Two: start(times:) 22
actor One: start(times:) 22
actor Two: start(times:) 23
actor Two: start(times:) 24
actor One: start(times:) 23
actor One: start(times:) 24
actor Two: start(times:) 25
actor One: start(times:) 25
actor One: start(times:) 26
actor Two: start(times:) 26
actor One: start(times:) 27
actor One: start(times:) 28
actor One: start(times:) 29
actor One: start(times:) 30
actor One: start(times:) 31
actor One: start(times:) 32
actor One: start(times:) 33
actor Two: start(times:) 27
actor One: start(times:) 34
actor Two: start(times:) 28
actor One: start(times:) 35
actor Two: start(times:) 29
actor One: start(times:) 36
actor Two: start(times:) 30
actor One: start(times:) 37
actor Two: start(times:) 31
actor Two: start(times:) 32
actor Two: start(times:) 33
actor Two: start(times:) 34
actor One: start(times:) 38
actor Two: start(times:) 35
actor Two: start(times:) 36
actor One: start(times:) 39
actor Two: start(times:) 37
actor One: start(times:) 40
actor One: start(times:) 41
actor One: start(times:) 42
actor One: start(times:) 43
actor Two: start(times:) 38
actor One: start(times:) 44
actor Two: start(times:) 39
actor One: start(times:) 45
actor Two: start(times:) 40
actor One: start(times:) 46
actor Two: start(times:) 41
actor One: start(times:) 47
actor Two: start(times:) 42
actor Two: start(times:) 43
actor One: start(times:) 48
actor One: start(times:) 49
actor Two: start(times:) 44
actor One: start(times:) 50
actor Two: start(times:) 45
actor One: start(times:) 51
actor One: start(times:) 52
actor One: start(times:) 53
actor One: start(times:) 54
actor One: start(times:) 55
actor One: start(times:) 56
actor Two: start(times:) 46
actor Two: start(times:) 47
actor Two: start(times:) 48
actor Two: start(times:) 49
actor One: start(times:) 57
actor Two: start(times:) 50
actor One: start(times:) 58
actor Two: start(times:) 51
actor One: start(times:) 59
actor Two: start(times:) 52
actor One: start(times:) 60
actor Two: start(times:) 53
actor One: start(times:) 61
actor Two: start(times:) 54
actor One: start(times:) 62
actor Two: start(times:) 55
actor Two: start(times:) 56
actor One: start(times:) 63
actor Two: start(times:) 57
actor One: start(times:) 64
actor One: start(times:) 65
actor Two: start(times:) 58
actor One: start(times:) 66
actor One: start(times:) 67
actor One: start(times:) 68
actor Two: start(times:) 59
actor One: start(times:) 69
actor One: start(times:) 70
actor Two: start(times:) 60
actor One: start(times:) 71
actor One: start(times:) 72
actor One: start(times:) 73
actor One: start(times:) 74
actor Two: start(times:) 61
actor One: start(times:) 75
actor Two: start(times:) 62
actor One: start(times:) 76
actor One: start(times:) 77
actor Two: start(times:) 63
actor One: start(times:) 78
actor Two: start(times:) 64
actor One: start(times:) 79
actor Two: start(times:) 65
actor One: start(times:) 80
actor Two: start(times:) 66
actor One: start(times:) 81
actor Two: start(times:) 67
actor One: start(times:) 82
actor One: start(times:) 83
actor Two: start(times:) 68
actor Two: start(times:) 69
actor One: start(times:) 84
actor Two: start(times:) 70
actor One: start(times:) 85
actor One: start(times:) 86
actor Two: start(times:) 71
actor One: start(times:) 87
actor One: start(times:) 88
actor Two: start(times:) 72
actor One: start(times:) 89
actor Two: start(times:) 73
actor One: start(times:) 90
actor One: start(times:) 91
actor Two: start(times:) 74
actor One: start(times:) 92
actor Two: start(times:) 75
actor One: start(times:) 93
actor Two: start(times:) 76
actor One: start(times:) 94
actor Two: start(times:) 77
actor One: start(times:) 95
actor Two: start(times:) 78
actor One: start(times:) 96
actor One: start(times:) 97
actor One: start(times:) 98
actor Two: start(times:) 79
actor Two: start(times:) 80
actor One: start(times:) 99
actor Two: start(times:) 81
actor Two: start(times:) 82
actor Two: start(times:) 83
actor One: start(times:) 100
actor Two: start(times:) 84
actor Two: start(times:) 85
actor Two: start(times:) 86
actor Two: start(times:) 87
actor Two: start(times:) 88
actor Two: start(times:) 89
actor Two: start(times:) 90
actor Two: start(times:) 91
actor Two: start(times:) 92
actor Two: start(times:) 93
actor Two: start(times:) 94
actor Two: start(times:) 95
actor Two: start(times:) 96
actor Two: start(times:) 97
actor Two: start(times:) 98
actor Two: start(times:) 99
actor Two: start(times:) 100
```
</details>

#### Isolated deinit
<!-- https://github.com/swiftlang/swift-evolution/blob/main/proposals/0371-isolated-synchronous-deinit.md -->

```swift
@globalActor
final actor Moonland {
	static let shared = Moonland()
}

@Moonland
func hello() {}

class MyClass {
	@Moonland deinit {
		hello()
	}
}

MyClass()
```