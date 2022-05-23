# Effective Kotlin Item 50: 쓸모없는 객체 참조를 제거해라

> 이번 챕터는 Effective Kotlin 챕터에 나오는 책 중 하나이다.

자동으로 메모리를 관리해주는 언어를 사용하는 프로그래머들은 인스턴스를 Free 해주는 것에 관해 잘 생각하지 않는다.
자바에서는 쉽게 예를 들면 GC 가 그러한 일을 수행한다. 하지만 메모리를 관리하는 것에 대한 것을 까먹게되면 종종 불필요한 메모리 소비 - memory leak 그리고 몇가지 경우에서 OutOfMemoryError 을 유발할 수 있다. 가장 **중요한 규칙은 특히 메모리에서 큰 부분을 차지하고 있는 특정 오브젝트나, 이와 같은 개체의 인스턴스가 많을 경우에 우리는 더이상 유용하지 않은 객체 참조를 유지해서는 안된다는 것이다.**

안드로이드에서는, "액티비티에 대한 참조 가지기": "Act Activity 를 많은 안드로이드 함수에서 필요로 하고, 편리하기때문에 초보자들은 그것을 Companion Objcet 또는 top-level property 에 저장하는 실수를 자주한다.

```kotlin
class MainActivity : Activity() {
​
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //...
        activity = this
    }
​
    //...
​
    companion object {
        // DON'T DO THIS! It is a huge memory leak
        var activity: MainActivity? = null
    }
}

```

Activity 의 참조를 companion objcet 에서 계속 들고있는 것은 GC 에게 우리의 Application 이 작동하는 동안에는 objcet 를 release 하는걸 허락하지 않는 다는 것을 의미한다.
이러한 상황을 향상시킬 몇가지 방법 이 있지만 자원을 정적으로 보유하지 않는 것이 최선이다. **Dependency 를 정적으로 저장하는 것보다 적절하게 저장하는 것이 좋다.** 
또한 우리는 다른 객체의 참조를 들고있는 오브젝트를 잡아둘때 Memory leak 을 유발할수 있음을 알고있어야 한다. 아래의 예시를 보면 우리는 Lambda function 에서 MainActivity 의 참조를 가지고 있음을 알 수 있다.

```kotlin
class MainActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //...

        // Be careful, we leak a reference to `this`
        logError = {
            Log.e(
                this::class.simpleName,
                it.message
            )
        }
    }

    //...

    companion object {
        // DON'T DO THIS! A memory leak
        var logError: ((Throwable) -> Unit)? = null
    }
}
```

하지만 메모리와 관련된 문제는 인지하기 힘들다. 스택을 구현한 아래의 예제를 보자.

```kotlin
class Stack {
    private var elements: Array<Any?> = arrayOfNulls(DEFAULT_INITIAL_CAPACITY)
    private var size = 0

    fun push(e: Any) {
        ensureCapacity()
        elements[size++] = e
    }

    fun pop(): Any? {
        if (size == 0) {
            throw EmptyStackException()
        }
        return elements[--size]
    }

    private fun ensureCapacity() {
        if (elements.size == size) {
            elements = elements.copyOf(2 * size + 1)
        }
    }

    companion object {
        private const val DEFAULT_INITIAL_CAPACITY = 16
    }
}
```

여기서 문제점을 찾을 수 있는가? 생각할 시간을 몇분 가져봐라.
문제는 우리가 `pop` 메소드를 호출할때 일어난다. **우리는 array 의 size 를 단순히 줄인것이지만, 우리는 줄인 array 의 원소를 free(C 에서의 Memory free) 해주지 않았다.**
우리가 1000 개에 달하는 사이즈의 array 를 1이 될때까지 pop 했다고 한다해도. 우리의 Stack 은 여전히 1000개의 원소들을 GC 가 Collection 하지 못하게 쥐고 있다.
이렇게 쓸모없는 objcet 들은 우리의 메모리를 낭비하고 있을것이다. 이것이 메모리 leak 을 유발하는 이유이다.
이러한 leak 이 모이면 우리는 OutOfMemoryError 를 마주하게 된다. 우리는 어떻게 이 구현을 수정할 수 있을까? 아주 간단하게 고치는 방법은 object 가 더이상 필요없다고 느껴질때 null 을 넣어주는 것이다.

```kotlin
fun pop(): Any? {
    if (size == 0)
        throw EmptyStackException()
    val elem = elements[--size]
    elements[size] = null
    return elem
}
```

우리는 그 값이 더이상 필요없다고 느껴질때 그것을 release(free) 해줘야 함을 알아야만 한다. 이러한 규칙을 놀랍게도 많은 클래스에 적용되 있다. 또 다른 예시를 보자.
우리가 MutableLazy property delegate 가 필요하다고 해보자. 이것은 lazy 하게 동작할 것 같지만, property 의 상태가 불변임을 보장해주어야 할 것이다. 나는 따라서 아래와 같이 구현했다.

```kotlin
fun <T> mutableLazy(
    initializer: () -> T
): ReadWriteProperty<Any?, T> =
    MutableLazy(initializer)

private class MutableLazy<T>(
    val initializer: () -> T
) : ReadWriteProperty<Any?, T> {

    private var value: T? = null
    private var initialized = false

    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        synchronized(this) {
            if (!initialized) {
                value = initializer()
                initialized = true
            }
            return value as T
        }
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        synchronized(this) {
            this.value = value
            initialized = true
        }
    }
}

// usage
var game: Game? by mutableLazy { readGameFromSave() }

fun setUpActions() {
    startNewGameButton.setOnClickListener {
        game = makeNewGame()
        startGame()
    }
    resumeGameButton.setOnClickListener {
        startGame()
    }
}
```

위의 MutableLazy 의 구현은 잘 동작할것 같지만 한가지  `initializer` 가 사용되고 난뒤 **cleaned(본문의 표현은 이런데 나는 release 라고 하고 싶다.)** 되지 않는다는 한가지 결함이 존재한다.
이것은 MutableLazy 가 필요없어 졌음에도 불구하고, MutableLazy 의 참조가 존재한다는 것을 의미한다. MutableLazy 를 향상시키는 방법은 아래와 같다.

```kotlin
fun <T> mutableLazy(
    initializer: () -> T
): ReadWriteProperty<Any?, T> =
    MutableLazy(initializer)

private class MutableLazy<T>(
    var initializer: (() -> T)?
) : ReadWriteProperty<Any?, T> {

    private var value: T? = null

    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        synchronized(this) {
            val initializer = initializer
            if (initializer != null) {
                value = initializer()
                this.initializer = null
            }
            return value as T
        }
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        synchronized(this) {
            this.value = value
            this.initializer = null
        }
    }
```

우리가 intializer 에 null 을 넣어줬을때 GC 에 의해 이전값은 재활용될것이다. 이러한 성능향상이 얼마나 중요할것 같은가? 객체들을 사용하는데 그다지 중요하지 않을 것이다. **"성급한 최적화는 모든 악의 근원이다"** 라는 말이 있다.
하지만 필요하지 않은 참조를 null 로 대체하는 것은 좋은 것이다. 많은 변수 또는 알지못하는 값들을 포함하는 Function Type 에게는 특별히 더 좋다. 예를 들면 위의 Stack 예시를 보면 이것은 General 한 도구이기 때문에 누군가에게는 Heavy 한 Object 를 담을 수 있으며, 우리는 어떻게 사용될지 알수 없다. **이러한 General 한 도구들은**(본문에서는 Such tools 라고 하나, 나는 tools 에 General tools 가 담겨있다고 의역했다.) **우리가 성능 부분에 관해 좀더 유의깊게 다뤄야 한다. 특히 우리가 Library 로 만든다면 말이다.** 예를들어, Kotlin stdlib 에서 lazy delegate 의 구현체 3가지 모두 initializer 를 사용한 후 Null 을 넣음을 알 수 있다.

```kotlin
private class SynchronizedLazyImpl<out T>(
    initializer: () -> T, lock: Any? = null
) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean =
        _value !== UNINITIALIZED_VALUE

    override fun toString(): String =
        if (isInitialized()) value.toString()
        else "Lazy value not initialized yet."

    private fun writeReplace(): Any =
        InitializedLazyImpl(value)
}
```

General 한 규칙은, **우리가 메모리를 관리한다는 마음가짐을 지니고 있어야 한다는 것**이다. 구현을 바꾸기 전에 우리는 항상 우리의 프로젝트에서 Best-Trade-off 가 무엇인지 생각해야 한다.
우리의 해결법은 오직 메모리와 성능만을 염두에 둔것이 아닌, 가독성과 확장성 또한 염두에 두어야 한다. 일반적으로 읽기 좋은 코드는 메모리와 성능 부분에서 좋다. 읽기 어려운 코드는 CPU Power 를 낭비하거나 메모리 leak 을 더욱 숨키고 있을 수 있다.
때때로, 이런 두가지 이념이 충돌하는데 대부분의 경우에서는 읽기 좋은 코드가 좋다. 우리가 성능과 메모리적 부분이 자주 고려되는 일반적 목적의 라이브러리(위에 적은 Stack 과 같은) 은 경우에 읽기 어려운 코드가 좋을 수도 있다.

Memory leak 에는 일반적으로 몇가지 원인이 있다. 첫째로 사용하지 않는 객체를 caches 해두는 것이다. 캐시를 해두는건 idea 같다고 생각할 수 있지만, 우리가 OutOfMemoryError 를 만났을때는 아무런 도움을 주지 못한다. 해결첵은 **soft-references** 를 사용하는 것이다. 이렇게 했을때, Memory 가 필요할때 종종 GC 에 의해 수집될 수 있으나, 대부분 객체는 존재하며 사용될 것 이다.
화면에 띄어지는 dialog 와 같은 몇몇 객체는 **weak-reference** 를 사용하는데 이 경우 display 되어 있다면 GC 가 되지 않는다. 만약 사라질때 참조를 더 이상 필요로 하지 않는다면, 이 객체는 참조 모델에 weak-reference 를 사용해야 하는 완벽한 후보이다.

Memory leak 의 가장 큰 문제는 어플리케이션이 Crash 나기 전까지 나타나지 않으며, 때때로 예측하기 힘들다는 것이다. 

## 몰랐던 워딩 및 문장

- There are some ways to improve this situation, but it is best not to hold such resources statically at all.  
    -> 이러한 상황을 향상시킬 몇가지 방법 이 있지만 자원을 정적으로 보유하지 않는 것이 최선이다.

- **subtle**  
    -> 알기힘든, 감지하기 힘든, 미묘한

- **accumulate**  
    -> 모으다.

- **flaw**  
    -> 결함

- **bearing in mind**
    -> 염두에 둔

## 원문 링크

https://kt.academy/article/ek-object-references