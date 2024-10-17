### Практика: async let и withTaskGroup

В этой главе мы **наглядно** разберемся с реальными примерами, а так же поймем когда использоваться `async let`, а когда `withTaskGroup`.

Пример приложения написан на самом современном стеке: `SwiftUI` и макросе `@Observation` для управления данными. Легенда: существует профиль кота и список имен других котов.
Необходимо получить:
1. аватарку кота
2. список имен других котов.

#### actor CatDownloader

Для этих целей я заранее подготовил актор `CatDownloader`, который будет скачивать аватарку и список котов.

```swift
struct Cat: Decodable, Identifiable {
	let id: Int
	let name: String
	let breed: String
	let photoUrl: String
}

actor CatDownloader {
	var decoder: JSONDecoder {
		let decoder = JSONDecoder()
		decoder.keyDecodingStrategy = .convertFromSnakeCase
		return decoder
	}
	
	// Вы можете использовать собственный URL для аватарки
	let avatarURL: URL = .init(string: "https://image.lexica.art/full_webp/4f696c7b-f280-43ce-80f0-dd211c7f553")!
	let catAPI: URL = .init(string: "https://raw.githubusercontent.com/Proekt-SwiftUI/sc-book/refs/heads/main/practice_data/cat_api.json?token=GHSAT0AAAAAACY4C77IXVFZ522PDU33FK6AZYPWQEQ")!
	
	// Скачиваем аватарку кота. В случае неудачи — возвращаем nil
	// Для простоты примера я не стал делать дополнительные проверки, статус код и прочее.
	func donwloadCatAvatar() async throws -> UIImage? {
		let imageData: Data = try await URLSession.shared.data(for: .init(url: avatarURL)).0
		
		guard let catImage: UIImage = UIImage(data: imageData) else {
			print("Картинка коцыка не найдена и/или не доступна")
			return nil
		}
		
		return catImage
	}
	
	// Скачиваем список котов, декодируем в массив [Cat] и возвращаем результат
	func getListOfCats() async throws -> [Cat] {
		let catsData: Data = try await URLSession.shared.data(for: .init(url: catAPI)).0
		let cats: [Cat] = try decoder.decode([Cat].self, from: catsData)
		return cats
	}
}
```

#### class CatViewModel

Для взаимодействия между актором и вьюхой, создадим `CatViewModel` с применением макроса `@Observable`:

```swift
@Observable
final class CatViewModel {
	let downloader = CatDownloader()
	
	var catImage: Image?
	var listOfCats: [Cat] = []
	
	func downloadCatImage() async {
		do {
			guard let finalImage: UIImage = try await downloader.donwloadCatAvatar() else {
				return
			}
			
			await MainActor.run {
				catImage = Image(uiImage: finalImage)
			}
			
			print("Аватар кота загрузился")
		} catch {
			print(error.localizedDescription)
		}
	}
	
	func getListOfCats() async {
		do {
			let decodedCats: [Cat] = try await downloader.getListOfCats()
			
			await MainActor.run {
				listOfCats = decodedCats
			}
			print("Список котов загрузился")
		} catch {
			print(error.localizedDescription)
		}
	}
}
```

#### MagicWithCat + SwiftUI

Переходим к реализации графического представления:

```swift
struct MagicWithCat: View {
	@State
	private var catData = CatViewModel()
	
	var body: some View {
		VStack {
			CatImageView(image: catData.catImage)
				.ignoresSafeArea(.all, edges: .top)
			
			Spacer()
			
			List(catData.listOfCats) { cat in
				Text(cat.name)
			}
		}
		.animation(.smooth, value: catData.catImage)
		.task {
			await catData.downloadCatImage()
			await catData.getListOfCats()
		}
	}
}

struct CatImageView: View {
	let image: Image?
	
	var body: some View {
		if let image {
			image
				.resizable()
				.aspectRatio(contentMode: .fill)
				.frame(height: 350)
				.clipShape(.rect(cornerRadius: 16))
		} else {
			RoundedRectangle(cornerRadius: 16)
				.foregroundStyle(.secondary)
				.frame(height: 350)
				.overlay {
					HStack {
						Text("Loading cat image")
							.font(.title2.bold())
							.foregroundStyle(.white)
						
						ProgressView()
							.tint(.white)
							.padding(7)
							.background(.secondary.opacity(0.25), in: .circle)
					}
				}
		}
	}
}
```

Вы можете не обращать внимания на код вью. Самое интересное, как и загрузка данных, происходит в модификаторе `.task {…}`.
При открытии превью канваса, мы увидим что первым загрузается аватар кота и после этого загружаются данные массива. В консоли вы увидите следующий вывод:

```text
Аватар кота загрузился
Список котов загрузился
```

Несмотря на асинхронные методы, вызов этих методов осуществляется **последовательно**. Вы можете легко в этом убедиться, поменяв методы местами и в таком случае, список котов загрузиться первым, а после аватарка:

```swift
.task {
    await catData.getListOfCats()
    await catData.downloadCatImage()
}
```

![Cat Practice image](images/practice_cat.png)

Чтобы использовать современный параллелизм и загружать данные асинхронно (без ожидания предыдущей задачи), мы можем изменить вызов методов создав 2-е структурированные задачи с помощью `async let` синтаксиса. В таком случае, задача, которая завершила свое выполнение первой — отдаст результат (или неудачу) первой:

```swift
.task {
    async let avatar: Void = catData.downloadCatImage()
    async let cats: Void = catData.getListOfCats()
    
    await avatar
    await cats
}

// Список котов загрузился
// Аватар кота загрузился
```

#### Рефакторинг

Мы можем изменить нашу вью модель, добавив отдельный метод для примера выше.
Такой способ оптимален, когда нет необходимости в большом кол-ве вызовов `async let` или цикле `for loop`:

```swift
// 1-ый способ
// Подходит для небольшого кол-ва асинхронных вызовов.
// Порядов вызова await не важен.
func getDataViaAsyncLet() async {
    async let image: Void = downloadCatImage()
    async let cats: Void = getListOfCats()

    await cats
    await image
}
```

И вызовем метод `getDataViaAsyncLet`:

```swift
.task {
    await catData.getDataViaAsyncLet()
}
```

Второй способ подходит, когда нам необходимо вызвать большое кол-во `async let` задач или требуется создать цикл `for await loop` для значительно большого кол-ва дочерних задач с помощью `TaskGroup`:

```swift
// 2-ой способ
// В том случае, когда у нас много причин для создания async let задач
// Поэтому можно воспользоваться taskGroup
func downloadViaTaskGroup() async {
    await withTaskGroup(of: Void.self) { group in
        group.addTask { await self.downloadCatImage() }
        group.addTask { await self.getListOfCats() }
    }
}
```

И вызываем в модификаторе `task`:

```swift
.task {
    await catData.downloadViaTaskGroup()
}
```
