---
layout: post
title:  "runCatching을 이용한 kotlin에서 예외 처리 방법."
author: uchun
categories: [ kotlin, exception ]
image: assets/images/covers/football.jpg
featured: true
---

exception이 발생할 수 있는 상황에서 try-catch를 이용해서 exception을 처리할 수 있습니다.<br>
kotlin 1.3부터는 exception이 발생할 수 있는 상황을 처리하기 위해 runCatching이라는 inline function을 제공하고 있습니다.

설명에 앞서 아래와 같이 랜덤하게 과일 이름을 출력하는 함수가 있습니다.

```kotlin
@Throws(Exception::class)
fun getRandomFruit(): String {
    val fruitName = listOf(
        "Avocado", "Blueberries", null,
        "Clementine", "Durian", "Guava"
    ).shuffled().first()

    return when (fruitName) {
        "Guava" -> throw IllegalStateException("Out of stock")
        null, "" -> throw NullPointerException()
        else -> fruitName
    }
}
```
여기서 구아바는 재고가 없어서 exception을 발생하게 했고.<br>
null이거나 empty여도 exception을 발생하게 했습니다.

getRandomFruit에서 발생하는 exception을 처리하기 위해서 아래와 같이 try-catch를 사용하여 처리할 수 있습니다.
```kotlin
val fruitName = try {
    getRandomFruit()
} catch (throwable: Throwable) {
    ""
}
```
kotlin 에서 try-catch는 expression이어서 java보다 편해졌습니다만<br>
try-catch만으로는 실행되는 영역과 결괏값을 변화시키는 영역, exception이 처리되는 영역, 최종적으로 값이 사용되는 영역의 구분이 번거롭습니다.

### runCatching 이용해보기

kotlin 에서 제공하는 runCatching은 아래와 같습니다. [공식 문서](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/run-catching.html)
```kotlin
public inline fun <R> runCatching(block: () -> R): Result<R> {
    return try {
        Result.success(block())
    } catch (e: Throwable) {
        Result.failure(e)
    }
}
```

그리고 앞에서 사용한 try-catch를 runCatching으로 바꾸어 보면 아래와 같이 표현할 수 있습니다.
```kotlin
val fruitResult = runCatching {
    getRandomString()
}
val fruitName = fruitResult.getOrNull()
```

runCatching의 return 값으로 전달받은 Result는 아래의 method와 properties를 제공합니다.

```kotlin
if (fruitResult.isSuccess) { }
if (fruitResult.isFailure) { }
val fruitName = fruitResult.getOrNull()
val throwable = fruitResult.exceptionOrNull()
```

isSuccess, isFailure 는 특별한 설명이 필요 없다고 생각하고,<br>
getOrNull으로 exception이 발생하지 않는 경우 value를, 그 외는 null을 받을 수 있고,<br>
exceptionOrNull은 그 반대입니다.

### map과 recover를 이용하여 값 변환하기

여기까지만 보면 조금 시시할 수 있습니다.<br>
하지만 Result에 대해 아래와 같은 extension 을 제공하고 있습니다.

```kotlin
Result<T>.getOrThrow(): T
Result<T>.getOrElse(onFailure: (exception: Throwable) -> R): R
Result<T>.getOrDefault(defaultValue: R): R
Result<T>.onSuccess(action: (value: T) -> Unit): Result<T>
Result<T>.onFailure(action: (exception: Throwable) -> Unit): Result<T>
Result<T>.fold(
    onSuccess: (value: T) -> R,
    onFailure: (exception: Throwable) -> R
): R

Result<T>.map(transform: (value: T) -> R): Result<R>
Result<T>.mapCatching(transform: (value: T) -> R): Result<R>
Result<T>.recover(transform: (exception: Throwable) -> R): Result<R>
Result<T>.recoverCatching(transform: (exception: Throwable) -> R): Result<R>
```

onSuccess, onFailure, fold로 성공과 실패(exception이 발생한 경우)를 따로 처리가 가능합니다.<br>
또는 getOrXXX로 exception이 발생하지 않고 잘 수행된 경우 원하는 결괏값을, 아닌 경우는 XXX 에 해당되는 동작을 하게 됩니다.

```kotlin
fruitResult.onSuccess {
    // 성공 시 받은 결괏값에 대한 처리
}.onFailure {
    // 실패 시 발생한 throwable을 처리
}

fruitResult.fold({
    // 성공 시 받은 결괏값에 대한 처리
}, {
    // 실패 시 발생한 throwable을 처리  
})

var fruitName: String? = null
// 실패 시 default 값을 반환
fruitName = fruitResult.getOrDefault("")
// 실패 시 else block의 결괏값을 반환
fruitName = fruitResult.getOrElse {
    when(it) {
        is IllegalStateException -> "Sold out"
        is NullPointerException -> "null"
        else -> throw it
    }
}
// 실패시 throwable이 다시 throw 됩니다.
fruitName = fruitResult.getOrThrow()
```

또한 map과 recover를 이용해 성공과 실패 시 원하는 값으로 바꿀 수 있습니다.<br>
둘 다 xxxCatching 을 제공하는데 map과 recover시 발생할 수 있는 exception 으로부터 안전하기 위해 runCatching으로 감싸저 있습니다.<br>
Result<T>로 반환하므로 chaining 하여 이용 가능합니다.<br>
또한 getOrElse 대신 아래와 같이 recover를 이용해 볼 수도 있습니다.

```kotlin
fruitResult.map {
    it.toUpperCase()
}

fruitResult.recover {
    when(it) {
        is IllegalStateException -> "Sold out"
        is NullPointerException -> "null"
        else -> throw it
    }
}
```

위에 얘기한 내용을 엮어보면 아래와 같은 방법으로 써볼 수 있습니다.<br>
(_map 경우는 큰 의미는 없지만 예제목적으로 mapCatching 을 사용하였습니다._)

```kotlin
val fruitName = runCatching {
    getRandomFruit()
}.mapCatching {
    it.toUpperCase()
}.recoverCatching {
    when(it) {
        is IllegalStateException -> "Sold out"
        is NullPointerException -> "null"
        else -> throw it
    }
}.getOrDefault("")
```

이 경우는 성공한 경우만 uppercase가 동작합니다 recover를 map보다 먼저 해주면 recover된 값 또한 map에서 변환될 수 있으니 작성 시 의도에 맞게 순서를 잘 정할 필요가 있습니다.

### 그 외 기타 사항

Result는 return 하려고 하면 **'kotlin.Result' cannot be used as a return type** 이라는 에러가 발생합니다.<br>
사용하고 싶은경우 다음의 [stackoverflow 글](https://stackoverflow.com/questions/52631827/why-cant-kotlin-result-be-used-as-a-return-type)에서 gradle 설정법에 대하여 설명 되어 있습니다.<br>
다만 위의 stackoverflow 답변에 있는 내용 대로 왜 막아두었는지 [여기](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#limitations)와 [여기](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#future-advancements)에 잘 설명되어 있습니다.

#### stackoverflow 및 관련 자료 링크
- https://stackoverflow.com/questions/52631827/why-cant-kotlin-result-be-used-as-a-return-type
- https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#limitations
- https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#future-advancements

#### 글 작성시 참고 한 글
- https://ahsensaeed.com/functional-style-error-handling-kotlin-runcatching-mapcatching/
