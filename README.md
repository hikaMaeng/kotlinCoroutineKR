* 본 글은 코틀린 공식레포의 문서를 번역한 글로서 원문은 아래 링크에 있습니다.
<a href="https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md" rel="noopener noreferrer" target="_blank">https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md</a>

## 개요

이 글은 코틀린의 코루틴에 대한 설명이다. 코루틴 컨셉은 다음에 나오는 것들로 잘 알려져 있거나 부분적으로 겹친다.

* generators/yield
* async/await
* composable/delimited continuations

### 목표
* Future나 비슷한 라이브러리의 특정 구현체에 대한 의존성이 없음
* generator와 async/await의 사용법을 대등한 수준으로 대체함
* 자바 NIO나 Futures의 다른 구현체 등, 이미 존재하고 있는 비동기 API를 코틀린 코루틴으로 감싸 활용할 수 있게 함

>**역주** : 컨셉은 오히려 ES2018에 포함된 <a href="https://github.com/tc39/proposal-async-iteration" rel="noopener noreferrer" target="_blank">Asynchronous Iterators</a> 과 비슷합니다만 다른 플랫폼이 await의 컨티뉴에이션 조건을 만족하는 특정 객체(ES에서는 프라미스)를 강요하는데 비해 코틀린 코루틴은 직접 컨티뉴에이션의 resume을 노출함으로서 어떠한 비동기API라도 연동할 수 있게 합니다.

### 목차
생략합니다. =o=;

## 사용 분야
코루틴은 suspendable computation(유보가능 연산)의 인스턴스라 할 수 있다. 따라서 특정 지점에서 연기시킨 코루틴은 나중에 다른 쓰레드에서 재게될 수 있다. 코루틴끼리는 서로가 데이터를 주고 받으며 호출할 수 있으므로 협업 멀티태스킹을 위한 매커니즘을 구현할 수 있다.
**역주**
suspendable computation이란 프로그램이 특정 위치에서 유보(suspend)할 수 있고 나중에 다른 쓰레드에서 실행을 재개할 수 있는 것을 말합니다.

### Asynchronous computations(비동기 연산)
코루틴을 사용해 볼 첫 예제는 비동기 처리분야다(c# 등 다른 언어에서는 async/await로 처리된다) 먼저 콜백을 통한 비동기 처리를 살펴보기 위해 비동기IO를 다뤄보자(아래 예제에선 관련 API를 간략화했다)

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

이 예에서 콜백 내부에 콜백이 있다는 걸 알아차릴 수 있다. 이 방식은 수 많은 보일러플래이트로부터 우릴 구해준다(콜백에게 buf인자를 명식적으로 전달할 필요가 없다. 콜백은 클로저를 통해 `buf`를 사용한다) 하지만 매번 들여쓰기가 증가하여 중첩이 1회 이상 될 것을 쉽게 예상할 수 있다("callback hell"에 대해 검색해보라)

위와 동일한 연산을 코루틴에서는 동기코드처럼 표현할 수 있다(IO api를 코루틴 요구사항에 맞게 수정한 라이브러리가 공급된 경우를 가정함)

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

`aRead()`와 `aWrite()`는 특별한 `suspending function`으로 코드의 실행을 일시 중단할 수 있으며(실행 중인 쓰레드를 차단하는 것은 아님) 호출이 완료되면 다시 시작될 수 있다. `aRead()` 후의 모든 코드가 람다로 감싸져 `aRead()`의 콜백으로 전달되고 `aWrite()`도 마찬가지로 생각해보면 이 코드가 위와 동일하지만 더 읽기 쉽다는 걸 알 수 있다.

코루틴은 특정 구현체에 의존적이지 않은 generic한 방식으로 사용되는 것을 목표로 하고 있으며 이 예제에 등장한 `launch{}, aRead(), aWrite()`는 단지 코루틴과 함께 작동하도록 라이브러리로 제공되는 함수일 뿐이다. `launch`는 코루틴 빌더로 코루틴을 만들어 실행하고 `aRead / aWrite`는 암묵적으로 컨티뉴에이션을 받아들이는 특별한 suspending 함수다(컨티뉴에이션은 단지 generic 콜백이다)

>코드에 등장하는 `launch{}`코루틴 빌더 섹션에서, aRead()는 콜백 감싸기섹션에서 다룬다.

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

예외 처리도 코루틴쪽 좀 더 편할 것을 예상할 수 있다(**역주** : 예외 처리도 동기화 문법에 맞게 되어있으므로 ^^)

### Futures(자바의 퓨쳐)

또 다른 비동기 스타일로 `Future`가 있다. 퓨쳐는 또한 promise나 deferred로 알려져있다. 이를 이용해 이미지 오버레이를 적용하는 API작성에 사용해보자.

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

> 위 코드 상의 `future{}`는 퓨쳐생성 섹션에서, `await()`는 suspending function에서 다룬다.

보다 자연스럽고 들여쓰기도 없는 합성로직(여기서는 보이지 않지만 예외처리를 포함한)이 표현되고, future를 지원하기 위해 c#이나 js처럼 특별한 키워드(async/await)가 없다. `future{}`나 `.await{}`는 단지 라이브러리에서 제공되는 함수일뿐이다.

### Generators(제네레이터)

지연 연산 시퀀스 등에 사용되는 제네레이터도 코루틴으로 대체할 수 있다(c#, python, js등에서는 `yield`키워드로 통제한다) 이런 지연 연산 시퀀스는 순차적인 코드로 보이지만 런타임에 요청된 요소만 계산하게 된다.

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
> `1, 1, 2, 3, 5, 8, 13, 21, 34, 55`가 출력될 것이다. <https://github.com/Kotlin/coroutines-examples/blob/master/examples/sequence/fibonacci.kt" target="_blank">여기</a>서 코드를 내려받을 수 있다

제네레이터의 강력함은 임의의 제어흐름을 지원한다는 것이다. `while`을 비롯해 `if`, `try / catch / finally`등 모든 제어문을 포함할 수 있다.

```kotlin
val seq = sequence {
    yield(firstItem) // 일시 중지 위치

    for (item in input) {
        if (!item.isValid()) break // 더 이상 요소를 만들어내지 않음
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // 일시 중지
    }
    
    try {
        yield(lastItem()) // 일시 중지 위치
    }
    finally {
        // finalization 코드
    }
} 
```

>`sequnce{}`와 `yield()`는 제한된 suspension섹션에서 다룬다.

이 구조에서는 `sequence{}`나 `yield()`처럼 yieldAll(sequence)도 라이브러리 함수로 만들어 사용할 수 있으므로 지연 연산 시퀀스끼리의 결합을 단순하고 효율적으로 구현할 수 있다(**역주** : es6의 yield* 처럼)

>**역주** : `sequence{}` 를 이용해 만들어내는 제네레이터는 거의 js의 제네레이터와 동일하지만 js의 `yield`가 외부와 값을 주고 받을 수 있는데 비해 `SequenceScope`의 `yield`는 값을 외부로 주는 것 밖에 할 수 없습니다.

## 비동기 UI
보통 UI를 포함하는 애플리케이션은 모든 UI작업을 처리하는 단일 이벤트 디스패치 스레드가 있다. 그 외의 다른 스레드에서 UI 상태를 수정하는 것은 보통 허용하지 않는다. 모든 UI라이브러리들은 어떤 실행구문을 UI쓰레드에서 작동하게 해주는 기본 요소를 제공합니다. 예를 들면 Swing의 `SwingUtilities.invokeLater`나 JavaFX의 `Platform.runLater`가 있고 Android는 `Activity.runOnUiThread` 등이 제공된다. Swing 애플리케이션에서 어떤 비동기 연산 후에 그 결과를 UI에 표시하는 예를 보자.

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

이 코드는 앞에서 다룬 콜백지옥과 유사하다. 이것 역시 코루틴으로 우아하게 해결할 수 있다.

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

>`Swing`컨텍스를 위한 이 코드는 컨티뉴이에션 인터셉터 섹션에서 다룬다.

모든 예외가 언어 본연의 구조로 처리된다.

>**역주** : `launch(컨텍스트)` 에서 코드가 실행될 쓰레드를 고를 수 있는데 `Swing`을 선택했으므로 람다의 내용은 `Swing`컨텍스트에서 실행된다. 하지만 `makeRequest()` 함수 내부에서 본인만의 컨텍스트를 또 정의할 수 있기 때문에 해당 코드는 백그라운드 쓰레드에서 실행되게 할 수 있습니다.

### 더 많은 사용 분야

코루틴은 다음을 포함하여 더 많은 곳에 사용될 수 있다.

* 채널 기반의 동시성(고루틴과 채널같은..)
* 액터 기반의 동시성
* 사용자 인터렉션이 필요한 백그라운드 프로세스(모달 다이얼로그같은)
* 통신 프로토콜 : 각 액터를 상태머신이 아닌 시퀀스로 구현
* 웹 응용 프로그램 워크플로우 : 사용자 등록, 전자 메일 유효성 검사, 로그인 (일시 중지된 코루틴은 직렬화되어 DB에 저장될 수 있다..미친)

## 코루틴 개요

이 절에서는 코루틴을 만들고 제어하는 표준 라이브러리와 이를 사용하는 언어 매커니즘의 개요를 설명한다.

### 용어
* coroutine - suspendable computation(일시정지 가능한 연산)의 인스턴스. 쓰레드와 유사한 개념으로 실행할 코드 블록을 갖고 비슷한 라이프사이클을 갖는다. 단, 생성되고 시작되지만 특정 스레드에 바인딩되지는 않는다. 한 스레드에서 실행을 일시정지하고 다른 스레드에서 재개할 수 있다. 또한 Future나 Promise처럼 값이나 예외로 완료된다.

* suspending 함수 - `suspend` 키워드를 붙여 만든 함수. 다른 suspending 함수를 호출해 현재 실행 스레드를 차단하지 않고 코드 실행을 일시 중단할 수 있다. suspending 함수는 일반 코드에서 호출할 수 없고 다른 suspend 함수나 suspending 람다(아래서 설명)에서만 호출할 수 있다. 예를 들어 `.await()` 및 `yield()`는 앞 서 예처럼 라이브러리에 정의된 suspending 함수다. 표준 라이브러리는 다른 suspending 함수를 정의할 때 사용되는 기본 suspending 함수를 제공한다.

* suspending 람다 - 코루틴에서 실행할 코드블록. 일반 람다와 같은 모양이지만 함수타입은 suspend modifier가 된다. 보통 람다처럼, suspending 람다는 suspending 함수의 간단한 익명 구문이다. suspending 함수를 호출해 현재 실행 스레드를 차단하지 않고 코드 실행을 일시 중지 할 수 있다. 예를 들어 launch, futers, sequence 함수 다음에 중괄호로 묶인 코드 블록은 모두 suspending 람다다.

* suspending 함수 타입 - suspending함수와 suspending람다의 타입이다. 보통 함수의 타입과 같지만 suspend 키워드를 붙여준다. 예를 들어 suspend()->Int는 인자없이 Int를 반환하는 suspending 함수다. suspend fun foo():Int 처럼 정의된 suspending 함수도 마찬가지 타입이다.

* 코루틴 빌더 - suspending 람다를 인자로 받는 함수로 코루틴을 생성하고 어떤 경우는 결과에 접근할 수 있는 옵션을 제공한다. 예를 들어 launch{}, future(), sequence()는 코루틴 빌더다. 표준 라이브러리는 다른 코루틴빌더를 정의할 수 있는 기본 코루틴 빌더를 제공한다.

* suspension point(유보지점) - 코루틴이 실행 중 유보한 곳. 구문 상 유보지점은 suspending함수의 호출이지만 실제 일시중지는 suspending 함수가 표준 라이브러리의 suspend를 실행해야 일어난다.

* continuation(컨티뉴에이션) - 유보지점에서의 코루틴 상태다. 개념적으로는 중지점 이후의 실행을 나타낸다.
```kotlin
sequence {
    for (i in 1..10) yield(i * i)
    println("over")
}  
```

위의 예에서 yield를 호출할 때마다 실행을 유보한다. 이러한 개념으로는 남은 실행을 컨티뉴에이션으로 표현하므로 10개의 컨티뉴에이션을 갖게 된다. 첫번째 루프에서 i = 2인 곳에서 유보하고 두 번째 실행에서는 i = 3일 때 유보한다. 마지막에 이르러 "over"를 출력하면 컨티뉴에이션은 완료된다. 코루틴이 생성되면 시작되기 전에 컨티뉴에이션을 초기화하며 이때 타입은 Continuation<Unit>이 된다. 이 컨티뉴에이션이 전체 실행을 구성하게 된다.

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

await()같은 일반적인 유보 구현은 다음과 같다.

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

`suspendCoroutine`이 코루틴 내에서 호출될 때 컨티뉴에이션의 인스턴스가 코루틴의 상태를 캡쳐하고 지정된 블록에 인자로 전달된다. 코루틴을 다시 실행하려면 블록은 해당 쓰레드나 다른 쓰레드에서 `continuation.resumeWith()`를 직접 호출하거나 `continuation.resume()` 또는 `continuation.resumeWithException()`을 호출해야 한다.

코루틴의 유보 상태는 `resumeWith`를 호출하지 않고 suspendCoroutine블록이 반환될 때 발생한다. 블록 내부에서 복귀하기 전에 continuation의 resume이 발생했다면 코루틴은 일시 중지되지 않은 것으로 간주하고 계속 실행된다.

`continuation.resumeWith()`에 전달된 결과는 suspendCoroutine호출의 결과가 되며, 이는 `.await()`의 결과가 된다.

같은 continuation은 resume을 두번 호출할 수 없으면 그럴 경우 `IllegalStateException`을 발생시킨다.

### 코루틴 빌더

suspending 함수는 보통 함수에서는 호출할 수 없으므로 표준 라이브러리에서는 suspending 스코프가 아닌 곳에서 코루틴을 실행할 수 있는 시작함수를 제공한다. 다음은 간단한 `launch{}` 코루틴 빌더다.

```kotlin
fun launch(context: CoroutineContext = EmptyCoroutineContext, block: suspend () -> Unit) =
    block.startCoroutine(Continuation(context) { result ->
        result.onFailure { exception ->
            val currentThread = Thread.currentThread()
            currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
        }
    })
```

위 구현은 `Continuation(context){..}`함수에게 `resumeWith`함수의 몸체와 컨텍스트를 제공하는 단축 표현이다. 이 컨티뉴에이션은 completion continuation으로 `block.startCoroutine(...)`에게 전달된다.

코루틴의 완료는 completion continuation을 실행한다. resumeWith함수는 코루틴이 성공이나 실패로 완료될 때 실행된다. launch는 실행 후 잊어버리는 코루틴이므로, Unit을 반환 타입으로 하는 suspending함수를 정의하고 실제로 resu,e함수의 결과를 무시한다. 코루틴이 예외로 완료된다면 현재 쓰레드의 예외 핸들러가 보고하게 된다.

startCoroutine은 표준라이브러리에 확장함수로 정의되어있다.

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
fun <R, T> (suspend  R.() -> T).startCoroutine(receiver: R, completion: Continuation<T>)
```

`startCoroutine`은 코루틴을 만들고 현재 쓰레드에서 즉시 실행을 시작하여 첫 번째 유보지점까지 간 뒤 반환한다. 일시중지점은 코루틴의 몸체의 suspending함수를 발동시키고, 코루틴이 다시 실행될 지는 해당 suspending함수에 달려있다.

### 코루틴 컨텍스트

코루틴 컨텍스트는 코루틴을 추가할 수 있는 사용자 정의 객체들의 영속화된 세트다. 코루틴의 스레드 정책, 로깅, 보안, 코루틴실행의 트랜젝션측면, 코루틴식별자 및 이름 등을 담당하는 객체가 포함될 수 있다. 코루틴과 코루틴 컨텍스트에 대한 간단한 비유를 들어보자.
코루틴을 가벼운 쓰레드라 생각해보자. 이 때 코루틴 컨텍스트는 쓰레드 지역변수의 컬렉션 같은 것이다. 쓰레드 지역변수가 가변적인데 비해 코루틴 컨텍스트는 불변적이라는 것만 다르다. 코루틴은 굉장히 가볍기 때문에 변경이 필요한 경우 새 코루틴을 쉽게 시작할 수 있으므로 불변성은 코루틴에 대한 큰 제약은 되지 않는다.

표준 라이브러리에는 컨텍스트 요소의 구상 객체가 포함되어 있지 않다. 하지만 인터페이스와 추상클래스를 통해 합성으로 라이브러리에 정의할 수 있고, 이를 통해 같은 컨텍스트의 요소들이 평화롭게 공존할 수 있다.

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

`CoroutineContext` 자신은 네 가지 핵심 연산을 제공한다.

* get연산자는 키를 인자로 받아 요소를 반환한다.
* fold함수는 컨텍스트 내의 모든 요소를 반복할 수단을 제공한다.
* plus연산자는 Set의 plus와 비슷하게 작동하며 왼쪽요소와 오른쪽 요소를 결합한 요소를 반환하며 이 때 키는 왼쪽요소의 키가 된다.
* minus연산자는 그 키를 포함하지 않게 컨텍스트를 반환한다.

코루틴 컨텍스트에 있는 `Element`는 컨텍스트 자체다. 이 요소만 있는 싱글톤 컨텍스트인 셈이다.  이 개념이 합성 컨텍스트를 생성할 수 있게 하는데 코루틴 컨텍스트 요소의 라이브러리 정의를 가져와 그들을 +로 결합하면 된다. 예를 들어 하나의 라이브러리가 유저의 승인 정보를 갖는 `auth`요소를 정의하고 다른 라이브러리에서는 몇몇 실행 컨텍스트 정보를 갖는 `threadPool` 객체를 정의했다면 `launch{}` 코루틴 빌더를 합성 컨텍스트인 `launch(auth + threadPool){...}` 로 사용할 수 있다.

> 참고 : kotlinx.coroutines는 코루틴 실행을 백그라운드 스레드의 공유 풀로 디스패치하는 `Dispatchers.Default` 객체를 포함하여 여러 컨텍스트 요소를 제공한다.

표준 라이브러리는 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html" target="_blank">EmptyCoroutineContext</a>를 제공하는데 이는 아무 요소도 포함하지 않는 `CoroutineContext`의 인스턴스다.

모든 서드파티 컨텍스트 요소는 표준 라이브러리의 kotlin.coroutines 패키지에서 제공하는 `AbstractCoroutineContextElement` 클래스를 확장해야 한다. 라이브러리에 정의된 컨텍스트 요소는 다음 스타일을 권장한다. 다음 예는 현재 사용자 이름을 저장하는 가상의 인증 컨텍스트 요소다.

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

이 예제는 <a href="https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/auth.kt" target="_blank">여기</a>서 찾을 수 있다.

요소 클래스의 companion object를 컨텍스트의 `Key`로 정의하여 컨텍스트의 엘리먼트를 유연하게 다룰 수 있게 합니다. 다음은 현재 사용자의 이름을 확인해야하는 suspending 함수의 가상 구현이다.

```kotlin
suspend fun doSomething() {
    val currentUser = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
    // do something user-specific
}
```

suspending함수에서 사용할 수 있는  kotlin.coroutines 패키지의 최상위 수준 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html" target="_blank">coroutineContext</a> 속성을 사용해 현재 코루틴의 컨텍스트를 검색한다.

### Continuation interceptor(컨티뉴에이션 인터셉터)

비동기 UI 예를 생각해보자. 비동기 UI 애플리케이션은 다양한 suspend 함수가 임의의 스레드에서 코루틴 실행이 재개되더라도, 코루틴 본체가 항상 UI스레드에서 실행되도록 해야한다. 이는 continuation interceptor를 사용해 실현한다. 그 전에 먼저 코루틴의 수명주기를 완전히 이해해야 한다. `launch{}` 코루틴 빌더 사용 예를 보자.

```kotlin
launch(Swing) {
    initialCode() // 초기화코드 실행
    f1.await() // 유보지점 #1
    block1() // 실행 #1
    f2.await() // 유보지점 #2
    block2() // 실행 #2
}
```
코루틴은 첫 번째 유보지점까지 `initialCode`을 실행하며 시작된다. 유보지점에서 유보되고 해당 suspend에 정의된 대로 일정 시간이 지나면 `resumes`를 통해 다시 시작되어 `block1`을 실행하고 다시 유보지점에서 유보되며, 이 후 다시 시작되어 `block2`을 실행 후 완료된다.

컨티뉴에이션 인터셉터는 차단하는 옵션이 있으며, 각 분리된 유보지점에서 재게되는 `initialCode`, `block1`, `block2`의 실행에 대응하는 컨티뉴에이션을 감싼다. 
코루틴의 초기화 코드는 최초 초기화된 컨티뉴에이션의 resume인 셈이다. 표준 라이브러리는 `kotlin.coroutines` 패키지에 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html" target="_blank">`ContinuationInterceptor`</a> 인터페이스를 제공한다.

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    fun releaseInterceptedContinuation(continuation: Continuation<*>)
}
```

`interceptContinuation` 함수는 코루틴의 컨티뉴에이션을 감싼다. 코루틴 프레임웍은 코루틴이 유보될 때마다 각각의 이어진 resume을 위해 실제 `continuation`을 감싼다.

```kotlin
val intercepted = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```

코루틴 프레임웍은 개별 컨티뉴에이션의 실제 인스턴스의 가로챈 컨티뉴에이션의 결과를 캐쉬하고 더 이상 필요하지 않은 경우 `releaseInterceptedContinuation (intercepted)`을 호출한다. 

>참고로 `await` 같은 suspend함수의 경우 실제로는 코루틴의 suspend를 실행하지 않을 수도 있다. suspend함수 섹션에서 보여준 future의 `await`구현은 이미 완료된 경우 실제로는 코루틴을 suspend하지 않는다(이 경우, 즉시 `resume`을 호출하고 실제 suspend없이 실행이 계속됨) 컨티뉴에이션은 오직 `suspendCoroutine`블록이 `resume`호출 없이 반환되는 코루틴이 실행되는 동안에만 실제 suspend가 발생되는데 이 때만 차단된다.

Swing 인터셉터의 구현을 통해 Swing UI의 이벤트 디스패치 스레드에서 실행하도록 디스패치하는 구체적인 예를 살펴보자. `SwingUtilities.invokeLater`를 사용하여 Swing 이벤트 디스패치 스레드에 컨티뉴에이션을 디스패치하는 `SwingContinuation` 래퍼 클래스의 정의다.

```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> {
    override val context: CoroutineContext = cont.context
    
    override fun resumeWith(result: Result<T>) {
        SwingUtilities.invokeLater { cont.resumeWith(result) }
    }
}
```

이제 해당 컨텍스트 요소로 사용할 `Swing`객체를 정의하고 `ContinuationInterceptor` 인터페이스를 구현한다.

```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

>위 코드는 <a href="https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/swing.kt" target="_blank">여기</a>에서 있다. 참고 : <a href="https://github.com/kotlin/kotlinx.coroutines" target="_blank">kotlinx.coroutines</a>에서 `Swing` 객체의 실제 구현은 현재 실행 중인 코루틴의 식별자를 제공하고 표시하는 코루틴 디버깅 기능도 지원한다.

이제 `Swing` 인자와 함께 `launch{}` 코루틴 빌더를 사용해 Swing 이벤트 디스패치 스레드에서 완전히 실행되는 코루틴을 실행할 수 있다.

```kotlin
launch(Swing) {
  // code in here can suspend, but will always resume in Swing EDT
}
```

><a href="https://github.com/kotlin/kotlinx.coroutines" target="_blank">kotlinx.coroutines</a>에서 Swing 컨텍스트의 실제 구현은 라이브러리의 시간 및 디버깅 기능과 통합되므로 더 복잡하다.

### Restricted suspension(제한된 유보)

<a href="https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#generators" target="_blank">generators</a> 예에서 `sequence{}` 및 `yield()`를 구현하려면 다른 종류의 코루틴 빌더 및 suspend함수가 필요하다. `sequence{}` 코루틴 빌더의 예제 코드는 다음과 같다.

```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

위에서 최초 suspend블록을 실행시키던 `startCoroutine`과 유사한 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/create-coroutine.html" target="_blank">createCoroutine</a>을 사용하고 있다. 이 함수는 코루틴을 생성하지만 시작하지 않는다. 대신 `Continuation<Unit>`으로 초기화 된 컨티뉴에이션을 반환한다.

```kotlin
fun <T> (suspend () -> T).createCoroutine(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutine(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

그 외에도 인자로 받은 suspend람다가 `SequenceScope<T>`를 수신자로 하는 <a href="https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver" target="_blank">확장람다</a>라는 점이다. `SequenceScope` 인터페이스는 제네레이터 코드를 작성할 block의 스코프를 제공하며 다음과 같이 정의된다.

```kotlin
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

>역주 : 여기서 말하는 스코프란 block에 올 람다에서 추가적인 기능인 yield를 쓸 수 있게 해주는 일종의 영역이란 뜻입니다.

`sequence{}`함수 코드를 보면 `SequenceCoroutine<T>` 클래스의 인스턴스를 만든 뒤 한 개의 인스턴스로 `createCoroutine`의 `receiver`인자와 `complete` 인자 양쪽에 사용한다. 이를 위해 `SequenceCoroutine<T>`클래스는 `SequenceScope<T>`와 `Continuation<Unit>`을 동시에 구상한다(이터레이터를 위해 AbstractIterator도 같이..)

```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceScope<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // AbstractIterator 구상
    override fun computeNext() { nextStep.resume(Unit) }

    // Completion continuation 구상
    override val context: CoroutineContext get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Unit>) {
        result.getOrThrow() // 에러처리
        done()
    }

    // Generator 구상
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```


> 위 코드는 <a href="https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequence.kt" target="_blank">여기</a>에서 얻을 수 있다.
  표준 라이브러리의 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/sequence.html" target="_blank">`sequence`</a>는 out-of-the-box 최적화를 구현한다. 
  또한 <a href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence-scope/yield-all.html">`yieldAll`</a>도 추가로 제공한다.
  
> 실제 `sequence` 코드는 `BuilderInference`실험실 기능을 사용하여 제네레이터 섹션에 있는 `fibonacci`를 구현하는데 이를 이용하면 `T`를 명시적으로 선언하지 않아도 된다. 대신 `yield`가 호출될 때 넘겨진 타입으로 추론한다.

`yield` `suspendCoroutine`을 사용해 코루틴을 중단시키고 그 시점의 컨티뉴에이션을 캡쳐한다. 컨티뉴에이션은 `computeNext`가 실행될 때 `nextStep`에 저장되고 이후 재게 시 사용된다.
 
하지만 `sequence{}`와 `yield()`는 block에서 임의의 suspend함수에 대응하지 않는다. 오직 `yield()`만 특별하게 컨티뉴에이션을 캡쳐하는 것이다. 따라서 동기적으로만 작동한다.
`sequence{}`도 받아들인 block내의 코드에서 컨티뉴에이션이 어떤 방식으로 캡쳐되고, 어디에 저장되며, 언제 재게될 지에 대한 절대적인 제어가 필요하다. 이런 개념은 restricted suspension scope(제한된 유보영역)로 구현된다.
restricted suspension(제한된 유보)는 스코프를 정의하는 클래스나 인터페이스에 `@RestrictsSuspension` 애노태이션으로 정의할 수 있고 당연히 위에 등장한 `SequenceScope`에도 적용할 수 있다.

```kotlin
@RestrictsSuspension
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

이 애노태이션은 `sequence{}`가 인자로 받은 block 내부를 비롯해 유사한 동기적 코루틴 빌더들의 block 내부에서 등장하는 suspend함수에 특정한 제한을 가할 수 있다.
`@RestrictsSuspension`가 붙은 클래스나 인터페이스의 인스턴스를 수신자로 하는 suspend함수들은 restricted suspending function(제한된 유보함수)라 부른다.
이들은 오직 수신자로 온 스코프객체에 정의된 suspend함수만 부를 수 있다. 즉 `fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit)`의 block 내부에서는 오직 `SequenceScope<T>`에서 제공한 메소드인 `yield()`만 suspend함수로 부를 수 있다는 것이다.

반대로 `SequenceScope`로 확장되지 않은 람다는 내부에서 `suspendContinuation`를 비롯해서 어떤 suspend함수라도 호출할 수 있다는 뜻이다. `sequence`코루틴의 유보를 실행하려면 반드시 `SequenceScope.yield`를 호출해야 하지만 `yield`는 `SequenceScope`의 메소드로서 구현 시 어떤 제약도 없다. 즉 `SequenceScope`확장한 람다가 제약을 받는 것이지 `yield`의 구현 코드가 제약되는 것은 아니다.

`sequence`같은 제한된 코루틴 빌더은 임의의 컨텍스트를 지원하는 것은 큰 의미가 없다. 왜냐면 컨텍스트로서 제공되는 `SequenceScope`같은 스코프용 클래스나 인터페이스는 제약된 코루틴이고 이들은 항상 `EmptyCoroutineContext`을 컨텍스트로 사용해야 하고 `SequenceCoroutine`도 `context` getter가 `EmptyCoroutineContext`를 반환하게 작성된다. 만약 제한된 코루틴이 `EmptyCoroutinesContext`외의 컨텍스트를 생성하려하면 `IllegalArgumentException`이 발생한다.

## 구현 상세

이 장에서는 코루틴 구현에 대한 세부사항을 간략히 살펴봅니다. 내부 클래스나 코드 생성 전략은 뵈우 API나 ABI를 유지하는 한 언제라도 바뀔 수 있습니다.

### Continuation passing style(컨티뉴에이션 패싱 스타일)

유보함수는 CPS를 구현합니다. 모든 유보함수와 유보람다는 추가적인 `Continuation`인자를 갖고 있으며 호출시 암묵적으로 전달받습니다. 앞 서 나왔던 `await` 유보함수의 정의를 다음과 같습니다.

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

그러나 CPS변환이 이뤄진 실제 시그니쳐는 다음과 같습니다.

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

원래 반환타입 `T`는 추가된 컨티뉴에이션 인자의 타입인자의 위치로 옮겨졌습니다.
변환 후의 반환타입 `Any?`는 유보함수의 동작을 나타낼 수 있게 설계되었습니다.
유보함수가 코루틴을 유보했을 때 `COROUTINE_SUSPENDED`라는 특별한 값을 반환합니다.
반대로 코루틴을 유보하지 않고 코루틴 실행을 계속하는 경우 결과를 반환하거나 직접 예외를 던집니다.
`await`함수가 구현한 반환타입 `Any?`는 실제로는 `COROUTINE_SUSPENDED`나 `T` 양쪽 모두를 합친 것으로 코틀린의 타입시스템에서는 지원되기 때문에 사용되었습니다.

유보함수의 실제 구현은 스택프레임에서 직접 컨티뉴에이션을 호출할 수 없는데 이는 장시간 실행되는 코루틴이 스택오버플로우를 유발하기 때문입니다.
표준 라이브라리에 들어있는 `suspendCoroutine`함수는 컨티뉴에이션의 호출을 추적하여 이러한 복잡성을 감추고 컨티뉴에이션의 호출시기나 방법과 상관없이 유보함수의 실제 구현 사항이 준수되도록 강제합니다.

### 상태머신

코루틴을 효과적으로 구현하는 것은 중요하므로 되도록이면 적은 수의 클래스와 객체를 생성해야 합니다. 많은 언어는 상태머신을 통해 이를 구현하며 코틀린도 마찬가지입니다. 코틀린의 경우 컴파일러가 이런 상태머신을 여러 개의 유보지점을 갖는 유보람다당 단 하나의 클래스를 생성합니다.

주 아이디어: 유보함수는 상태머신으로 컴파일되며 여기서 상태란 유보지점에 해당합니다. 
예: 두 개의 유보지접을 갖는 유보블록을 살펴봅니다.
 
```kotlin
val a = a()
val y = foo(a).await() // 유보지점 #1
b()
val z = bar(a, y).await() // 유보지점 #2
c(z)
``` 

위 코드에는 세 가지 상태가 존재합니다.
 
 * 초기화지점 (어떤 유보지점보다 앞임)
 * 첫 번째 유보지점 이후
 * 두 번째 유보지점 이후
 
각 상태는 이 블록 내 컨티뉴에이션의 각 진입점이 됩니다.
(초기화 컨티뉴에이션은 코드 젤 처음에서 진행됩니다)

이 코드는 익면 클래스로 컴파일되며 상태머신을 구현한 메소드와 상태머신의 현재 상태를 잡아둘 필드와 각 상태 사이에 공유한 코루틴의 지역변수를 위한 필드를 갖게 됩니다(코루틴의 클로저를 위한 필드 또한 있을 수 있습니다만 이 예에서는 없습니다) 위의 코드에 대해 `await`유보함수의 호출을 위해 CPS를 사용한 의사 자바코드입니다.

  
``` java
class <anonymous_for_state_machine> extends SuspendLambda<...> {
    // 상태머신의 현재 상태
    int label = 0
    
    // 코루틴의 지역 변수
    A a = null
    Y y = null
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        // 이 호출에서 결과는 `null`
        a = a()
        label = 1
        result = foo(a).await(this) // 'this'가 컨티뉴에이션으로 전달됨
        if (result == COROUTINE_SUSPENDED) return // await가 유보실행된 경우 반환됨
      L1:
        // 외부코드가 .await() 의 결과를 전달하는 이 컨티뉴에이션을 재게함 
        y = (Y) result
        b()
        label = 2
        result = bar(a, y).await(this) // 'this'가 컨티뉴에이션으로 전달됨
        if (result == COROUTINE_SUSPENDED) return // await가 유보실행된 경우 반환됨
      L2:
        // 외부코드가 .await() 의 결과를 전달하는 이 컨티뉴에이션을 재게함 
        Z z = (Z) result
        c(z)
        label = -1 // 더 이상의 단계를 허용하지 않음
        return
    }          
}    
```  

Note that there is a `goto` operator and labels because the example depicts what happens in the 
byte code, not in the source code.

Now, when the coroutine is started, we call its `resumeWith()` — `label` is `0`, 
and we jump to `L0`, then we do some work, set the `label` to the next state — `1`, call `.await()`
and return if the execution of the coroutine was suspended. 
When we want to continue the execution, we call `resumeWith()` again, and now it proceeds right to 
`L1`, does some work, sets the state to `2`, calls `.await()` and again returns in case of suspension.
Next time it continues from `L3` setting the state to `-1` which means 
"over, no more work to do". 

A suspension point inside a loop generates only one state, 
because loops also work through (conditional) `goto`:
 
```kotlin
var x = 0
while (x < 10) {
    x += nextNumber().await()
}
```

is generated as

``` java
class <anonymous_for_state_machine> extends SuspendLambda<...> {
    // The current state of the state machine
    int label = 0
    
    // local variables of the coroutine
    int x
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        
      L0:
        x = 0
      LOOP:
        if (x >= 10) goto END
        label = 1
        result = nextNumber().await(this) // 'this' is passed as a continuation 
        if (result == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L1:
        // external code has resumed this coroutine passing the result of .await()
        x += ((Integer) result).intValue()
        label = -1
        goto LOOP
      END:
        label = -1 // No more steps are allowed
        return 
    }          
}    
```  

### Compiling suspending functions

The compiled code for suspending function depends on how and when it invokes other suspending functions.
In the simplest case, a suspending function invokes other suspending functions only at _tail positions_ 
making _tail calls_ to them. This is a typical case for suspending functions that implement low-level synchronization 
primitives or wrap callbacks, as shown in [suspending functions](#suspending-functions) and
[wrapping callbacks](#wrapping-callbacks) sections. These functions invoke some other suspending function
like `suspendCoroutine` at tail position. They are compiled just like regular non-suspending functions, with 
the only exception that the implicit continuation parameter they've got from [CPS transformation](#continuation-passing-style)
is passed to the next suspending function in tail call.

In a case when suspending invocations appear in non-tail positions, the compiler creates a 
[state machine](#state-machines) for the corresponding suspending function. An instance of the state machine
object in created when suspending function is invoked and is discarded when it completes.

> Note: in the future versions this compilation strategy may be optimized to create an instance of a state machine 
only at the first suspension point.

This state machine object, in turn, serves as the _completion continuation_ for the invocation of other
suspending functions in non-tail positions. This state machine object instance is updated and reused when 
the function makes multiple invocations to other suspending functions. 
Compare this to other [asynchronous programming styles](#asynchronous-programming-styles),
where each subsequent step of asynchronous processing is typically implemented with a separate, freshly allocated,
closure object.

### 코루틴 내장함수

코틀린 표준라이브러리는 `kotlin.coroutines.intrinsics` 패키지를 제공합니다.
이 패키지에는 코루틴 시스템 내부의 상세 구현을 외부에 노출하는 몇 가지 정의를 내포하고 있습니다.
이번 섹션에서는 이를 설명하며 주의 깊게 사용해야 합니다.

패키지가 제공하는 이 정의들은 일반적인 코드에는 사용하면 안됩니다. 따라서 IDE의 자동완성에는 `kotlin.coroutines.intrinsics` 패키지가 숨겨져 있습니다. 이 정의들을 사용하려면 import문을 직접 추가해야 합니다.

```kotlin
import kotlin.coroutines.intrinsics.*
```

표준 라이브러리 상의 `suspendCoroutine` 유보함수의 실제 구현은 코틀린으로 작성되었으며 소스코드는 표준 라이브러리 소스패키지의 일부로 제공됩니다. 코루틴의 안전한 사용을 위해 상태머신의 실제 컨티뉴에이션은 각 코루틴의 유보지점마다 추가 객체로 감싸집니다. 이는 비동기 계산 및 future같은 진짜 비동기 사용시 완벽하게 어울립니다. 왜냐면 추가 할당된 객체의 비용보다 해당되는 비동기용 객체의 비용이 런타임에 훨씬 크기 때문입니다. 하지만 제네레이터의 경우는 추가 비용이 엄청 크기 때문에 intrinsics 패키지는 저수준의 성능에 민감한 기본요소를 제공합니다.

표준 라이브러리의 `kotlin.coroutines.intrinsics`패키지는 <a target="_blank" href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/suspend-coroutine-unintercepted-or-return.html">`suspendCoroutineUninterceptedOrReturn`</a> 를 제공하는데 시그니처는 아래와 같습니다.

```kotlin
suspend fun <T> suspendCoroutineUninterceptedOrReturn(block: (Continuation<T>) -> Any?): T
```

이 함수는 유보함수의 CPS에 직접적인 제어를 제공하고 컨티뉴에이션의 _unintercepted_ 를 노출한다. 이는 `Continuation.resumeWith`를 호출 시 ContinuationInterceptor를 경유하지 않게 할 수 있다는 뜻이다. 이는 제한된 유보(restricted suspension)로 동작하는 동기적 코루틴을 작성할 때 사용될 수 있다. 제한된 유보로 지정된 코루틴은 빈 컨텍스트를 사용하므로 컨티뉴에이션 인터셉터가 정의될 수 없다. 뿐만 아니라 현재 실행중인 쓰레드가 이미 원하는 컨텍스트에 있다고 알려진 경우에도 사용될 수 있습니다.
그렇지 않으면 `kotlin.coroutines.intrinsics`패키지에서 확장된 함수로 인터셉트된 컨티뉴에이션을 얻어야 합니다.

```kotlin
fun <T> Continuation<T>.intercepted(): Continuation<T>
```

`Continuation.resumeWith` 는 인터셉트된 컨티뉴에이션을 호출하게 될 것입니다.

이제 `suspendCoroutineUninterceptedOrReturn`함수에 전달된 `block`은 코루틴이 유보될 때(이 경우 `Continuation.resumeWith`는 한 번만 호출됨) <a target="_blank" href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/-c-o-r-o-u-t-i-n-e_-s-u-s-p-e-n-d-e-d.html">`COROUTINE_SUSPENDED`</a>를 반환할 수 있으며 `T`를 반환하거나 예외를 던질 수 있습니다.

이 규칙을 따르지 않으면 `suspendCoroutineUninterceptedOrReturn`를 사용시 버그를 찾거나 테스트를 재현하기가 매우 어렵습니다. 이 규칙은 보통 `buildSequence`/`yield`같은 코루틴보다는 따르지 쉽습니다만, `suspendCoroutineUninterceptedOrReturn`를 이용해 `await`같은 비동기 유보함수를 작성할 시에는 `suspendCoroutine`의 도움없이 바르게 구현하기게 극단적으로 어렵기 때문에 권장하지 않습니다.

`kotlin.coroutines.intrinsics`패키지에는 <a target="_blank" href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/create-coroutine-unintercepted.html">`createCoroutineUnintercepted`</a> 라는 함수도 있는 시그니처는 다음과 같습니다.

```kotlin
fun <T> (suspend () -> T).createCoroutineUnintercepted(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

`createCoroutine`와 유사하게 작동하지만 초기화 컨티뉴에이션이 인터셉터 없이 반환됩니다.
`suspendCoroutineUninterceptedOrReturn`비슷하게 동기적 코루틴은 더 좋은 성능을 낼 수 있습니다.
예를들어 `sequence{}`빌더의 최적화버전은 `createCoroutineUnintercepted`를 아래와 같이 사용합니다.

```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutineUnintercepted(receiver = this, completion = this)
    }
}
```

`yield`도 `suspendCoroutineUninterceptedOrReturn`를 이용해 아래와 같이 최적화합니다.
단 `yield`는 항상 유보하므로 대응하는 블록은 항상 `COROUTINE_SUSPENDED`를 반환합니다.

```kotlin
// 제네레이터 구현
override suspend fun yield(value: T) {
    setNext(value)
    return suspendCoroutineUninterceptedOrReturn { cont ->
        nextStep = cont
        COROUTINE_SUSPENDED
    }
}
```

> <a target="_blank" href="https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/optimized/sequenceOptimized.kt">여기</a>서 전체 코드를 얻을 수 있습니다.

저수준의 `startCoroutine`에 대한 두 가지 구현체도 <a target="_blank" href="http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/start-coroutine-unintercepted-or-return.html">`startCoroutineUninterceptedOrReturn`</a>라는 이름으로 제공됩니다.

```kotlin
fun <T> (suspend () -> T).startCoroutineUninterceptedOrReturn(completion: Continuation<T>): Any?
fun <R, T> (suspend R.() -> T).startCoroutineUninterceptedOrReturn(receiver: R, completion: Continuation<T>): Any?
```

`startCoroutine`와 두 가지 차이점이 있는데 우선 `ContinuationInterceptor`가 코루틴을 시작할 때 자동으로 시작하지 않아 호출자는 필요시 적절한 컨텍스트의 실행을 보장해야 합니다.
두 번째로 코루틴이 유보하지 않지만 값을 반환하거나 예외를 던진다면 `startCoroutineUninterceptedOrReturn`의 호출이 이를 처리한다는 것입니다. 코루틴이 유보된다면 `COROUTINE_SUSPENDED`이 반환됩니다.

`startCoroutineUninterceptedOrReturn`의 가장 큰 용도는 `suspendCoroutineUninterceptedOrReturn`와 결합하여 같은 컨텍스트지만 다름 블록의 코드를 유보된 코루틴을 계속 실행하는 것입니다.


```kotlin 
suspend fun doSomething() = suspendCoroutineUninterceptedOrReturn { cont ->
    // 실행이 필요한 코드의 블록을 계산하거나 생성
    startCoroutineUninterceptedOrReturn(completion = block) // suspendCoroutineUninterceptedOrReturn 의 결과를 
}
```

## Appendix

This is a non-normative section that does not introduce any new language constructs or 
library functions, but covers some additional topics dealing with resource management, concurrency, 
and programming style, as well as provides more examples for a large variety of use-cases.

### Resource management and GC

Coroutines don't use any off-heap storage and do not consume any native resources by themselves, unless the code
that is running inside a coroutine does open a file or some other resource. While files opened in a coroutine must
be closed somehow, the coroutine itself does not need to be closed. When coroutine is suspended its whole state is 
available by the reference to its continuation. If you lose the reference to suspended coroutine's continuation,
then it will be ultimately collected by garbage collector.

Coroutines that open some closeable resources deserve a special attention. Consider the following coroutine
that uses the `sequence{}` builder from [restricted suspension](#restricted-suspension) section to produce
a sequence of lines from a file:

```kotlin
fun sequenceOfLines(fileName: String) = sequence<String> {
    BufferedReader(FileReader(fileName)).use {
        while (true) {
            yield(it.readLine() ?: break)
        }
    }
}
```

This function returns a `Sequence<String>` and you can use this function to print all lines from a file 
in a natural way:
 
```kotlin
sequenceOfLines("https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt")
    .forEach(::println)
```

> You can get full code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt)

It works as expected as long as you iterate the sequence returned by the `sequenceOfLines` function
completely. However, if you print just a few first lines from this file like here:

```kotlin
sequenceOfLines("https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt")
        .take(3)
        .forEach(::println)
```

then the coroutine resumes a few times to yield the first three lines and becomes _abandoned_.
It is Ok for the coroutine itself to be abandoned but not for the open file. The 
[`use` function](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 
will not have a chance to finish its execution and close the file. The file will be left open
until collected by GC, because Java files have a `finalizer` that closes the file. It is 
not a big problem for a small slide-ware or a short-running utility, but it may be a disaster for
a large backend system with multi-gigabyte heap, that can run out of open file handles 
faster than it runs out of memory to trigger GC.

This is a similar gotcha to Java's 
[`Files.lines`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#lines-java.nio.file.Path-)
method that produces a lazy stream of lines. It returns a closeable Java stream, but most stream operations do not
automatically invoke the corresponding 
`Stream.close` method and it is up to the user to remember about the need to close the corresponding stream. 
One can define closeable sequence generators 
in Kotlin, but they will suffer from a similar problem that no automatic mechanism in the language can
ensure that they are closed after use. It is explicitly out of the scope of Kotlin coroutines
to introduce a language mechanism for an automated resource management.

However, usually this problem does not affect asynchronous use-cases of coroutines. An asynchronous coroutine 
is never abandoned, but ultimately runs until its completion, so if the code inside a coroutine properly closes
its resources, then they will be ultimately closed.

### Concurrency and threads

Each individual coroutine, just like a thread, is executed sequentially. It means that the following kind 
of code is perfectly safe inside a coroutine:

```kotlin
launch { // starts a coroutine
    val m = mutableMapOf<String, String>()
    val v1 = someAsyncTask1() // start some async task
    val v2 = someAsyncTask2() // start some async task
    m["k1"] = v1.await() // map modification waiting on await
    m["k2"] = v2.await() // map modification waiting on await
}
```

You can use all the regular single-threaded mutable structures inside the scope of a particular coroutine.
However, sharing mutable state _between_ coroutines is potentially dangerous. If you use a coroutine builder
that installs a dispatcher to resume all coroutines JS-style in the single event-dispatch thread, 
like the `Swing` interceptor shown in [continuation interceptor](#continuation-interceptor) section,
then you can safely work with all shared
objects that are generally modified from this event-dispatch thread. 
However, if you work in multi-threaded environment or otherwise share mutable state between
coroutines running in different threads, then you have to use thread-safe (concurrent) data structures. 

Coroutines are like threads in this sense, albeit they are more lightweight. You can have millions of coroutines running on 
just a few threads. The running coroutine is always executed in some thread. However, a _suspended_ coroutine
does not consume a thread and it is not bound to a thread in any way. The suspending function that resumes this
coroutine decides which thread the coroutine is resumed on by invoking `Continuation.resumeWith` on this thread 
and coroutine's interceptor can override this decision and dispatch the coroutine's execution onto a different thread.

## Asynchronous programming styles

There are different styles of asynchronous programming.
 
Callbacks were discussed in [asynchronous computations](#asynchronous-computations) section and are generally
the least convenient style that coroutines are designed to replace. Any callback-style API can be
wrapped into the corresponding suspending function as shown [here](#wrapping-callbacks). 

Let us recap. For example, assume that you start with a hypothetical _blocking_ `sendEmail` function 
with the following signature:

```kotlin
fun sendEmail(emailArgs: EmailArgs): EmailResult
```

It blocks execution thread for potentially long time while it operates.

To make it non-blocking you can use, for example, error-first 
[node.js callback convention](https://www.tutorialspoint.com/nodejs/nodejs_callbacks_concept.htm)
to represent its non-blocking version in callback-style with the following signature:

```kotlin
fun sendEmail(emailArgs: EmailArgs, callback: (Throwable?, EmailResult?) -> Unit)
```

However, coroutines enable other styles of asynchronous non-blocking programming. One of them
is async/await style that is built into many popular languages.
In Kotlin this style can be replicated by introducing `future{}` and `.await()` library functions
that were shown as a part of [futures](#futures) use-case section.
 
This style is signified by the convention to return some kind of future object from the function instead 
of taking a callback as a parameter. In this async-style the signature of `sendEmail` is going to look like this:

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult>
```

As a matter of style, it is a good practice to add `Async` suffix to such method names, because their 
parameters are no different from a blocking version and it is quite easy to make a mistake of forgetting about
asynchronous nature of their operation. The function `sendEmailAsync` starts a _concurrent_ asynchronous operation 
and potentially brings with it all the pitfalls of concurrency. However, languages that promote this style of 
programming also typically have some kind of `await` primitive to bring the execution back into the sequence as needed. 

Kotlin's _native_ programming style is based on suspending functions. In this style, the signature of 
`sendEmail` looks naturally, without any mangling to its parameters or return type but with an additional
`suspend` modifier:

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult
```

The async and suspending styles can be easily converted into one another using the primitives that we've 
already seen. For example, `sendEmailAsync` can be implemented via suspending `sendEmail` using
[`future` coroutine builder](#building-futures):

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult> = future {
    sendEmail(emailArgs)
}
```

while suspending function `sendEmail` can be implemented via `sendEmailAsync` using
[`.await()` suspending function](#suspending-functions)

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult = 
    sendEmailAsync(emailArgs).await()
```

So, in some sense, these two styles are equivalent and are both definitely superior to callback style in their
convenience. However, let us look deeper at a difference between `sendEmailAsync` and suspending `sendEmail`.

Let us compare how they **compose** first. Suspending functions can be composed just like normal functions:

```kotlin
suspend fun largerBusinessProcess() {
    // a lot of code here, then somewhere inside
    sendEmail(emailArgs)
    // something else goes on after that
}
```

The corresponding async-style functions compose in this way:

```kotlin
fun largerBusinessProcessAsync() = future {
   // a lot of code here, then somewhere inside
   sendEmailAsync(emailArgs).await()
   // something else goes on after that
}
```

Observe, that async-style function composition is more verbose and _error prone_. 
If you omit `.await()` invocation in async-style
example,  the code still compiles and works, but it now does email sending process 
asynchronously or even _concurrently_ with the rest of a larger business process, 
thus potentially modifying some shared state and introducing some very hard to reproduce errors.
On the contrary, suspending functions are _sequential by default_.
With suspending functions, whenever you need any concurrency, you explicitly express it in the source code with 
some kind of `future{}` or a similar coroutine builder invocation.

Compare how these styles **scale** for a big project using many libraries. Suspending functions are
a light-weight language concept in Kotlin. All suspending functions are fully usable in any unrestricted Kotlin coroutine.
Async-style functions are framework-dependent. Every promises/futures framework must define its own `async`-like 
function that returns its own kind of promise/future class and its own `await`-like function, too.

Compare their **performance**. Suspending functions provide minimal overhead per invocation. 
You can checkout [implementation details](#implementation-details) section.
Async-style functions need to keep quite heavy promise/future abstraction in addition to all of that suspending machinery. 
Some future-like object instance must be always returned from async-style function invocation and it cannot be optimized away even 
if the function is very short and simple. Async-style is not well-suited for very fine-grained decomposition.

Compare their **interoperability** with JVM/JS code. Async-style functions are more interoperable with JVM/JS code that 
uses a matching type of future-like abstraction. In Java or JS they are just functions that return a corresponding
future-like object. Suspending functions look strange from any language that does not support 
[continuation-passing-style](#continuation-passing-style) natively.
However, you can see in the examples above how easy it is to convert any suspending function into an 
async-style function for any given promise/future framework. So, you can write suspending function in Kotlin just once, 
and then adapt it for interoperability with any style of promise/future with one line of code using an appropriate 
`future{}` coroutine builder function.

### Wrapping callbacks

Many asynchronous APIs have callback-style interfaces. The `suspendCoroutine` suspending function 
from the standard library (see [suspending functions](#suspending-functions) section)
provides for an easy way to wrap any callback into a Kotlin suspending function. 

There is a simple pattern. Assume that you have `someLongComputation` function with callback that 
receives some `Value` that is a result of this computation.

```kotlin
fun someLongComputation(params: Params, callback: (Value) -> Unit)
```

You can convert it into a suspending function with the following straightforward code:
 
```kotlin
suspend fun someLongComputation(params: Params): Value = suspendCoroutine { cont ->
    someLongComputation(params) { cont.resume(it) }
} 
```

Now the return type of this computation is explicit, but it is still asynchronous and does not block a thread.

> Note that [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) contains a framework for 
  cooperative cancellation of coroutines. It provides `suspendCancellableCoroutine` function that is 
  similar to `suspendCoroutine`, but with cancellation support. See the 
  [section on cancellation](http://kotlinlang.org/docs/reference/coroutines/cancellation-and-timeouts.html) 
  in its guide for more details.

For a more complex example let us take a look at
`aRead()` function from [asynchronous computations](#asynchronous-computations) use case. 
It can be implemented as a suspending extension function for Java NIO 
[`AsynchronousFileChannel`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/AsynchronousFileChannel.html)
and its 
[`CompletionHandler`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/CompletionHandler.html)
callback interface with the following code:

```kotlin
suspend fun AsynchronousFileChannel.aRead(buf: ByteBuffer): Int =
    suspendCoroutine { cont ->
        read(buf, 0L, Unit, object : CompletionHandler<Int, Unit> {
            override fun completed(bytesRead: Int, attachment: Unit) {
                cont.resume(bytesRead)
            }

            override fun failed(exception: Throwable, attachment: Unit) {
                cont.resumeWithException(exception)
            }
        })
    }
```

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/io/io.kt).
  Note: the actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)
  supports cancellation to abort long-running IO operations.

If you are dealing with lots of functions that all share the same type of callback, then you can define a common
wrapper function to easily convert all of them to suspending functions. For example, 
[vert.x](http://vertx.io/) uses a particular convention that all its asynchronous functions receive 
`Handler<AsyncResult<T>>` as a callback. To simplify the use of arbitrary vert.x functions from coroutines,
the following helper function can be defined:

```kotlin
inline suspend fun <T> vx(crossinline callback: (Handler<AsyncResult<T>>) -> Unit) = 
    suspendCoroutine<T> { cont ->
        callback(Handler { result: AsyncResult<T> ->
            if (result.succeeded()) {
                cont.resume(result.result())
            } else {
                cont.resumeWithException(result.cause())
            }
        })
    }
```

Using this helper function, an arbitrary asynchronous vert.x function `async.foo(params, handler)`
can be invoked from a coroutine with `vx { async.foo(params, it) }`.

### Building futures

The `future{}` builder from [futures](#futures) use-case can be defined for any future or promise primitive
similarly to the `launch{}` builder as explained in [coroutine builders](#coroutine-builders) section:

```kotlin
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
        CompletableFutureCoroutine<T>(context).also { block.startCoroutine(completion = it) }
```

The first difference from `launch{}` is that it returns an implementation of
[`CompletableFuture`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), 
and the other difference is that it is defined with a default `CommonPool` context, so that its default
execution behavior is similar to the 
[`CompletableFuture.supplyAsync`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-)
method that by default runs its code in 
[`ForkJoinPool.commonPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--).
The basic implementation of `CompletableFutureCoroutine` is straightforward:

```kotlin
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
    override fun resumeWith(result: Result<T>) {
        result
            .onSuccess { complete(it) }
            .onFailure { completeExceptionally(it) }
    }
}
```

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/future/future.kt).
  The actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) is more advanced,
  because it propagates the cancellation of the resulting future to cancel the coroutine.

The completion of this coroutine invokes the corresponding `complete` methods of the future to record the
result of this coroutine.

### Non-blocking sleep

Coroutines should not use [`Thread.sleep`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-),
because it blocks a thread. However, it is quite straightforward to implement a suspending non-blocking `delay` function by using
Java's [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)

```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
    Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont ->
    executor.schedule({ cont.resume(Unit) }, time, unit)
}
```

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/delay/delay.kt).
  Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) also provides `delay` function.

Note, that this kind of `delay` function resumes the coroutines that are using it in its single "scheduler" thread.
The coroutines that are using [interceptor](#continuation-interceptor) like `Swing` will not stay to execute in this thread,
as their interceptor dispatches them into an appropriate thread. Coroutines without interceptor will stay to execute
in this scheduler thread. So this solution is convenient for demo purposes, but it is not the most efficient one. It
is advisable to implement sleep natively in the corresponding interceptors.

For `Swing` interceptor that native implementation of non-blocking sleep shall use
[Swing Timer](https://docs.oracle.com/javase/8/docs/api/javax/swing/Timer.html)
that is specifically designed for this purpose:

```kotlin
suspend fun Swing.delay(millis: Int): Unit = suspendCoroutine { cont ->
    Timer(millis) { cont.resume(Unit) }.apply {
        isRepeats = false
        start()
    }
}
```

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/swing-delay.kt).
  Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) implementation of `delay` is aware of
  interceptor-specific sleep facilities and automatically uses the above approach where appropriate. 

### Cooperative single-thread multitasking

It is very convenient to write cooperative single-threaded applications, because you don't have to 
deal with concurrency and shared mutable state. JS, Python and many other languages do 
not have threads, but have cooperative multitasking primitives.

[Coroutine interceptor](#coroutine-interceptor) provides a straightforward tool to ensure that
all coroutines are confined to a single thread. The example code
[here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/threadContext.kt) defines `newSingleThreadContext()` function that
creates a single-threaded execution services and adapts it to the coroutine interceptor
requirements.

We will use it with `future{}` coroutine builder that was defined in [building futures](#building-futures) section
in the following example that works in a single thread, despite the
fact that it has two asynchronous tasks inside that are both active.

```kotlin
fun main(args: Array<String>) {
    log("Starting MyEventThread")
    val context = newSingleThreadContext("MyEventThread")
    val f = future(context) {
        log("Hello, world!")
        val f1 = future(context) {
            log("f1 is sleeping")
            delay(1000) // sleep 1s
            log("f1 returns 1")
            1
        }
        val f2 = future(context) {
            log("f2 is sleeping")
            delay(1000) // sleep 1s
            log("f2 returns 2")
            2
        }
        log("I'll wait for both f1 and f2. It should take just a second!")
        val sum = f1.await() + f2.await()
        log("And the sum is $sum")
    }
    f.get()
    log("Terminated")
}
```

> You can get fully working example [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/threadContext-example.kt).
  Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) has ready-to-use implementation of
  `newSingleThreadContext`. 

If your whole application is based on a single-threaded execution, you can define your own helper coroutine
builders with a hard-coded context for your single-threaded execution facilities.
  
## Asynchronous sequences

The `sequence{}` coroutine builder that is shown in [restricted suspension](#restricted-suspension)
section is an example of a _synchronous_ coroutine. Its producer code in the coroutine is invoked
synchronously in the same thread as soon as its consumer invokes `Iterator.next()`. 
The `sequence{}` coroutine block is restricted and it cannot suspend its execution using 3rd-party suspending
functions like asynchronous file IO as shown in [wrapping callbacks](#wrapping-callbacks) section.

An _asynchronous_ sequence builder is allowed to arbitrarily suspend and resume its execution. It means
that its consumer shall be ready to handle the case, when the data is not produced yet. This is
a natural use-case for suspending functions. Let us define `SuspendingIterator` interface that is
similar to a regular 
[`Iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterator/) 
interface, but its `next()` and `hasNext()` functions are suspending:
 
```kotlin
interface SuspendingIterator<out T> {
    suspend operator fun hasNext(): Boolean
    suspend operator fun next(): T
}
```

The definition of `SuspendingSequence` is similar to the standard
[`Sequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence/index.html)
but it returns `SuspendingIterator`:

```kotlin
interface SuspendingSequence<out T> {
    operator fun iterator(): SuspendingIterator<T>
}
```

We also define a scope interface for that is similar to a scope of a synchronous sequence,
but it is not restricted in its suspensions:

```kotlin
interface SuspendingSequenceScope<in T> {
    suspend fun yield(value: T)
}
```

The builder function `suspendingSequence{}` is similar to a synchronous `sequence{}`.
Their differences lie in implementation details of `SuspendingIteratorCoroutine` and
in the fact that it makes sense to accept an optional context in this case:

```kotlin
fun <T> suspendingSequence(
    context: CoroutineContext = EmptyCoroutineContext,
    block: suspend SuspendingSequenceScope<T>.() -> Unit
): SuspendingSequence<T> = object : SuspendingSequence<T> {
    override fun iterator(): SuspendingIterator<T> = suspendingIterator(context, block)
}
```

> You can get full code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/suspendingSequence/suspendingSequence.kt).
  Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) has an implementation of
  `Channel` primitive with the corresponding `produce{}` coroutine builder that provides more 
  flexible implementation of the same concept.

Let us take `newSingleThreadContext{}` context from
[cooperative single-thread multitasking](#cooperative-single-thread-multitasking) section
and non-blocking `delay` function from [non-blocking sleep](#non-blocking-sleep) section.
This way we can write an implementation of a non-blocking sequence that yields
integers from 1 to 10, sleeping 500 ms between them:
 
```kotlin
val seq = suspendingSequence(context) {
    for (i in 1..10) {
        yield(i)
        delay(500L)
    }
}
```
   
Now the consumer coroutine can consume this sequence at its own pace, while also 
suspending with other arbitrary suspending functions. Note, that 
Kotlin [for loops](https://kotlinlang.org/docs/reference/control-flow.html#for-loops)
work by convention, so there is no need for a special `await for` loop construct in the language.
The regular `for` loop can be used to iterate over an asynchronous sequence that we've defined
above. It is suspended whenever producer does not have a value:


```kotlin
for (value in seq) { // suspend while waiting for producer
    // do something with value here, may suspend here, too
}
```

> You can find a worked out example with some logging that illustrates the execution
  [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/suspendingSequence/suspendingSequence-example.kt)

### Channels

Go-style type-safe channels can be implemented in Kotlin as a library. We can define an interface for 
send channel with suspending function `send`:

```kotlin
interface SendChannel<T> {
    suspend fun send(value: T)
    fun close()
}
```
  
and receiver channel with suspending function `receive` and an `operator iterator` in a similar style 
to [asynchronous sequences](#asynchronous-sequences):

```kotlin
interface ReceiveChannel<T> {
    suspend fun receive(): T
    suspend operator fun iterator(): ReceiveIterator<T>
}
```

The `Channel<T>` class implements both interfaces.
The `send` suspends when the channel buffer is full, while `receive` suspends when the buffer is empty.
It allows us to copy Go-style code into Kotlin almost verbatim.
The `fibonacci` function that sends `n` fibonacci numbers in to a channel from
[the 4th concurrency example of a tour of Go](https://tour.golang.org/concurrency/4)  would look 
like this in Kotlin:

```kotlin
suspend fun fibonacci(n: Int, c: SendChannel<Int>) {
    var x = 0
    var y = 1
    for (i in 0..n - 1) {
        c.send(x)
        val next = x + y
        x = y
        y = next
    }
    c.close()
}

```

We can also define Go-style `go {...}` block to start the new coroutine in some kind of
multi-threaded pool that dispatches an arbitrary number of light-weight coroutines onto a fixed number of 
actual heavy-weight threads.
The example implementation [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/go.kt) is trivially written on top of
Java's common [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html).

Using this `go` coroutine builder, the main function from the corresponding Go code would look like this,
where `mainBlocking` is shortcut helper function for `runBlocking` with the same pool as `go{}` uses:

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val c = Channel<Int>(2)
    go { fibonacci(10, c) }
    for (i in c) {
        println(i)
    }
}
```

> You can checkout working code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-4.kt)

You can freely play with the buffer size of the channel. 
For simplicity, only buffered channels are implemented in the example (with a minimal buffer size of 1), 
because unbuffered channels are conceptually similar to [asynchronous sequences](#asynchronous-sequences)
that were covered before.

Go-style `select` control block that suspends until one of the actions becomes available on 
one of the channels can be implemented as a Kotlin DSL, so that 
[the 5th concurrency example of a tour of Go](https://tour.golang.org/concurrency/5)  would look 
like this in Kotlin:
 
```kotlin
suspend fun fibonacci(c: SendChannel<Int>, quit: ReceiveChannel<Int>) {
    var x = 0
    var y = 1
    whileSelect {
        c.onSend(x) {
            val next = x + y
            x = y
            y = next
            true // continue while loop
        }
        quit.onReceive {
            println("quit")
            false // break while loop
        }
    }
}
```

> You can checkout working code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-5.kt)
  
Example has an implementation of both `select {...}`, that returns the result of one of its cases like a Kotlin 
[`when` expression](https://kotlinlang.org/docs/reference/control-flow.html#when-expression), 
and a convenience `whileSelect { ... }` that is the same as `while(select<Boolean> { ... })` with fewer braces.
  
The default selection case from [the 6th concurrency example of a tour of Go](https://tour.golang.org/concurrency/6) 
just adds one more case into the `select {...}` DSL:

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val tick = Time.tick(100)
    val boom = Time.after(500)
    whileSelect {
        tick.onReceive {
            println("tick.")
            true // continue loop
        }
        boom.onReceive {
            println("BOOM!")
            false // break loop
        }
        onDefault {
            println("    .")
            delay(50)
            true // continue loop
        }
    }
}
```

> You can checkout working code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-6.kt)

The `Time.tick` and `Time.after` are trivially implemented 
[here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/time.kt) with non-blocking `delay` function.
  
Other examples can be found [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/) together with the links to 
the corresponding Go code in comments.

Note, that this sample implementation of channels is based on a single
lock to manage its internal wait lists. It makes it easier to understand and reason about. 
However, it never runs user code under this lock and thus it is fully concurrent. 
This lock only somewhat limits its scalability to a very large number of concurrent threads.

> The actual implementation of channels and `select` in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  is based on lock-free disjoint-access-parallel data structures.

This channel implementation is independent 
of the interceptor in the coroutine context. It can be used in UI applications
under an event-thread interceptor as shown in the
corresponding [continuation interceptor](#continuation-interceptor) section, or with any other one, or without
an interceptor at all (in the later case, the execution thread is determined solely by the code
of the other suspending functions used in a coroutine).
The channel implementation just provides thread-safe non-blocking suspending functions.
  
### Mutexes

Writing scalable asynchronous applications is a discipline that one follows, making sure that ones code 
never blocks, but suspends (using suspending functions), without actually blocking a thread.
The Java concurrency primitives like 
[`ReentrantLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)
are thread-blocking and they should not be used in a truly non-blocking code. To control access to shared
resources one can define `Mutex` class that suspends an execution of coroutine instead of blocking it.
The header of the corresponding class would like this:

```kotlin
class Mutex {
    suspend fun lock()
    fun unlock()
}
```

> You can get full implementation [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/mutex/mutex.kt).
  The actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  has a few additional functions.

Using this implementation of non-blocking mutex
[the 9th concurrency example of a tour of Go](https://tour.golang.org/concurrency/9)
can be translated into Kotlin using Kotlin's
[`try-finally`](https://kotlinlang.org/docs/reference/exceptions.html)
that serves the same purpose as Go's `defer`:

```kotlin
class SafeCounter {
    private val v = mutableMapOf<String, Int>()
    private val mux = Mutex()

    suspend fun inc(key: String) {
        mux.lock()
        try { v[key] = v.getOrDefault(key, 0) + 1 }
        finally { mux.unlock() }
    }

    suspend fun get(key: String): Int? {
        mux.lock()
        return try { v[key] }
        finally { mux.unlock() }
    }
}
```

> You can checkout working code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-9.kt)

### Migration from experimental coroutines

Coroutines were an experimental feature in Kotlin 1.1-1.2. The corresponding APIs were exposed
in `kotlin.coroutines.experimental` package. The stable version of coroutines, available since Kotlin 1.3,
uses `kotlin.coroutines` package. The experimental package is still available in the standard library and the 
code that was compiled with experimental coroutines still works as before.

Kotlin 1.3 compiler provides support for invoking experimental suspending functions and passing suspending
lambdas to the libraries that were compiled with experimental coroutines. Behind the scenes, the 
adapters between the corresponding stable and experimental coroutine interfaces are created. 

### References

* Further reading:
   * [Coroutines Reference Guide](http://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html) **READ IT FIRST!**.
* Presentations:
   * [Introduction to Coroutines](https://www.youtube.com/watch?v=_hfBv0a09Jc) (Roman Elizarov at KotlinConf 2017, [slides](https://www.slideshare.net/elizarov/introduction-to-coroutines-kotlinconf-2017))
   * [Deep dive into Coroutines](https://www.youtube.com/watch?v=YrrUCSi72E8) (Roman Elizarov at KotlinConf 2017, [slides](https://www.slideshare.net/elizarov/deep-dive-into-coroutines-on-jvm-kotlinconf-2017))
   * [Kotlin Coroutines in Practice](https://www.youtube.com/watch?v=a3agLJQ6vt8) (Roman Elizarov at KotlinConf 2018, [slides](https://www.slideshare.net/elizarov/kotlin-coroutines-in-practice-kotlinconf-2018))
* Language design overview:
  * Part 1 (prototype design): [Coroutines in Kotlin](https://www.youtube.com/watch?v=4W3ruTWUhpw) 
    (Andrey Breslav at JVMLS 2016)
  * Part 2 (current design): [Kotlin Coroutines Reloaded](https://www.youtube.com/watch?v=3xalVUY69Ok&feature=youtu.be) 
    (Roman Elizarov at JVMLS 2017, [slides](https://www.slideshare.net/elizarov/kotlin-coroutines-reloaded)) 

### Feedback

Please, submit feedback to:

* [Kotlin YouTrack](http://kotl.in/issue) on issues with implementation of coroutines in Kotlin compiler and feature requests.
* [`kotlinx.coroutines`](https://github.com/Kotlin/kotlinx.coroutines/issues) on issues in supporting libraries.

## Revision history

This section gives an overview of changes between various revisions of coroutines design.

### Changes in revision 3.3

* Coroutines are no longer experimental and had moved to `kotlin.coroutines` package.
* The whole section on experimental status is removed and migration section is added.
* Some non-normative stylistic changes to reflect evolution of naming style.
* Specifications are updated for new features implemented in Kotlin 1.3:
  * More operators and different types of functions are supports.
  * Changes in the list of intrinsic functions:
  * `suspendCoroutineOrReturn` is removed, `suspendCoroutineUninterceptedOrReturn` is provided instead.
  * `createCoroutineUnchecked` is removed, `createCoroutineUnintercepted` is provided instead.
  * `startCoroutineUninterceptedOrReturn` is provided.
  * `intercepted` extension function is added.
* Moved non-normative sections with advanced topics and more examples to the appendix at end of the document to simplify reading.

### Changes in revision 3.2

* Added description of `createCoroutineUnchecked` intrinsic.

### Changes in revision 3.1

This revision is implemented in Kotlin 1.1.0 release.

* `kotlin.coroutines` package is replaced with `kotlin.coroutines.experimental`.
* `SUSPENDED_MARKER` is renamed to `COROUTINE_SUSPENDED`.
* Clarification on experimental status of coroutines added.

### Changes in revision 3

This revision is implemented in Kotlin 1.1-Beta.

* Suspending functions can invoke other suspending function at arbitrary points.
* Coroutine dispatchers are generalized to coroutine contexts:
  * `CoroutineContext` interface is introduced.
  * `ContinuationDispatcher` interface is replaced with `ContinuationInterceptor`.
  * `createCoroutine`/`startCoroutine` parameter `dispatcher` is removed.
  * `Continuation` interface includes `val context: CoroutineContext`.
* `CoroutineIntrinsics` object is replaced with `kotlin.coroutines.intrinsics` package.

### Changes in revision 2

This revision is implemented in Kotlin 1.1-M04.

* The `coroutine` keyword is replaced by suspending functional type.
* `Continuation` for suspending functions is implicit both on call site and on declaration site.
* `suspendContinuation` is provided to capture continuation is suspending functions when needed.
* Continuation passing style transformation has provision to prevent stack growth on non-suspending invocations.
* `createCoroutine`/`startCoroutine` coroutine builders are introduced.
* The concept of coroutine controller is dropped:
  * Coroutine completion result is delivered via `Continuation` interface.
  * Coroutine scope is optionally available via coroutine `receiver`.
  * Suspending functions can be defined at top-level without receiver.
* `CoroutineIntrinsics` object contains low-level primitives for cases where performance is more important than safety.








