# Swift Combine Chain

초판: 2024-12-05

Swift Combine 체인이 만들어지는 과정을 정리.

위치가 옮겨질 수도 있겠지만, 전체 코드는 대략 아래 어디쯤 있을 것이다.\
https://github.com/drypot/hello-swift/tree/main/HelloSwiftFrameworkTests/ReactiveX

## 절차

Combine 체인에 참여하는 오브젝트들은 위 아래로 5 번이상 횡단을 한다.

1차 횡단에서는 위에서 아래로 publisher 나 operator 같은,
초기화 데이터만 가지고 있는 가벼운 밸류타입들을 만든다.
1차 횡단 마지막에 최종 subscriber 도 만든다. subscriber 들은 모두 클래스 타입이다.

데이터 이동을 위해 실제로 만들어 지는 체인은 subscription 들의 체인이다.
중간 subscription 들은 publishing 과 subscribing 기능을 동시에 가진다.
subscription 들은 모두 클래스 타입이다.

2차 횡단에서는 아래에서 위로 올라가며 subscription 오브젝트들을 만든다.
2차 횡단에서는 아래에서 위로 절반만 연결된 상태가 된다.
제일 꼭때기 publisher subscription 까지 만들었으면 다시 아래로 3차 횡단을 시작한다.

3차 횡단에서는 상위 subscription 들을 하위 subscription 에 연결한다.
3차 횡단 마지막에 최종 subscriber 에 도착하면 다시 위로 4차 횡단이 시작된다.

4차 횡단에서는 아래에서 위로 데이터 request 를 보낸다.
이 request 연쇄는 제일 꼭대기 publisher subscription 까지 올라간다.

5차 횡단에서는 꼭대기 publisher subscription 에서 아래로 데이터를 내려보낸다.
데이터가 더 없으면 완료 신호를 내려 보낸다. 마지막 subscriber 까지 데이터가 내려오면 최종 소비한 후 추가 데이터 요구를 위로 올려보낸다.

## 1차 횡단

1차 횡단에서는 위에서 아래로 publisher 나 operator 같은 가벼운 밸류타입들이 만들어 진다.
1차 횡단 마지막에 최종 subscriber 도 만드는데 subscriber 들은 모두 클래스 타입이다.

publisher 나 operator 는 초기화 데이터만 가지고 있는 가벼운 밸류타입이다.
데이터 이동을 위해 실제로 만들어 지는 체인은 subscription 들의 체인이다.
중간 subscription 들은 publisher 와 subscriber 기능을 동시에 가진다.
subscription 들은 모두 클래스 타입이다.

### publisher 생성

    let publisher = CustomPublisher<Int, Never>(values: [1, 2, 3, 4, 5])

    ...
    
    struct CustomPublisher<Output, Failure: Error> : Publisher {

        let values: [Output]

        init(values: [Output]) {
            self.values = values
        }    

최상단 publisher 부터 만들기 시작한다.
publisher 들은 가벼운 밸류 타입이다.
이후 사용할 기초 데이터만 보관하고 끝낸다.

### 잠시 Protocol 의 Associated Type 에 대해서

Publisher 의 Output, Failure 는 associated type 이고 추상 타입이다.
struct 구현에서 제네릭 없이 `var values: [Output]` 같은 식으로 바로 사용하지 못한다.
추상 타입이니까 컴파일러가 뭘 해야할지 모른다.

해서 Publisher protocol 의 associcated type 을 구체화 해야 한다.
associcated type 이 쓰이는 곳마다 구체 타입을 일일이 적어도 된다.
typealias 로 구체 타입을 명시적으로 적어줄 수도 있다.
generic 으로 구체 타입을 입힐 수도 있다.

generic 의 type parameter 들은 구체 타입이다.
associcated type 과 같은 이름의 type parameter 를 쓸 수 있다.
잘 생각해 보면 protocol 입장에서도 사용자 입장에서도 이게 은근 괜찮다.
generic 의 구체 타입을 typealias 비스무리하게 쓸 수 있어서 편리하다.

위에서 Output, Failure 는 Generic 구체 타입이다.
associcated type 과 같은 이름으로 필요한 자리들을 훌륭하게 매꾼다.

### operator 생성

    let operator_ = CustomOperator(upstream: publisher) { $0 * 2 }

    ...

    struct CustomOperator<Upstream, Output>: Publisher
    where Upstream: Publisher {
    
        typealias Failure = Upstream.Failure

        let upstream: Upstream
        let map: (Upstream.Output) -> Output

        init(upstream: Upstream, map: @escaping (Upstream.Output) -> Output) {
            self.upstream = upstream
            self.map = map
        }

밸류 타입의 가벼운 operator 를 생성한다.
upstream publisher 와 이후 사용할 기초 데이터 보관까지만 한다.
upstream 을 인자로 받긴 하지만 연결된 상태는 아니다.

operator 도 publisher 이므로 자기만의 Output, Failure 타입이 필요하다.
Output 은 보통 Upstream 에서 유도할 수 없는 새로운 타입이므로 새로운 제네릭 타입을 투입한다.
Failure 는 보통 Upstream.Failure 에서 유도한다.

### subscriber 생성

    let subscriber = CustomSubscriber()

    ...

    class CustomSubscriber: Subscriber {
        typealias Input = Int
        typealias Failure = Never

        let logger = SimpleLogger<Int>()

        init() {
        }

최종 subscriber 를 가볍게 생성한다.
subscriber 들은 클래스 타입이다.

subscriber 는 Input, Failure associated type 들을 사용한다.
로깅하기 편하게 제네릭 대신 Int, Never 로 구체 타입을 적어줬다.

필요한 3종 밸류가 모두 만들어 졌다.
publisher, operator, subscriber.

## 2차 횡단

2차 횡단에서는 아래에서 위로 올라가며 subscription 오브젝트들을 만든다.
가볍게 초기화되고 아래에서 위로 절반만 연결된 상태가 된다.
제일 꼭때기 publisher subscription 까지 만들었으면 다시 아래로 3차 횡단을 시작한다.

### operator.subscribe(subscriber)

    operator_.subscribe(subscriber)

Publisher 인터페이스에는 비슷한 두 메서드가 있다.
`subscribe(subscriber)` 와 `receive(subscriber)`.

`subscribe(subscriber)`는 자기 일이 끝나면 `receive(subscriber)`를 호출한다.
`receive(subscriber)`가 더 원초적인 기능을 한다.
`subscribe(subscriber)`에는 Combine 에서 자체적으로 사용하는 서비스 기능들이 추가되어 있다.

커스텀 publisher 나 operator 를 만들 때는 `receive(subscriber)`만 구현한다.
호출할 때는 가능하면 `subscribe(subscriber)`를 사용한다.

operator 에 최종 subscriber 를 `operator_.subscribe(subscriber)` 하면 subscription 체인이 만들어 지기 시작한다.

### operator.receive(subscriber)

    operator_.receive(subscriber)

    ...
        
    struct CustomOperator<Upstream, Output>: Publisher
    where Upstream: Publisher {

        ...
        
        func receive<S>(subscriber: S)
        where S: Subscriber, Output == S.Input, Failure == S.Failure {
            let subscription = CustomOperatorSubscription(
                subscriber: subscriber,
                map: map
            )
            upstream.subscribe(subscription)
        }
    }


`operator.receive(subscriber)`는 operator subscription 을 생성하고 전달 받은 subscriber 와 기초 데이터를 담아 놓는다.

operator subscription 은 upstream publisher 에 대해 subscriber 로 동작한다.
upstream 의 subscribe 메서드를 콜해서 상위 publisher 에 자신을 등록한다.
이 작업은 중간에 사용되는 operator 갯수에 따라 반복적으로 일어난다.

### OperatorSubscription

    class CustomOperatorSubscription<S, Input, Failure>: Subscriber, Subscription
    where S: Subscriber, Failure: Error, S.Failure == Failure {

        private var subscriber: S?
        private let map: (Input) -> S.Input
        private var subscription: Subscription?

        init(subscriber: S, map: @escaping (Input) -> S.Input) {
            self.subscriber = subscriber
            self.map = map
        }

operator subscription 의 모습은 이러하다.
subscription 들은 모두 클래스 타입이다.

제네릭 type parameter 들이 여러 개 등장한다.
S 는 하위 subscriber 를 위한 타입이다.
subscription 자체도 상위 subscription 에 대해 subscriber 기능을 하기 때문에
다른 subscriber 들 처럼 Input, Failure 타입을 지정해야 한다.

### publisher.receive(subscriber)

    struct CustomPublisher<Output, Failure: Error> : Publisher {

        let values: [Output]

        init(values: [Output]) {
            self.values = values
        }

        func receive<S>(subscriber: S)
        where S: Subscriber, Failure == S.Failure, Output == S.Input {
            let subscription = CustomPublisherSubscription(
                subscriber: subscriber,
                values: values
            )
            subscriber.receive(subscription: subscription)
        }
    }

상방 2차 횡단 끝으로 최상단 `publisher.receive(subscriber)`에 도착한다.
publisher subscription 을 만든다.
위에 뭐가 없으므로 더 subscribe 할 일은 없다.
대신 `subscriber.receive(subscription)`을 호출해서 하방으로 3차 횡단을 시작한다.

## 3차 횡단

3차 횡단에서는 상위 subscription 들을 하위 subscription / subscriber 에 연결한다.
2차 횡단에서 아래 subscription들을 상위 subscription에 연결했다면,
3차 횡단에서는 상위 subscription 을 하위 subscription 에 연결한다.
3차 횡단 마지막에 최종 subscriber에 도착하면 다시 위로 4차 횡단이 시작된다.

## subscription.receive(subscription)

    class CustomOperatorSubscription<S, Input, Failure>: Subscriber, Subscription
    where S: Subscriber, Failure: Error, S.Failure == Failure {

        private var subscriber: S?
        private let map: (Input) -> S.Input
        private var subscription: Subscription?

        ...
        
        func receive(subscription: Subscription) {
            self.subscription = subscription
            subscriber?.receive(subscription: self)
        }

상위 subscription 을 받으면 subscription 변수에 일단 저장해둔다.
4차 상방 횡단에서 사용하게 된다.
그리고 하위 subscription / subscriber 에 자신을 연결한다. 반복.

## subscriber.receive(subscription)

    class CustomSubscriber: Subscriber {
        typealias Input = Int
        typealias Failure = Never

        let logger = SimpleLogger<Int>()

        ...
        
        func receive(subscription: Subscription) {
            logger.append(-99)
            subscription.request(.unlimited)
        }

최하단 subscriber 에 도착한다.
이제 상방으로 다시 4차 횡단을 시작한다.

## 4차 횡단

4차 횡단에서는 아래에서 위로 데이터 request 를 보낸다.
이 request 연쇄는 제일 꼭대기 publisher subscription 까지 올라간다.

### operator.request(Subscribers.Demand)

    class CustomOperatorSubscription<S, Input, Failure>: Subscriber, Subscription
    where S: Subscriber, Failure: Error, S.Failure == Failure {

        ...

        func request(_ demand: Subscribers.Demand) {
            subscription?.request(demand)
        }

오퍼레이터의 request 메서드들을 거치고.


### publisher.request(Subscribers.Demand)

    class CustomPublisherSubscription<S, Output>: Subscription
    where S: Subscriber, S.Input == Output {
        private var subscriber: S?

        let values: [Output]
        var index = 0

        ...
        
        func request(_ demand: Subscribers.Demand) {
            var remainingDemand = demand
            while remainingDemand > 0 && index < values.count {
                let value = values[index]
                index += 1
                _ = subscriber?.receive(value)
                remainingDemand -= 1
            }
            if index == values.count {
                subscriber?.receive(completion: .finished)
            }
        }

최상단 publisher.request 에 도착하면 요청된 만큼의 데이터를 하방으로 보내기 시작한다.

## 5차 횡단

5차 횡단에서는 꼭대기 publisher subscription 에서 아래로 데이터를 내려보낸다.
데이터가 더 없으면 완료 신호를 내려 보낸다.

### operator.receive(Input)

    class CustomOperatorSubscription<S, Input, Failure>: Subscriber, Subscription
    where S: Subscriber, Failure: Error, S.Failure == Failure {

        private var subscriber: S?
        private let map: (Input) -> S.Input
        private var subscription: Subscription?

        ...

        func receive(_ input: Input) -> Subscribers.Demand {
            return subscriber?.receive(map(input)) ?? .none
        }

        func receive(completion: Subscribers.Completion<Failure>) {
            subscriber?.receive(completion: completion)
        }

`operator.receive(Input)` 메서드가 위에서 내려오는 데이터를 받는다.
적당한 처리를 하고 아래 subscriber 에 데이터를 계속 전달한다.

### subscriber.receive(Input)

    class CustomSubscriber: Subscriber {
        typealias Input = Int
        typealias Failure = Never

        let logger = SimpleLogger<Int>()

        ...
        
        func receive(_ input: Input) -> Subscribers.Demand {
            logger.append(input)
            return .unlimited
        }

        func receive(completion: Subscribers.Completion<Never>) {
            logger.append(99)
        }
    }

최하단 `subscriber.receive(Input)`에서 데이터를 소비한다.
그리고 위로 추가 데이터를 요구하는 Demand 를 리턴한다.
데이터가 끝날 때까지 이 흐름을 반복한다.

