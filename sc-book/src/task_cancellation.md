# Task cancellation (отмена задачи)

Отмена задачи необходима для индикации о том, что приложению больше не нужен результат задачи.
В случае отмены, задача должна остановиться, вернув частичный результат или выдать ошибку.
Выполнение задачи можно представить как последовательность периодов, в течение которых она выполнялась. Каждый такой период заканчивается в точке приостановки `await` или завершает задачу. Такие периоды выполнения представлены экземплярами [PartialAsyncTask][partialTask]:

```swift
typealias PartialAsyncTask = UnownedJob
```

> Для взаимодействия с частичным результатом необходимо реализовать кастомный исполнитель (executor).

<!-- Показать пример реализации кастомного executora -->

В примере с супом, мы может остановить приготовление, если клиент ушел, решил заказать пюрешку с котлеткой или пришло время закрыть кухню.
Что может привести к отмене задачи ? В случае **SC** (структурированных задач), задачи отменяются неявно при выходе из области видимости, хотя мы можем вручную вызвать метод [cancelAll()][cancellAll] для группы задач [TaskGroup][taskGroup], для отмены текущих и будующих дочерних задач.

![Cancel Group][cancel_task]

В случае с неструктурированными задачи, отмена происходит явно с помощью метода [cancel()][https://developer.apple.com/documentation/swift/task/cancel()]

![Cancel UC][cancel_task_uc]

В результате, отмена родительской задачи приводит к отмене всех дочерних задач.



> Отмена задач является кооперативной, поэтому дочерние задачи не останавливаются немедленно/мгновенно.

It simply sets the "isCancelled" flag on that task. Actually acting on the cancellation is done in your code.
Cancellation is a race.
If the task is cancelled before our check, "makeSoup" throws a "SoupCancellationError".
If the task is cancelled after the guard executes, the program will carry on preparing the soup.

---

Remember, cancellation does not stop a task from running, it only signals to the task that it has been cancelled and should stop running as soon a possible.

Cancellation is a purely Boolean state; there’s no way to include additional information like the reason for cancellation. This reflects the fact that a task can be canceled for many reasons, and additional reasons can accrue during the cancellation process.

[partialTask](https://developer.apple.com/documentation/swift/partialasynctask)
[cancellAll](https://developer.apple.com/documentation/swift/taskgroup/cancelall())
[taskGroup](https://developer.apple.com/documentation/swift/taskgroup)
[cancel_task](../resources/task_cancel.png)
[cancel_task_uc](../resources/task_cancel_uc.png)
