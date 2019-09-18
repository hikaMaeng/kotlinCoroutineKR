* 본 글은 코틀린 공식레포의 문서를 번역한 글로서 원문은 아래 링크에 있습니다.
<a href="https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md" rel="noopener noreferrer" target="_blank">https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md</a>

## 개요

코틀린 코루틴에 대해 기술하는 글로 코루틴의 컨셉은 다음의 것들을 부분적으로 커버한다.

* generators/yield
* async/await
* composable/delimited continuations

### 목표:
* Future의 특정 구현체나 다른 라이브러리에 의존성이 없게 한다.
* 제네레이터와 async/await의 사용성을 동등하게 커버한다.
* 자바의 NIO 등 이미 존재하고 있는 비동기 API들을 코틀린 코루틴으로 감싸 활용할 수 있게 한다.

**역주**
컨셉은 오히려 ES2018에 포함된 <a href="https://github.com/tc39/proposal-async-iteration" rel="noopener noreferrer" target="_blank">Asynchronous Iterators</a> 과 비슷합니다만 다른 플랫폼이 await의 컨티뉴에이션 조건을 만족하는 특정 객체(ES에서는 프라미스)를 강요하는데 비해 코틀린 코루틴은 직접 컨티뉴에이션의 resume을 노출함으로서 어떤 비동기API라도 연동할 수 있게 합니다.


## 사용 분야
코루틴은 suspendable computation의 인스턴스라 할 수 있다(suspendable computation이란 특정 위치에서 일시중지(suspend)할 수 있고 나중에 다른 쓰레드에서 실행을 재개할 수 있는 것을 말한다)
코루틴끼리는 서로가 데이터를 주고 받으며 호출할 수 있어 협업 멀티태스킹을 위한 매커니즘을 구현할 수 있다.

### Asynchronous computations
코루틴이 사용될 첫 번째 예는 Asynchronous computations분야다(c# 등 다른 언어에서는 async/await로 처리된다) 먼저 비동기처리가 콜백으로는 어떻게 작동하는지 살펴보기 위해 비동기IO를 다뤄보자.

```kotlin
// 비동기적으로 `buf`를 읽고, 완료되면 람다를 실행한다
inChannel.read(buf) {
    // 이 람다는 읽기가 끝나면 실행된다
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)
    
    // 비동기적으로 `buf`에 쓰며, 완료되면 람다를 실행한다
    outChannel.write(buf) {
        // 이 람다는 쓰기가 완료되면 실행된다
        ...
        ...
        outFile.close()          
    }
}
```

위의 콜백 예에서 들여쓰기가 점점 증가하며 중첩이 1회 이상 될 것을 쉽게 예상할 수 있다("callback hell"에 대해 검색해보라)

위와 동일한 기능을 코루틴에서는 동기코드처럼 표현할 수 있다(IO api를 코루틴에 맞게 수정한 경우)

```kotlin
launch {
    // 비동기적으로 읽는 동안 일시정지
    val bytesRead = inChannel.aRead(buf) 
    // 읽기가 완료되야 이 줄부터 시작
    ...
    ...
    process(buf, bytesRead)
    // 비동기 쓰기가 완료될 때까지 일시정지
    outChannel.aWrite(buf)
    // 쓰기가 완료되어야 이 줄부터 시작
    ...
    ...
    outFile.close()
}
```

aRead()와 aWrite()는 특별한 일시 중지 기능으로 코드의 실행을 일시 중단할 수 있으며(실행 중인 쓰레드를 차단하는 것은 아님) 호출이 완료되면 다시 시작될 수 있다. 위의 콜백 스타일에 비하면 기능은 동일하지만 더 읽기 쉽다.

명시적으로 전달된 콜백은 루프의 중간에서 비동기 호출이 까다롭지만, 코루틴에서는 다음과 같은 코드가 보통이다.

```kotlin
launch {
    while (true) {
        // 비동기로 읽는 동안 일시정지
        val bytesRead = inFile.aRead(buf)
        // 읽기가 끝나면 계속 진행
        if (bytesRead == -1) break
        ...
        process(buf, bytesRead)
        // 비동기로 쓰는 동안 일시정지
        outFile.aWrite(buf) 
        // 쓰기가 끝나면 계속 진행
        ...
    }
}
```

예외 처리도 코루틴쪽 좀 더 편할 것을 예상할 수 있다(역주:예외 처리도 동기화 문법에 맞게 되어있으므로)

### Futures
또 다른 비동기 스타일인 Future로 이미지 오버레이를 적용해보자.
```kotlin
val future = runAfterBoth(
    loadImageAsync("...original..."), // Future 생성
    loadImageAsync("...overlay...")   // Future 생성
) {
    original, overlay ->
    ...
    applyOverlay(original, overlay)
}
```

코루틴으로 재작성해보면
```kotlin
val future = future {
    val original = loadImageAsync("...original...") // Future 생성
    val overlay = loadImageAsync("...overlay...")   // Future 생성
    ...
    // 이미지 로딩시까지 일시정지
    // 둘 다 로딩이 완료되면 `applyOverlay(...)` 실행
    applyOverlay(original.await(), overlay.await())
}
```

위에서 c#이나 js처럼 특별한 키워드(async/await)가 없으며 들여쓰기도 깊어지지 않고 동기적으로 코드를 표현하고 있다.

### Generators
지연 시퀀스 등에 사용되는 c#, python, js등의 제네레이터도 코루틴으로 대체할 수 있다. 이런 지연 시퀀스는 순차적인 코드로 보이지만 런타임에 요청된 요소만 계산하게 된다.
```kotlin
// fibonacci의 타입은 Sequence<Int>
val fibonacci = sequence {
    yield(1) // 첫 번째 피보나치 수
    var cur = 1
    var next = 1
    while (true) {
        yield(next) // 다음 피보나치 수
        val tmp = cur + next
        cur = next
        next = tmp
    }
}
```

이 코드는 피보나치 수의 지연 시퀀스를 만들어낸다. 잠재적으로 무한 수열이므로 take()등을 사용할 수 있다.

```kotlin
println(fibonacci.take(10).joinToString())
```

제네레이터는 동기 구문을 지원하므로 흐름제어와 예외처리를 모두 사용할 수 있다.

```kotlin
val seq = sequence {
    yield(firstItem) // 일시 중지

    for (item in input) {
        if (!item.isValid()) break // 더 이상 요소를 만들어내지 않음
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // 일시 중지
    }
    
    try {
        yield(lastItem()) // 일시 중지
    }
    finally {
        // finalization 코드
    }
} 
```

이 방식에서 yieldAll(sequence)표현이 가능하기 때문에 지연 시퀀스끼리의 결합을 단순하고 효율적으로 구현할 수 있다(역주: es6의 yield* 처럼)

**역주**
sequence{} 를 이용해 만들어내는 제네레이터는 거의 js의 제네레이터와 동일하지만 js의 yield가 외부와 값을 주고 받을 수 있는데 비해 SequenceScope의 yield는 값을 외부로 주는 것 밖에 할 수 없다.

## 비동기 UI
보통 UI작업이 발생하는 단일 이벤트 스레드가 있다. 그 외의 다른 스레드에서 UI 상태를 수정하는 것은 불가능하며, UI스레드에서 실행시킬 수 있는 라이브러리를 제공하는 경우가 대부분이다(예를 들어 Swing은 SwingUtilities.invokeLater, JavaFX는 Platform.runLater, Android는 Activity.runOnUiThread 등) 다음 Swing의 예를 보자.

```kotlin
makeAsyncRequest {
    // 이 람다는 비동기 요청이 완료되면 실행된다
    result, exception ->
    
    if (exception == null) {
        // UI에 결과 표시
        SwingUtilities.invokeLater {
            display(result)   
        }
    } else {
       // 예외처리
    }
}
```

사실 앞에서 다룬 콜백지옥과 유사하다. 코루틴의 우아한 해결을 보자.

```kotlin
launch(Swing) {
    try {
        // 비동기적으로 요청을 만들 때까지 일시정지
        val result = makeRequest()
        // UI에 결과를 표시, Swing컨텍스트가 UI쓰레드에서 처리될 것을 보장함
        display(result)
    } catch (exception: Throwable) {
        // 예외처리
    }
}
```

모든 예외가 언어 본연의 구조로 처리된다.

**역주**
launch(컨텍스트) 에서 코드가 실행될 쓰레드를 고를 수 있는데 Swing을 선택했으므로 람다의 내용은 Swing컨텍스트에서 실행된다. 하지만 makeRequest() 함수 내부에서 본인만의 컨텍스트를 또 정의할 수 있기 때문에 해당 코드는 백그라운드 쓰레드에서 실행되게 할 수 있다.

### 더 많은 사용 분야
코루틴은 다음을 포함하여 더 많은 곳에 사용될 수 있다.
* 채널 기반의 동시성(고루틴과 채널같은..)
* 액터 기반의 동시성
* 사용자 인터렉션이 필요한 백그라운드 프로세스(모달 다이얼로그같은)
* 통신 프로토콜:각 액터를 상태머신이 아닌 시퀀스로 구현
* 웹 응용 프로그램 워크플로우 : 사용자 등록, 전자 메일 유효성 검사, 로그인 (일시 중지된 코루틴은 직렬화되어 DB에 저장될 수 있다..미친)

## 코루틴 개요
이 절에서는 코루틴을 만들고 제어하는 표준 라이브러리와 이를 사용하는 언어 매커니즘의 개요를 설명한다.

### 용어
* coroutine - suspendable computation의 인스턴스. 쓰레드와 유사한 개념으로 실행할 코드 블록을 갖고 비슷한 라이프사이클을 갖는다. 단, 생성되고 시작되지만 특정 스레드에 바인딩되지는 않는다. 한 스레드에서 실행을 일시 중단하고 다른 스레드에서 재개할 수 있다. 또한 Future나 Promise처럼 값이나 예외로 완료된다.

* suspending 함수 - suspend modifier로 표시된 함수. 다른 suspending 함수를 호출해 현재 실행 스레드를 차단하지 않고 코드 실행을 일시 중단할 수 있다. suspending 함수는 일반 코드에서 호출할 수 없고 다른 suspend 함수나 suspending 람다에서만 호출할 수 있다. 예를 들어 .await() 및 yield()는 앞 서 예처럼 라이브러리에 정의된 함수를 일시 중지 한다. 표준 라이브러리는 다른 모든 suspending 함수를 정의할 때 사용되는 기본 suspending 함수를 제공한다.

* suspending 람다 - 코루틴에서 실행할 코드블록. 일반 람다와 같은 모양이지만 함수타입은 suspend modifier가 된다. 보통 람다처럼, suspending 람다는 suspending 함수의 간단한 익명 구문이다. suspending 함수를 호출해 현재 실행 스레드를 차단하지 않고 코드 실행을 일시 중지 할 수 있다. 예를 들어 launch, futers, sequence 함수 다음에 중괄호로 묶인 코드 블록은 모두 suspending 람다다.

* suspending 함수 타입 - suspending함수와 suspending람다의 타입이다. 보통 함수의 타입과 같지만 suspend 키워드를 붙여준다. 예를 들어 suspend()->Int는 인자없이 Int를 반환하는 suspending 함수다. suspend fun foo():Int 처럼 정의된 suspending 함수도 마찬가지 타입이다.

* 코루틴 빌더 - suspending 람다를 인자로 받는 함수로 코루틴을 생성하고 어떤 경우는 결과에 접근할 수 있는 옵션을 제공한다. 예를 들어 launch{}, future(), sequence()는 코루틴 빌더다. 표준 라이브러리는 다른 코루틴빌더를 정의할 수 있는 기본 코루틴 빌더를 제공한다.

* suspension point(일시중지점) - 코루틴이 실행 중 일시중지한 곳. 구문 상 일시중지점은 suspending함수의 호출이지만 실제 일시중지는 suspending 함수가 표준 라이브러리의 suspend를 실행해야 일어난다.

* continuation(컨티뉴에이션) - 일시중지점에서의 코루틴 상태다. 개념적으로는 중지점 이후의 실행을 나타낸다.
```kotlin
sequence {
    for (i in 1..10) yield(i * i)
    println("over")
}  
```

위의 예에서 yield를 호출할 때마다 일시 중지된다. 이러한 개념으로는 남은 실행을 컨티뉴에이션으로 표현하므로 10개의 컨티뉴에이션을 갖게 된다. 첫번째 루프에서 i = 2인 짖ㅁ에서 일시정지하고 두 번째 실행에서는 i = 3일때 일시정지한다. 마지막에 이르러 "over"를 출력하면 컨티뉴에이션은 완료된다. 코루틴이 생성되면 시작되기 전에 컨티뉴에이션을 초기화하며 이때 타입은 Continuation<Unit>이 된다. 이 컨티뉴에이션이 전체 실행을 구성하게 된다.

처음 밝혔듯 코루틴의 중요 요구사항은 유연성이다. 기존의 비동기 API와 다양한 예를 지원하고 컴파일러에 하드코딩된 부분을 최소화하려 한다. 결과적으로 컴파일러는 오직 suspending 함수, suspending 람다 및 suspending함수 타입만 지원한다. 표준 라이브러리에는 몇 가지 기본 요소가 있지만 나머지는 애플리케이션 라이브러리에 있다.

### 컨티뉴에이션 인터페이스
표준인터페이스에 정의된 Continuation 인터페이스를 살펴보자(역주:kotlinx.coroutines패키지에 있었으나 지금은 kotlin.coroutines에 정식으로 있다)

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resumeWith(result: Result<T>)
}
```

컨텍스트는 코루틴과 관련된 정보는 담고 있으며 resumeWith메소드는 코루틴 완료시 성공 또는 실패를 보고하는 사용되는 콜백이다.
같은 패키지 내부에 편의용 수신함수 두 개가 정의되어있다.
```kotlin
fun <T> Continuation<T>.resume(value: T)
fun <T> Continuation<T>.resumeWithException(exception: Throwable)
```

### Suspending functions
await()같은 일반적인 일시중지 기능의 구현은 다음과 같다.
```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) // 일반적으로 future가 완료될 때
                cont.resume(result)
            else // future가 예외로 종료된 경우
                cont.resumeWithException(exception)
        }
    }
```

suspend 키워드는 이 함수가 코루틴의 실행을 일시 중지 시킬 수 있다는 것을 나타냅니다. 이 CompletableFuture<T>의 확장 함수는 실제 사용 시 좌에서 우로 따라 자연스레 읽힌다.

```kotlin
doSomethingAsync(...).await()
```

suspending함수는 보통 함수도 부를 수 있지만 일시중지를 위해서는 반드시 다른 suspending함수를 불러야 한다. 특히 이 awiat구현은 표준라이브러리에 정의된 최상위 suspending함수인 suspendCoroutine을 호출한다.

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

suspendCoroutine이 코루틴 내에서 호출될 때 컨티뉴에이션의 인스턴스가 코루틴의 상태를 캡쳐하고 지정된 블록에 인자로 전달된다. 코루틴을 다시 실행하려면 블록은 해당 쓰레드나 다른 쓰레드에서 continuation.resumeWith()를 직접 호출하거나 continuation.resume() 또는 continuation.resumeWithException()을 호출해야 한다.

코루틴의 일시정지 상태는 resumeWith를 호출하지 않고 suspendCoroutine블록이 반환될 때 발생한다. 블록 내부에서 복귀하기 전에 cotinuation의 resume이 발생했다면 코루틴은 일시 중지되지 않은 것으로 간주하고 계속 실행된다.

continuation.resumeWith()에 전달된 결과는 suspendCoroutine호출의 결과가 되며, 이는 .await()의 결과가 된다.

같은 continuation은 resume을 두번 호출할 수 없으면 그럴 경우 IllegalStateException을 발생시킨다.

### 코루틴 빌더
suspending 함수는 보통 함수에서는 호출할 수 없으므로 표준 라이브러리에서는 suspending 스코프가 아닌 곳에서 코루틴을 실행할 수 있는 시작함수를 제공한다. 다음은 간단한 launch{} 코루틴 빌더다.
```kotlin
fun launch(context: CoroutineContext = EmptyCoroutineContext, block: suspend () -> Unit) =
    block.startCoroutine(Continuation(context) { result ->
        result.onFailure { exception ->
            val currentThread = Thread.currentThread()
            currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
        }
    })
```
위 구현은 Continuation(context){..}함수에게 resumeWith함수의 몸체와 컨텍스트를 제공하는 단축 표현이다. 이 컨티뉴에이션은 completion continuation으로 block.startCoroutine(...)에게 전달된다.

코루틴의 완료는 completion continuation을 실행한다. resumeWith함수는 코루틴이 성공이나 실패로 완료될 때 실행된다. launch는 실행 후 잊어버리는 코루틴이므로, Unit을 반환 타입으로 하는 suspending함수를 정의하고 실제로 resu,e함수의 결과를 무시한다. 코루틴이 예외로 완료된다면 현재 쓰레드의 예외 핸들러가 보고하게 된다.

startCoroutine은 표준라이브러리에 확장함수로 정의되어있다.
```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
fun <R, T> (suspend  R.() -> T).startCoroutine(receiver: R, completion: Continuation<T>)
```
startCoroutine은 코루틴을 만들고 현재 쓰레드에서 즉시 실행을 시작하여 첫번째 일시정지점까지 간 뒤 반환한다. 일시중지점은 코루틴의 몸체의 suspending함수를 발동시키고, 코루틴이 다시 실행될지는 해당 suspending함수에 달려있다.

### 코루틴 컨텍스트
코루틴 컨텍스트는 코루틴을 추가할 수 있는 사용자 정의 객체들의 영속화된 세트다. 코루틴의 스레드 정책, 로깅, 보안, 코루틴실행의 트랜젝션측면, 코루틴식별자 및 이름 등을 담당하는 객체가 포함될 수 있다. 코루틴과 코루틴 컨텍스트에 대한 간단한 비유를 들어보자.
코루틴을 가벼운 쓰레드라 생각해보자. 이 때 코루틴 컨텍스트는 쓰레드 지역변수의 컬렉션 같은 것이다. 쓰레드 지역변수가 가변적인데 비해 코루틴 컨텍스트는 불변적이라는 것만 다르다. 코루틴은 굉장히 가볍기 때문에 변경이 필요한 경우 새 코루틴을 쉽게 시작할 수 있으므로 불변성은 코루틴에 대한 큰 제약은 되지 않는다.

표준 라이브러리에는 컨텍스트요소에 대한 구상객체가 포함되어 있지 않다. 하지만 인터페이스와 추상클래스를 통해 합성으로 라이브러리에 정의할 수 있고, 이를 통해 같은 컨텍스트의 요소들이 평화롭게 공존할 수 있다.

개념 상, 코루틴 컨텍스트는 요소의 인덱싱 셋이며 각 요소는 고유키가 있다. 사실 셋과 맵이 혼합된 구조다. 요소는 맵처럼 키를 갖지만 키는 셋처럼 요소와 직접 연결된다. 표준 라이브러리는 kotlin.coroutines패키지 안의 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html" target="_blank">CoroutineContext</a>에 최소한의 인터페이스로 정의되어있다.

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?
    fun <R> fold(initial: R, operation: (R, Element) -> R): R
    operator fun plus(context: CoroutineContext): CoroutineContext
    fun minusKey(key: Key<*>): CoroutineContext

    interface Element : CoroutineContext {
        val key: Key<*>
    }

    interface Key<E : Element>
}
```
```CoroutineContext``` 자신은 네 가지 핵심 연산을 제공한다.
* get연산자는 키를 인자로 받아 요소를 반환한다.
* fold함수는 컨텍스트 내의 모든 요소를 반복할 수단을 제공한다.
* plus연산자는 Set의 plus와 비슷하게 작동하며 왼쪽요소와 오른쪽 요소를 결합한 요소를 반환하며 이 때 키는 왼쪽요소의 키가 된다.
* minus연산자는 그 키를 포함하지 않게 컨텍스트를 반환한다.

코루틴 컨텍스트에 있는 ```Element```는 컨텍스트 자체다. 이 요소만 있는 싱글톤 컨텍스트인 셈이다.  이 개념이 합성 컨텍스트를 생성할 수 있게 하는데 코루틴 컨텍스트 요소의 라이브러리 정의를 가져와 그들을 +로 결합하면 된다. 예를 들어 하나의 라이브러리가 유저의 승인 정보를 갖는 ```auth```요소를 정의하고 다른 라이브러리에서는 몇몇 실행 컨텍스트 정보를 갖는 ```threadPool``` 객체를 정의했다면 ```launch{}```코루틴 빌더를 합성 컨텍스트인 ```launch(auth + threadPool){...}```로 사용할 수 있다.

참고 : kotlinx.coroutines는 코루틴 실행을 백그라운드 스레드의 공유 풀로 디스패치하는 Dispatchers.Default 객체를 포함하여 여러 컨텍스트 요소를 제공한다.

표준 라이브러리는 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html" target="_blank">EmptyCoroutineContext</a>를 제공하는데 이는 아무 요소도 포함하지 않는 ```CoroutineContext```의 인스턴스다.

모든 서드파티 컨텍스트 요소는 표준 라이브러리의 kotlin.coroutines 패키지에서 제공하는 AbstractCoroutineContextElement 클래스를 확장해야 한다. 라이브러리에 정의된 컨텍스트 요소는 다음 스타일을 권장한다. 다음 예는 현재 사용자 이름을 저장하는 가상의 인증 컨텍스트 요소다.

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

이 예제는 <a href="https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/auth.kt" target="_blank">여기</a>서 찾을 수 있다.

요소 클래스의 companion object를 컨텍스트의 ```Key```로 정의하여 컨텍스트의 엘리먼트를 유연하게 다룰 수 있게 합니다. 다음은 현재 사용자의 이름을 확인해야하는 suspending 함수의 가상 구현이다.

```kotlin
suspend fun doSomething() {
    val currentUser = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
    // do something user-specific
}
```
suspending함수에서 사용할 수 있는  kotlin.coroutines 패키지의 최상위 수준 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html" target="_blank">coroutineContext</a> 속성을 사용해 현재 코루틴의 컨텍스트를 검색한다.

### 컨티뉴에이션 인터셉터

비동기 UI 사용 사례를 생각해보자. 비동기 UI 애플리케이션은 다양한 suspend 함수가 임의의 스레드에서 코루틴 실행되더라도 코루틴 본문 자체가 항상 UI 스레드에서 실행되도록 해야한다. 이는 연속 인터셉터를 사용하여 수행됩니다. 우선, 우리는 코 루틴의 수명주기를 완전히 이해해야합니다. launch {} 코 루틴 빌더를 사용하는 코드 스 니펫을 고려하십시오.



