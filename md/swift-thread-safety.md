# Swift Thread Safety

초판: 2024-04-15

## var, let,

var 에 대해서 여러 스레드에서 동시 접근하지 말 것.

let 에 대해서, 그것이 밸류 타입이라면, 여러 스레드에서 읽을 수 있다.

하지만 밸류 타입이 레퍼런스 타입 프로퍼티를 품고 있다면,
레퍼런스 타입의 모든 프로퍼티가 리커시브하게 let 이어야 여러 스레드에서 읽을 수 있다.

<https://forums.swift.org/t/understanding-swifts-value-type-thread-safety/41406/14>

## Global var, Static properties

    let maximumNumberOfLoginAttempts = 10
    var currentLoginAttempt = 0

    struct MyType {
        static let storedGlobal1: String = "foo"
        static var storedGlobal2: String = "foo"
        ...
    }

(신기하게도) Swift 전역 변수, 상수는 lazy 수식이 없어도 lazy 하다.
초기화는 Thread Safe 하다.

Static properties 들 또한 lazy 하다.
이들 초기화도 Thread Safe 하다.

Thread Safe 한데 심지어 퍼포먼스 문제도 없다고 한다.\
<https://mikeash.com/pyblog/friday-qa-2014-06-06-secrets-of-dispatch_once.html>

Swift globals and static members are atomic and lazily computed\
<https://www.jessesquires.com/blog/2020/07/16/swift-globals-and-static-members-are-atomic-and-lazily-computed/>

Global and Local Variables\
<https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Global-and-Local-Variables>

## 싱글턴

    class Singleton {
    	static let shared = Singleton()
    	private init() {}
        ...
    }

일반적인 싱글턴 패턴도 쓰고 싶으면 쓸 수 있다.

## 하지만

전역 변수의 초기화만 Thread Safe 할 뿐,
변수 값을 변경하거나 의미있는 작업 단위에서 Thread safety 를 보장받으려면
적당한 방법들을 써야 한다.

옛날 사람들은 lock, semaphore, serial dispatch queue, 이런 것들을 썼는데,

## Lock

    class Counter {
        private var lock = OSAllocatedUnfairLock()
        private var count = 0
        func increment() {
            lock.withLock {
                count += 1 
            }
        } 
    }

장점, 제일 빠름,

## DispatchQueue

    class Queue {
        private var counter = 0.0
        private let queue = DispatchQueue(label: "counter")
        
        func increment() {
            queue.sync {
                counter += 1.2
            }
        }
    }

장점이 없다, 앞으로 안 쓰일 것 같다,

## Actor

    actor Counter {
        var count = 0
        func increment() {
            count += 1 
        }
    }
    
    var counter = Counter()
    
    await counter.increment()
    await print(counter.count)

actor 에 대한 외부 접근이 자동으로 스케쥴링된다.
내부에서 수동으로 락을 잡을 필요가 없다.

단점, Lock 에 비해 느리다,

<https://www.avanderlee.com/swift/actors/>

## MainActor

    @MainActor
    final class HomeViewModel {
         ....
    }

    final class HomeViewModel {
        @MainActor var images: [UIImage] = []
        ....
        @MainActor func updateViews() {
            ....
        }
    }

클래스 접근을 Main thread 로 제한하고 싶으면 `@MainActor` 어노테이션을 쓰면 된다.

<https://www.avanderlee.com/swift/mainactor-dispatch-main-thread/>

## Benchmark

    unfair locks:     1.0
    actors:           6.66
    dispatch queue:   16.2

OSAllocatedUnfairLock 이 빠르긴 하다;

<https://forums.swift.org/t/simple-state-protection-via-actor-vs-dispatchqueue-sync/66184/7>

