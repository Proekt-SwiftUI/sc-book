# Приоритет задачи

В этом разделе мы рассмотрим, как дерево задач помогает распространить приоритет и избегать инверсию приоритетов.

Приоритет — это способ сообщить операционной системе, насколько срочной является та или иная задача.
Некоторые задачи, такие как реагирование на нажатие кнопки, должны выполняться немедленно, иначе вы увидите зависание в приложении. Другие задачи, как предварительная загрузка данных с сервера, должны выполняться в фоне, незаметно для пользователя.

Инверсия приоритетов случается, когда задачи с более высоким приоритетом ожидают результата от задачи с более низким приоритетом.
По-умолчанию, дочерние задачи наследуют приоритет от родительской задачи. Функция `makeSoup` запущена на среднем (medium) приоритете, поэтому все дочерние задачи будут выполняться так же на среднем приоритете.

![Medium Priority][medium_priority]

Представьте, что ресторан посетил **VIP** гость, который захотел супа. В таком случае, мы бы хотели приготовить и доставить суп как можно скорее, поэтому установим высокий (high) приоритет для задачи.
При ожидании (`await`) супа, приоритет всех дочерних задач повышается, **гарантируя**, что ни одна задача с высоким приоритетом не будет ожидать задачу с низким приоритетом. Это позволяет избегать инверсии приоритетов.

![High Priority][high_priority]

Concurrency runtime использует очередь приоритетов для планирования задач, поэтому задачи с более высоким приоритетом выполняются раньше задач с более низким приоритетом.
Задача сохраняет повышенный приоритет до конца времени жизни.

> Мы можем повысить приоритет задачи, но механиза отмены повышения приорита не существует.

За приоритет задач отвечает структура [TaskPriority][priority_struct]. Открепленные задачи (detached), созданные с помощью [Task.detach(priority:operation:)][detached_task] не наследуют приоритет, поскольку они не имеют родительской задачи (не прикреплены к текущей задаче).

В качестве дополнительного примера, попробуйте запустить код ниже:

```swift
func executeAt(priority: TaskPriority) async {
    let currentPriority = Task.currentPriority

    while (priority != currentPriority) {
        print("Task priority = \(currentPriority) != fn \(priority)")
        try await Task.sleep(for: .seconds(1))
    }
}
```

[medium_priority]: images/medium_priority.png
[high_priority]: images/high_priority.png
[priority_struct]: https://developer.apple.com/documentation/swift/taskpriority
[detached_task]: https://developer.apple.com/documentation/swift/task/detached(priority:operation:)-3lvix
