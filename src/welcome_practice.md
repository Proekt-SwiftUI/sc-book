# Практика

Данный раздел посвящен практическому использованию **SC**.
По мере продвижения, главы будут усложняться.

1. Fetch and Display Images
Build an app that fetches images from an API like the Dog API and displays them in a collection view. Use async/await to handle the asynchronous network requests and update the UI when the images are loaded.

2. Async Sequence Example
Create an app that demonstrates the usage of async sequences in Swift. For example, you could build a chat app that receives messages asynchronously and displays them in a table view. Use AsyncSequence and AsyncIteratorProtocol to handle the message stream.

3. Structured Concurrency with Tasks
Develop an app that showcases structured concurrency using Swift's Task API. For instance, you could build a game where multiple tasks (representing different game elements) run concurrently. Use Task.detached or TaskGroup to manage the tasks and demonstrate task prioritization.

4. Пример использования API, `Codable`.

5. Локальные уведомления `@preconcurrency`.

6. Обработка подключения к сети — `AsyncStream`.

7. Различные примеры с тестов компилятора. Async/Await in WebSockets

8. Unit Testing Async/Await Logic
Create a sample app and write unit tests for its async/await logic. Use the new XCTest methods to run tests asynchronously and prevent deadlocks by awaiting expectations. This project will help you practice testing async/await code and learn best practices for unit testing concurrency.