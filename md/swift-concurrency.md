# Swift Concurrency

초판: 2026-07-05

2026년 Swift Concurrency 의 상태에 대한 간단한 메모.

Swift Concurrency 가 지난 몇 년 진화하면서 관련 키워드만 30가지가 넘게 나왔다;
앞으로 써야할 것, 이젠 좀 쓰지 말아야할 것들이 난무중이다.
여기서는 2026년 흐름만 간단히 적어둔다.
정말 중요하지만 꼭 언급하지 않아도 되는 몇 가지는 뺐다;

자세한 것은 좋은 문서들이 많으니 검색해서 더 읽어보셔야;

<https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency>  
<https://developer.apple.com/documentation/swift/concurrency>  
<https://www.hackingwithswift.com/quick-start/concurrency>

## Asynchronous code

Asynchronous code 란 중간에 실행 단절이 생길 수 있는 코드를 말한다.
네트웍을 호출하거나, IO 나 UI 다이얼로그를 요청하면 거기서 일단 내 코드 실행이 중단된다.
요청한 서비스가 왼료되면 그제서야 이어서 실행된다.
이런 식의 코드들을 Asynchronous code, 비동기 코드라고 한다.

Asynchronous code 구현은 여러 스타일이 있다.

전통적인 Callback, 아직도 많이 쓴다.

    let panel = NSOpenPanel()
    panel.allowsMultipleSelection = false
    panel.canChooseDirectories = true
    ...
    panel.beginSheetModal(for: window) { response in  // <---
        if response == .OK, let url = panel.url {
            completion(url)
        }
    }

async/await, 요즘 주로 사용해야 하는 스타일.

    func scheduleAutoSave(after seconds: Int) {
        autoSaveTask?.cancel()
        autoSaveTask = Task { [weak self] in  // <--
            try? await Task.sleep(for: .seconds(seconds))  // <---
            guard let self else { return }
            guard !Task.isCancelled else { return }
            self.autoSaveFile()
        }
    }

Combine/ReactiveX, 가끔 유용하고 재미도 있지만 공부하는데 시간이 많이 걸려서 여기서는 논외로 한다.

    NotificationCenter.default
        .publisher(for: NSWindow.didResizeNotification, object: window)
        .sink { notification in
            saveWindowSize(window)
        }
        .store(in: &cancellables)

## Parallel code

Parallel code 란 여러 코드 단위가 동시에 실행되는 것이다.

Asynchronous code 가 꼭 Parallel code 인 것은 아니다.
간단한 Asynchronous code 에선 프레임웍에 서비스를 요청한 후 내 코드는 하염없이 기다리는 경우가 대부분이다.

하지만 Parallel code 를 만들려면 Asynchronous 환경이 조성되어 있어야 한다.
Asynchronous code 를 여럿 동시에 실행시킬 수 있게 되면 그게 Parallel code 이기 때문이다.

Concurrency 는 Asynchronous, Parallel 개념을 싸잡아 말할 때 쓰는 단어다.

Parallel code 에도 여러 구현 스타일이 있다.

Thread 직접, 요즘엔 거의 안 쓴다.

    class A {
        func start() {
            let t = Thread(
                target: self, 
                selector: #selector(run(msg:)),
                object: "helloworld"
            )
            t.start()
        }

        @objc func run(msg: String) {
            for i in 1...1000 {
                print("\(i): \(msg)")
            }
        }
    }

    let a = A()
    a.start()

DispatchQueue, 최근까지 많이 썼다.

    func runBackgroundCode2() {
        DispatchQueue.global().async { [unowned self] in  // global background thread
            self.log(message: "On background thread")
            DispatchQueue.main.async {                    // main thread
                self.log(message: "On main thread")
            }
        }
    }

DispatchQueue, parallel 예.

    DispatchQueue.concurrentPerform(iterations: 10) {
        print($0)
    }

async/await + @concurrent, 앞으론 이렇게 해야 한다. 저 아래에서 추가 설명.

    class ImageProcessor {        
        @concurrent   // <-- @concurrent 붙이면 펑션이 백그라운드에서 실행된다.
        func convertImageToGrayscale(data: Data) async -> Data {
            ...
            return data
        }
    }
    
    class GalleryViewModel {
        private let processor = ImageProcessor()
        
        func onUploadButtonTapped() async {        
            let dummyData = Data()
            let processedData = await processor.convertImageToGrayscale(data: dummyData)
        }
    }

## Actor

과거엔 Thread 같은 저수준 API만 있었다.
여러 Thread에서 공통 데이터에 접근할 때는 프로그래머의 수작업 코드가 교통정리를 해야 했다.

Actor는 이 교통정리 작업을 언어차원에서 자동화한다.
Actor는 변수와 펑션들을 Actor 범위로 격리시킨다.
Actor를 넘나들려면 await 해야 한다.

이렇게 생겼다. 여러 병렬 코드에서 logger에 접근해도 교통정리가 알아서 된다.

    actor TemperatureLogger {
        let label: String
        var measurements: [Int]
        private(set) var max: Int
    
        init(label: String, measurement: Int) {
            self.label = label
            self.measurements = [measurement]
            self.max = measurement
        }

        func update(with measurement: Int) {
            measurements.append(measurement)
            if measurement > max {
                max = measurement
            }
        }
    }

    let logger = TemperatureLogger(label: "Outdoors", measurement: 25)
    ...
    await logger.update(with: 30)
    ...
    print(await logger.max)

## MainActor

이제 과거의 Thread 개념을 쓰지 않고 Actor 개념을 쓰기 때문에
과거의 Main Thread 같은 것이 필요한데 그게 MainActor이다.

Swift 프로프램의 기본 실행 공간은 MainActor로 격리된다.
백그라운드 Task 들이 MainActor 안에 있는 변수나 펑션에 접근하려면 await 해야 한다.

과거에는 이렇게 명시적으로 MainActor 표시를 하거나,

    @MainActor
    final class DocumentViewModel: ObservableObject {
    
        @Published var text = ""
    
        func updateText(_ value: String) {
    
            text = value
    
        }
    }

백그라운드 태스크에서 MainActor로 돌아올 때 이런 식으로 했었는데,

    let text = await loadText()
    await MainActor.run {
        self.label.stringValue = text
    }

Swift 6.2 부터 @MainActor 어노테이션을 붙이지 않아도 모든 타입과 펑션은 MainActor 안에 있다고 가정한다.
이제 MainActor를 직접 표시할 필요는 거의 없어졌다고 보면 된다.
기본 코드가 MainActor로 격리되어 있다는 개념만 알면 된다.

## @concurrent

기본 코드가 MainActor에서 실행되고 있다면 백그라운드 실행은 어떻게 해야 할까.

위에 Parallel code 에서 보인데로 펑션에 @concurrent 를 붙이면 된다.
이 펑션들은 CPU 코어 여유 내에서 병렬로 실행된다.

    @concurrent  // <-- 
    func convertImageToGrayscale(data: Data) async -> Data {
        ...
        return data
    }

    ...    
    let processedData = await processor.convertImageToGrayscale(data: dummyData)

하나가 아니라 비슷한 작업을 다발로 실행하려면 TaskGroup 을 사용한다.

    @concurrent  // <--
    public func searchParallel(rootURL: URL, searchText: String) async throws -> [SearchResult] {
        ...
        return try await withThrowingTaskGroup(of: SearchResult?.self) { group in
            for fileItem in fileItems {
                group.addTask(priority: .userInitiated) { // 병렬처리 코드를 group 에 쌓는다.
                    ...
                    return SearchResult(...)
                }
            }
            var results: [SearchResult] = []
            while let result = try await group.next() {  // 병렬처리 결과가 하나씩 튀어나온다.
                ...
                results.append(result)
            }
            return results // 백그라운드에서 모은 결과를 MainActor로 돌려준다.
        }
    }

    Task {
        do {
            searchResults = try await searchParallel(rootURL: rootURL, searchText: searchText)
        } catch {
            ...
        }
    }

참고로 과거엔 백그라운드 실행을 위해 Task.detached 를 썼는데 이제 필요없다.
@concurrent 만 쓰면 된다.

## Task

다 좋은데 Swift 코드의 기본은 Synchronous 하다. async/await 코드들은 Task 나 async 펑션 안에서만 쓸 수 있다.

Synchronous 흐름중에 Asynchronous 코드를 쓰려면 Task로 구획을 줘야 한다.

    func scheduleAutoSave(after seconds: Int) {  // <-- Synchronous
        guard seconds > 0 else { return }
        autoSaveTask?.cancel()
        autoSaveTask = Task { [weak self] in  // <-- 여기서부터 Asynchronous
            try? await Task.sleep(for: .seconds(seconds))  // <-- Task 안에서 await 가능
            guard let self else { return }
            guard !Task.isCancelled else { return }
            self.autoSaveFile()
        }
    }

또는 async 펑션 안에서.

    func openFiles(_ urls: [URL]) async {  // <--
        do {
            try await bufferManager.openFilesParallel(urls)  // <--
        } catch {
            ...
        }
    }


Task 자체가 백그라운드 / 병렬처리 시작을 만들지 않는다.
Task 블럭은 일단 옆에 미뤄져 있다가, 현재 Actor가 하고 있던 일이 끝나면 같은 Actor에서 이어서 실행된다.
사용자 코드는 주로 MainActor에서 실행된다.
즉 Task는 현재 실행되고 있는 Actor 환경을 물려받고, 대기하고 있다가, 거기에 이어서 실행되는 단위이다.
다시 반복하지만 백그라운드 병렬처리를 하려면 Task가 아니라 @concurrent 펑션으로 해야 한다.
과거에 쓰던 Task.detached는 이제 쓰이지 않는다.

## Sendable

Actor 공간 간에는 아무 데이터나 넘길 수 없다.
여러 스레드가 동시에 사용해도 안전한 타입만 넘길 수 있다.
이 조건을 충족하는 타입이 Sendable 이다.

Value, Actor, Immutable classes 등은 기본적으로 Sendable 하다.
Value 는 복사로 넘어가기 때문에 받은 곳에서 수정해도 문제가 없다.
Actor 는 스스로 교통정리를 하기 때문에 문제가 없다.
Immutable classes 오브젝트들은 프로퍼티가 변하는 경우가 없기 때문에 문제가 없다.

그렇다면 문제가 있을 때는,

## Sendable + nonisolated

사용자가 만드는 타입들은 기본적으로 모두 MainActor로 격리된다.
(package 모듈은 예외다.)
그래서 Actor 경계를 넘나들어야 하는 타입들은 밸류 타입이라도 @MainActor 를 지워야한다.
이를 위해 nonisolated 를 사용한다. 액터간 사용자 타입을 넘기기 위해선 일단 필수다.

    nonisolated struct SearchResult: Identifiable {
        let id = UUID()
        let url: URL
        let title: String
        ...
    }

## Sendable + nonisolated + @unchecked Sendable

struct 가 class 프로퍼티를 가지면 Sendable 이 깨진다.
여러 스레드에서 class 오브젝트를 동시에 수정하면 안되기 때문이다.
개발자가 봤을 때 동시에 수정할 일이 없는 경우나
Lock 을 사용해서 수작업으로 동기화한다면 강제로 Sendable 하다고 정의할 수 있다.
이렇게 하고 싶을 때 @unchecked 를 Sendable 앞에 붙인다.

    nonisolated struct Box: @unchecked Sendable {
        let buffer: GPXBuffer  // <-- class type
        ...
    }

nonisolated, @unchecked, Sendable, 처음엔 복잡해 보이지만 곰곰히 생각해 보면 써야하는 이유가 모두 분명하다.

## Sendable + Lock

수작업 Lock이 필요하다면 OSAllocatedUnfairLock 사용하면 된다.

    nonisolated struct IntSequenceWithLockIterator: IteratorProtocol, Sendable {
        private let lock: OSAllocatedUnfairLock<Int>

        public init(current: Int = 0) {
            self.lock = OSAllocatedUnfairLock<Int>(initialState: current)
        }

        public func next() -> Int? {
            return lock.withLock { state in
                defer {
                    state += 1
                }
                return state
            }
        }
    }

## 마무리

언제 뭘 써야하나.

내가 직접 병렬 실행 코드를 만들어야 할 일은 많지 않다.
내 모든 코드는 보통 MainActor에서 동작한다.
기본 애플 프레임웍이 제공하는 비동기 함수들을 쓸 때는 
Task + async/await 정도만 쓰면 된다.
Asynchronous code.


내 코드 자체에서 CPU를 많이 쓰는 작업을 백그라운드에서 돌려야할 경우,
async 함수에 @concurrent를 붙여 쓰면 된다.
Parallel code.

백그라운드에서 비슷한 작업을 다발로 돌려야 할 경우, @concurrent 에 TaskGroup 까지 붙여쓰면 된다.

Actor 경계를 넘어다니는 타입을 만들어야 할 경우.
Sendable 하게 만든다.
단순 밸류라면 자동으로 Sendable 하다.
자동으로 안 될 때는 원인에 따라 nonisolated, @unchecked Sendable 을 같이 사용해야 한다.

Task { } 는 작업 단위를 만들고 현재 Actor 뒤에 스케쥴링한다. 백그라운드나 병렬 작업과 아무 상관이 없다.
