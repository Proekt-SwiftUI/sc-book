### Практика: AsyncSequence

В этой главе мы разберем технологию `AsyncSequence` в Swift, которая позволяет работать с последовательностями данных асинхронно. AsyncSequence полезен, когда нам нужно получать данные постепенно или по частям, например, при чтении из потока данных или получении событий. Мы рассмотрим реальные примеры использования и узнаем, как управлять асинхронными последовательностями.

#### Что такое AsyncSequence?

`AsyncSequence` — это протокол, который предоставляет интерфейс для асинхронного итератора. Вместо того чтобы возвращать все элементы сразу, как обычные последовательности `Sequence`, `AsyncSequence` возвращает элементы по одному, позволяя обрабатывать их асинхронно.

Это особенно полезно для работы с операциями, которые могут занять неопределенное время, например:

- Сетевые запросы с постепенным поступлением данных.
- Потоки данных, такие как события или сообщения.
- Чтение файлов или данных из внешних источников.

#### Основные концепции

- `AsyncIterator` — объект, который итерирует через элементы асинхронной последовательности.
- `for await` — специальная конструкция для итерации по элементам AsyncSequence.

#### Network status

Давайте рассмотрим простой пример асинхронной последовательности, которая используется для отслеживания статуса сети в реальном времени.
Этот подход позволяет нам удобно и эффективно обрабатывать изменения в статусе интернет-соединения.

```swift
extension NWPath.Status {
	var isConnected: Bool {
		self != .satisfied
	}
}

struct NetworkStatus: AsyncSequence {
	typealias Element = Bool

	func makeAsyncIterator() -> AsyncStream<Element>.Iterator {
		let monitor = NWPathMonitor()
		let monitorQueue = DispatchQueue(label: String(describing: Self.self) + "Queue")

		return AsyncStream { continuation in
			monitor.pathUpdateHandler = { path in
				continuation.yield(path.status.isConnected)
			}

			monitor.start(queue: monitorQueue)

			continuation.onTermination = { @Sendable _ in
				monitor.cancel()
			}
		}
		.makeAsyncIterator()
	}
}
```

1. `NetworkStatus` реализует протокол AsyncSequence. Протокол `AsyncSequence` требует реализации типа алиаса `Element` (является типом данных `Bool`) и метод `makeAsyncIterator()`. Последний возвращает асинхронный итератор, который будет управлять последовательностью значений.

2. В основе нашего подхода лежит `NWPathMonitor` из фреймворка `Network`, который предназначен для отслеживания статуса сети. Этот монитор позволяет следить за изменениями подключения, включая переключения между Wi-Fi, сотовой связью и отсутствием сети.

- `monitor.pathUpdateHandler`: данный обработчик вызывается каждый раз, когда происходит изменение состояния сети. Мы используем его, чтобы передать новое значение состояния через `continuation.yield`.
- `monitor.start(queue:)`: для запуска `NWPathMonitor` необходимо указать, в каком потоке (очереди) будет обрабатываться его работа. Здесь используется очередь `DispatchQueue`, специально созданная для мониторинга.

3. Асинхронные последовательности (sequence), такие как `AsyncStream`, обеспечивают удобный способ создания и управления последовательными данными, которые асинхронно поступают во времени. В нашем коде `AsyncStream` создаёт поток значений на основе изменений статуса сети. Этот поток затем возвращает итератор, который мы передаём дальше.

- `continuation.yield`: передаёт данные (в данном случае — состояние сети) асинхронно в поток, когда изменяется статус сети.
- `continuation.onTermination`: обрабатывает завершение потока. Когда поток завершён, монитор сети `NWPathMonitor` останавливается с помощью метода `cancel()`.


#### Использование

Используя асинхронный цикл `for await`, мы можем последовательно получать значения состояния сети. Этот цикл будет постоянно ждать изменения состояния и реагировать на него.

```swift
var isNetworkAvailable: Bool = false

for await networkAlive in NetworkStatus() {
    isNetworkAvailable = networkAlive
}
```