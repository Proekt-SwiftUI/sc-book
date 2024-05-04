# О чем эта книга ?

В данной книге я попытался рассказать о фундаментальных основах SC на языке Swift, принципах и разнице между Unstrured Concurrency, ручной и автоматической отмене задач, приоритете задачи и отладки.

### Зачем нам нужен новый подход ?

<!-- Structured concurrency enables you to reason about concurrent code using well-defined points where execution branches off and runs concurrently, and where results from that execution rejoin, similar to how "if"-blocks and "for"-loops define how control-flow behaves in synchronous code. Concurrent execution is triggered when you use an "async let", a task group, or create a task or detached task.
Results rejoin the current execution at a suspension point, indicated by an "await".
Not all tasks are structured. -->

Structured concurrency (далее **SC**) позволяет рассуждать о конкуретном вычислении используя специальные точки, позволяет узнать о разветвлениях, конкурентных вычислениях и увидеть результат вычислений, подобно тому, как работает блок условия `if else` в синхронном коде.

Конкурентная задача начинается, когда вы используете `async let`, создаете задачу, в том числе открепленную (detached) задачу или группу задач.
Задача возобновляются в точке приостановке (suspention point), обозначаемой `await`.
Однако не все задачи являются структурированными.

> Структурированные задачи создаются с помощью `async let` и групп задач, а не структурированные — используя `Task` и `Task.detached`.

![Structured VS Unstructured][StructVSunsctruct]

<!-- Structured tasks live to the end of the scope where they are declared, like local variables, and are automatically cancelled when they go out of scope, making it clear how long the task will live.
Whenever possible, prefer structured Tasks. The benefits of structured concurrency discussed later do not always apply to unstructured tasks.
Before we dive into the code, let's come up with a concrete example. -->

Структурированные задачи живут до конца области видимости, подобно локальным переменным и автоматически отменяются при выходе из области видимости, давая понять как долго задача должна жить.

> Старайтесь, по возможности, использовать **SC**, вместо не структурированных задач (Unstructured).

О преимуществах **SC** вы можете прочитать ниже, а пока посмотрим на код:

```swift

```

[StructVSunsctruct]: ../book/beginning/sc_vs_uc.png
