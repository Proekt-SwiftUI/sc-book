### Практика: async let и withTaskGroup

В этой главе мы **наглядно** разберемся с реальными примерами и поймем, когда использовать `async let`, а когда `withTaskGroup`.

Пример приложения написан на современном стеке: `SwiftUI` и макросе `@Observable` для управления данными. Легенда: существует профиль кота и список имен других котов.
Необходимо получить:

1. Аватарку кота.
2. Список имен других котов.

#### actor CatDownloader

Для загрузки данных я подготовил актор `CatDownloader`, который будет скачивать аватарку и список котов.

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
	
	// URL для аватарки
	let avatarURL: URL = .init(string: "https://image.lexica.art/full_webp/4f696c7b-f280-43ce-80f0-dd211c7f553")!
	let catAPI: URL = .init(string: "https://raw.githubusercontent.com/Proekt-SwiftUI/sc-book/refs/heads/main/practice_data/cat_api.json?token=GHSAT0AAAAAACY4C77IXVFZ522PDU33FK6AZYPWQEQ")!
	
	// Скачиваем аватарку кота. В случае неудачи возвращаем nil
	func downloadCatAvatar() async throws -> UIImage? {
		let imageData: Data = try await URLSession.shared.data(for: .init(url: avatarURL)).0
		
		guard let catImage: UIImage = UIImage(data: imageData) else {
			print("Картинка кота не найдена и/или не доступна")
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
			guard let finalImage: UIImage = try await downloader.downloadCatAvatar() else {
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

Теперь создадим графический интерфейс:

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

Основное внимание стоит уделить модификатору .task {…}, где происходит загрузка данных. Сначала загружается аватар кота, затем список имен. В консоли вы увидите:

```text
Аватар кота загрузился
Список котов загрузился
```
Это происходит последовательно, так как сначала выполняется первая задача `downloadCatImage()`, а затем вторая `getListOfCats()`. Мы можем поменять их местами, чтобы сначала загружался список:

```swift
.task {
    await catData.getListOfCats()
    await catData.downloadCatImage()
}
```

![Cat Practice image](images/practice_cat.png)

Чтобы сделать вызовы параллельными, мы можем использовать `async let`. Тогда обе задачи будут выполняться одновременно:

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

1. Пример с `async let`

Этот метод хорош для небольшого числа асинхронных задач, где порядок выполнения не имеет значения:

```swift
// Подходит для небольшого количества задач, порядок вызова await не важен
func getDataViaAsyncLet() async {
    async let image: Void = downloadCatImage()
    async let cats: Void = getListOfCats()

    await cats
    await image
}
```

Вызовем задачу:

```swift
.task {
    await catData.getDataViaAsyncLet()
}
```

2.	Пример с withTaskGroup

Этот метод полезен, когда требуется большое количество асинхронных вызовов или работа с циклами:

```swift
// Подходит для большого количества задач или работы с циклами
func downloadViaTaskGroup() async {
    await withTaskGroup(of: Void.self) { group in
        group.addTask { await self.downloadCatImage() }
        group.addTask { await self.getListOfCats() }
    }
}
```

Вызовем задачу:

```swift
.task {
    await catData.downloadViaTaskGroup()
}
```

#### Когда использовать async let и когда withTaskGroup

- async let — подходит для параллельного выполнения небольшого количества задач, когда порядок завершения не важен. Хорошо, если задачи не зависят друг от друга.
- withTaskGroup — используется для более сложных сценариев, когда нужно динамически создавать и управлять задачами, например, при большом количестве запросов или циклах с дочерними задачами.
