---
layout: post
title:  "코틀린 부트캠프 (몰랐던 개념들)"
date:   2018-09-25 15:49:57 +0900
categories: Learning Tech
---

Udacity 감사합니다. [Kotlin bootcamp!](https://www.udacity.com/course/kotlin-bootcamp-for-programmers--ud9011) :)

### 코틀린 REPL
- IntelliJ 에서 바로 확인가능
![Kotlin REPL](https://i.gyazo.com/a52acc31fd23748a0d7a02255d296c23.png)

### 코틀린 Boxing
```kotlin
// use primitive 'int' as an object
1.toLong()
1

// or, put int in a box
val boxed: Number = 1
boxed.toLong()
1
```

### 코틀린 operators
- Double bang operator
    - 만약 null 이면 NullpointerException 을 날림
- Question mark operator
    - 만약 null 이면 실행되지 하지 않고 null 을 반환함
    - Elvis 오퍼레이터로 null 을 반환하지 않고 다른 값을 반환하도록 명시 가능

```kotlin
val b:String? = null
println(b?.length) => null
println(b?.length ?: 0) => 0
```

### Loop and control
- if 문에 range 사용가능

```kotlin
if (fish in 1..100) println(fish)
for (i in 'b' .. 'g') print(i)
for (i in 1..5) print(i)
for (i in 5 downTo 1) print(i)
for (i in 3..6 step 2) print(i)
```
- `when`  사용가능 (java 에서는 if, else 가 난무했을텐데...)

```kotiln
var welcome Message = "Hello and welcome to Kotlin"
when (welcomeMessage.length) {
    0 -> println("Nothing to say?")
    in 1..50 -> println("Perfect")
    else -> println("Too long!")
}
```
- java 에도 있던 for-loop 가능

```kotlin
for (element in swarm) println(element)
```
- index 와 함께 loop

```kotiln
for((index, element) in swarm.withIndex()) {
    println("Fish at $index is $element")
}
```


### 함수
- compact function

```kotlin
fun getDirtySensorReading() = 20
getDirtySensorReading() // 호출하면 바로 20 반환
```

- 코틀린 standard lib 를 이용하여 control flow 를 만들 수 있음

```kotlin
repeat(2){
   println("A fish is swimming")
}
```

- eager vs lazy
    - 기본은 eager, `asSequence()` 를 사용하면 lazy 하게 구현 가능

```kotlin
val filtered = decorations.asSequence().filter { it[0] == 'p' } // println(filtered) 는 아직 filtering 하기 전임
println(filtered.toList()) // 사용할 때, lazy 하게 계산함

val lazyMap = decorations.asSequence().map { println("map: $it") it }
println(lazyMap) // map 하지 않아
println("first: ${lazyMap.first()}") // 직접 사용할 때, map 연산을 시작
```

### 고차 함수(high order function): 함수를 인자로 활용가능
- 3가지 방법으로 인자로 넘기기 가능

```kotlin
fun updateDirty(dirty: Int, operation: (Int) -> Int): Int {
    return operation(dirty)
}

fun feedFish(dirty: Int) = dirty + 10

val waterFilter: (Int) -> Int = { dirty -> dirty/2 }
dirty = updateDirty(dirty, waterFilter) // 선언된 람다로 넘기기
dirty = updateDirty(dirty, ::feedFish) // 선언된 함수 이름으로 넘기기
dirty = updateDirty(dirty, { dirty -> // 람다로 선언하기
    dirty + 50
}
```

### Class
- object 는 기본적으로 public
- internal 키워드는 해당 모듈 안에서만 접근 가능하다는 의미
- init 키워드는 primary constructor 에서 실행될 로직을 명시해줄 수 있음
    - primary constructor 는 아무런 코드를 가지지못해. 그래서 init keyword 로 그걸 분리;;

```kotlin
class InitOrderDemo(name: String) {
    val firstProperty = "First property: $name".also(::println)
    
    init {
        println("First initializer block that prints ${name}")
    }
    
    val secondProperty = "Second property: ${name.length}".also(::println)
    
    init {
        println("Second initializer block that prints ${name.length}")
    }
}
fun main(args: Array<String>) {
    InitOrderDemo("hello")
}

/*
First property: hello
First initializer block that prints hello
Second property: 5
Second initializer block that prints 5
*/
```

- class 는 기본적으로 상속을 막음 
    - 하려면, open 이라고 명시해줘야함
    - 멤버변수도 동일
- 싱글턴 클래스 object!
- companion object 키워드는 static 이란 의미로, 기존 자바에서 클래서 내 `static` block 처럼 사용가능
    - 그 안에서 정적 함수는 @JvmStatic 을 붙여야 사용가능

### By 
- delegation 으로 어떤 인터페이스 구현을 사용할지 지정 가능

![By!](https://gyazo.com/d6376cd415df0ffc21c3432d1e57b561.png)

### Sealed class
- 같은 모듈안에서만 상속할 수 있도록 강제
- 다른 곳에서 상속할 수 없음
- 그래서 필요한 타입을 딱 정해줄 수 있음 (커스텀한 타입 선언 막음)

### Pair
```kotlin
val equipment = "fishnet" to "catching fish" // 라고하면 pair 로 묶임

// unpack 가능
var (tool, use) = equipment

// 함수 반환
fun giveMeATool(): Pair<String, String> {
    return ("fishnet" to "catching")
}
```

### Map
- map 에서는 getOrDefault() 가 존재

```kotiln
println(cures.getOrDefault("bloating", "sorry I don't know"))
```

- 아니면 실행도록도 할 수 있음

```kotlin
cures.getOrElse("bloating") { "No cure for this" }
```

### Const
- `const val num = 5` 는 컴파일타임에 결정됨
    - 그래서, `const val result = complexFuntionCall()` 이런게 안됨 

### Extension Function 
- 확장 함수는 기존 존재하는 클래스에 함수 추가하는 것

```kotlin
fun String.hasSpaces() = find { it == ' ' } != null
```

- 확장 함수는 static 으로 지정되어서 자식 클래스가 재정의한 메소드가 있다고해도 부모 클래스의 extension 함수가 실행됨.

![extension](https://gyazo.com/1952e861cfebe8f6270162231b5f2cee.png)

- extension 에서 private 는 접근 할 수 없음

![private](https://gyazo.com/3aa0f87170a6e75678f09296e7e35757.png)

- Receiver
    - 확장 함수를 만들 때, receiver 를 사용하면 내부에서 this 가 접근 가능
    - 목적은 DSL 만드는 것

```kotlin
fun task(): List<Boolean> {
    val isEven: Int.() -> Boolean = { this % 2 == 0 }
    val isOdd: Int.() -> Boolean = { this % 2 != 0 }

    return listOf(42.isOdd(), 239.isOdd(), 294823098.isEven())
}
```
```kotlin
fun String?.notEmptyString(callback: Callback) {
    if (!this.isNullOrEmpty()) { 
        callback.notEmptyString(this!!)
    }
}

// 람다로 this 넘기기
fun String?.notEmptyString(callback: (String) -> Unit) {
    if (!this.isNullOrEmpty()) {
        callback(this!!)    
    }
}

// 이를 extension 을 사용해서 receiver 로 this 를 넘길 수 있음 인자 없이
fun String?.notEmptyString(callback :String.() -> Unit) {
    if (!this.isNullOrEmpty) {
        this!!.callback()
    }
}
```

### Generic
- 자바 Generic 과 동일
- 의도한 타입이 들어올 수 있도록 `T: WaterSupply` (부모 클래스가 WaterSupply 인 클래스만 들어올 수 있도록) 지정 가능
- Variant

```kotlin
class Aquarium<T: WaterSupply>(val waterSupply: T) ...
fun addItemTo(aquarium: Aquarium<WaterSupply>) = println("item added")

// Aquarium 자식인 FishStoreWater, LakeWater 가 있을 때
fun addItemTo(fishStoreWater) // Compile error! 

/* 왜냐면 인자 타입을 Aquarium<WaterSupply> 를 기대 했기 때문에, 이를 가능케하려면 Aquarium<out T: WaterSupply> 로 T 가 WaterSupply 의 out 놈들은 받을 수 있음 */
```
- T (읽기, 쓰기 모두 가능)
- Out types can be used as return values. (읽기만 가능한 서브타입)
- In types can be used as parameters. (쓰기만 가능한 슈퍼타입)
- 메소드도 generic 사용가능

### Annotation
- 자바와 동일하게 애노테이션 생성가능

```kotlin
@ImAPlant class Plant {
    fun trim() {}
}
```

- 이를 어떤 클래스에 붙이고 reflection 처럼 클래스 메타정보로 가져올 수 있음

```kotlin
val classObj = Plant::class
for (annotation in classObj.annotations) {
    println(annotation.annotationClass.simpleName)
    val annotated = classObj.findAnnotation<ImAPlant>()
    annotated?.apply {
        println("Found a plant annotation!")
    }
}
```

### inline
- `inline` 를 달면 매번 object 를 만들지 않는다. 성능에 좋음

```kotlin
myWith(fish.name) {
    capitalize()
}

inline fun myWith(name: String, block: String.() -> Unit) {
    name.block()
}
```

### SAM
- Single Abstract Method
- 함수를 하나로 가지는 인터페이스를 lambda로 구현할 수 있음

### 그 밖에
- 코틀린에서는 모든 것이 Expression! (코틀린에서는 자바 void 를 Unit 으로 표현함)
- named parameter 사용가능

