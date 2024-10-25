# Заглядываем внутрь async let

<!-- 
1. Как создается задача с помощью async let. Это тянет на отдельную статью.
2. Жизненный цикл async let
3. Как компилятор понимает, что задача созданная с помощью async let, будет структурированной или не структурированной ?
 -->

Каждая задача, созднанная с помощью `async let` является дочерней задачей.

### Жизненный цикл async let

1. Перед созданием задачи, компилятор выделяет большое фиксированное кол-во памяти. Вместе с этим устанавливаются внутренние флаги, записи о дочерних задачах и контекст.
2. Инициализация задачи и её прикрепление к родительской задачи.
3. Приостановка и возобновление. Сама по себе `async let` не является точкой приостановки, но при обращение к `async let` задаче с помощью ключевого слова `await`, текущая задача приостанавливает свое выполнение и сохраняет состояние.
4. Обработка ошибок. Задача может вернуть ошибку вместо результата.
5. Отмена задачи при выходе из области видимости.
<!-- https://github.com/apple/swift/blob/8f5980666de3b5c8a7fc6c1ec2891f7f8f91d03b/stdlib/public/Concurrency/AsyncLet.cpp#L332C1-L333C27 -->
6. Удаление записей о дочерних задачах.
7. Освобождение памяти для дочерних задач и самой `async let` задачи.

При создании задачи с помощью `async let` [компилятор определяет](https://github.com/apple/swift/blob/8f5980666de3b5c8a7fc6c1ec2891f7f8f91d03b/stdlib/public/Concurrency/Task.cpp#L586C1-L596C2) является ли она структурированной или не структурированной с помощью вспомогательных методов:

```cpp
static inline bool taskIsStructured(JobFlags jobFlags) {
  return jobFlags.task_isAsyncLetTask() || jobFlags.task_isGroupChildTask();
}

static inline bool taskIsUnstructured(TaskCreateFlags createFlags, JobFlags jobFlags) {
  return !taskIsStructured(jobFlags) && !createFlags.isInlineTask();
}

static inline bool taskIsDetached(TaskCreateFlags createFlags, JobFlags jobFlags) {
  return taskIsUnstructured(createFlags, jobFlags) && !createFlags.copyTaskLocals();
}
```

### На примере

Посмотрим на самом маленьком примере:

```swift
func makeNames() async -> [String] {
	["Siri", "ChatGPT", "Vertex"]
}

async let names: [String] = makeNames() // 1
await names // 2


// Пример с настоящими данными
async let data: Data = URLSession.shared.data(from: .init(string: "https://speedtest.selectel.ru/10MB")!).0
let result = try await data

print("Data size: \(result.count)")

// или более короткой синтаксис
// print("Data size: \(try await data)")
```

Под первым комментарием `// 1` мы создаем задачу с помощью `async let` синтаксиса. Важно понять, что в этом случае задача не приостанавливается, поскольку ключевое слово `await` отсутствует.
Только во второй комментарии мы указываем ключевое слово `await`, с помощью которого задача приостанавливается и возвращает результат.

Чуть ниже я указал пример с настоящими данными (10MB), попробуйте запустить его.
