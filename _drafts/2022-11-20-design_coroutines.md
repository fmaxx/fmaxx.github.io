---
layout:     post
title:      Дизайн Kotlin Coroutines
date:       2022-11-20 12:02:00
summary:    Корутины изнутри.
categories: Android Kotlin Coroutines Error Exception
---

Перевод статьи [Design of Kotlin Coroutines](https://proandroiddev.com/design-of-kotlin-coroutines-879bd35e0f34/){:target="_blank"}

![Как вообще создаются корутины?](/images/design_coroutines/1.jpeg)

Большинство Android разработчиков используют корутины, но кто знает как создаются корутины за сценой? 

Наш примерный план такой:

1. **Определения**
2. **CPS — Continuation Passing Style**
3. **Принцип работы Kotlin корутин**
    * **3.1 Создание корутин**
        * 3.1.1 launch()
        * 3.1.2 start()
        * 3.1.3 invoke()
        * 3.1.4 startCoroutineCancellable()
        * 3.1.5 resumeWithCancellable()
        * 3.1.6 resumeWith()
        * 3.1.7 invokeSuspend()
        * 3.1.8 Выводы по созданию корутин 
    * **3.2 Анализ байткода**

<br/>
## **1. Определения**

# Что такое корутина? 
> Корутина это экземпляр приостанавливаемого блока вычислений. Концептуально похожа на поток (thread), также конкурентно запускается блок кода. Однак корутина не привязана к какому-либо потоку. Кроме того корутина может приостанавить свое выполнение в одном потоке и возобновить в другом.

В асинхронных программах задачи работают параллельно на отдельных потоках, не ожидая, пока другие задачи выполнятся. Неверное использование мультипоточности может приводить к интенсивному использованию CPU, тем самым резко снизить производительность вашего приложения. Потоки являются достаточно дорогим ресурсом. Корутины - легковесная альтернатива нативным потокам.

<br/>
# Что такое suspend функция? 

> Suspend это функция, которая может быть запущена (**_started_**), приостановлена (**_paused_**) и возобновлена (**_resumed_**). Одна из самых важных вещей о suspend функции - она может быть вызвана только из корутины или другой suspend функции.

![suspend функция](/images/design_coroutines/2.jpeg)

Когда корутина приостановлена, текущий поток на котором работала корутина становится свободным для исполнения других корутин. При этом корутина может возобновить свою работу уже на другом потоке. Как вывод, можно запускать множество корутин на небольшом количестве потоков. Ниже мы увидим, как все это работает.

<br/>
# Suspend против обычных функций:
- у suspend функций ноль и больше точек приостановки, тогда как обычные функции не имеют таких точек;
- обычные функции не могут напрямую вызывать suspend функции, так как первые не поддерживают точки приостановки;
- suspend функции могут вызывать обычные функции, потому что обычные функции имеют ноль точек приостановки.

Это простая suspend функция:

```kotlin
suspend fun functionA(): String {
     return  "hello"
}
```

Как мы говорили выше, мы не можем вызвать suspend функцию внутри обычной. Попробуем декомпилировать **functionA()**  (идем **Tools->Kotlin->Show Kotlin Bytecode**) и посмотреть, что там происходит:

```java
@Nullable
public static final Object functionA(@NotNull Continuation $completion) {
  return "hello";
}
```
![что же случилось?](/images/design_coroutines/3.jpeg)

Вам вероятно интересно откуда приходит аргумент Continuation и что он значит. Сейчас мы выясним откуда он появляется, а позже увидим, что он значит. Каждая suspend функция проходит преобразование к **CPS-Continuation Passing Style**.

Можно представить приостановку работы корутины на таком примере. Представим, что мы играем в игру и хотим приостановить (suspend) ее, а затем продолжить с этого же места. В этом случае, мы сохраняем состояние игры и когда захотим продолжить, мы можем восстановить из сохранения последнюю приостановленную точку. Когда процесс приостановлен, он возвращает объект Continuation. 

<br/>

## **2. CPS — Continuation Passing Style**
Представим suspend функцию с _n_ аргументами (_p_1, p_2... p_n_) и возвращаемым результатом `T`. После CSP трансформации к suspend функции добавляется аргумент _p_n+1_ с типом `Continuation<T>` и возвращаемый результат становится `Any?` потому что:

- если функция возвращает какой-то результат, мы получаем `T`;
- если функция приостановилась, она вернет специальный сигнал `COROUTINE_SUSPENDED` - функция в suspended состоянии.

![CPS — Continuation Passing Style](/images/design_coroutines/4.jpeg)

<br/>

## **3. Принцип Kotlin корутины**

Давайте посмотрим как создать корутину на конкретном примере и разберем ход ее работы.

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        lifecycleScope.launch {
            val randomNum = getRandomNum() // suspension point #1
            val sqrt = getSqrt(randomNum.toDouble()) // suspension point #2
            log(sqrt.toString())
        }
    }

    private suspend fun getRandomNum(): Int {
        delay(1000)
        return (1..1000).shuffled().first()
    }

    private suspend fun getSqrt(num: Double): Double {
        delay(2000)
        return sqrt(num)
    }

    private fun log(text: String) {
        Log.i(this@MainActivity::class.simpleName, text)
    }
}
```
В этом примере у на две suspend функции. `getRandomNum()` функция возвращает случайное число от 1 до 1000. `getSqrt()` расчитывает квадратный корень. 

Поставим точку останова на линии 6 (`lifecycleScope.launch...`) и запустим в режиме отладки. Посмотрим трассировку стеков до запуска корутины.

![line 6 breakpoint](/images/design_coroutines/5.jpeg)

<br/>

Трассировка должна выглядеть примерно так:

```
invokeSuspend:75, MainActivity$startCoroutine$1 (me.aleksandarzekovic.exploringcoroutines)
resumeWith:33, BaseContinuationImpl (kotlin.coroutines.jvm.internal)
resumeCancellableWith:266, DispatchedContinuationKt (kotlinx.coroutines.internal)
startCoroutineCancellable:30, CancellableKt (kotlinx.coroutines.intrinsics)
startCoroutineCancellable$default:25, CancellableKt (kotlinx.coroutines.intrinsics)
invoke:110, CoroutineStart (kotlinx.coroutines)
start:126, AbstractCoroutine (kotlinx.coroutines)
launch:56, BuildersKt__Builders_commonKt (kotlinx.coroutines)
launch:1, BuildersKt (kotlinx.coroutines)
launch$default:47, BuildersKt__Builders_commonKt (kotlinx.coroutines)
launch$default:1, BuildersKt (kotlinx.coroutines)
startCoroutine:75, MainActivity (me.aleksandarzekovic.exploringcoroutines)
onCreate:71, MainActivity (me.aleksandarzekovic.exploringcoroutines)
...
```

<br/>

Порядок выполнения:

![порядок выполнения](/images/design_coroutines/6.jpeg)


<br/>

## **3.1 Конструирование корутин**

# 3.1.1 launch()

![launch](/images/design_coroutines/7.jpeg)

Запускает новую корутину без блокировки текущего потока и возвращает ссылку на корутину [`Job`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/){:target="_blank"}.

<br/>

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

<br/>

# **CoroutineScope**

Определяет область действия (scope) для корутины. Каждая **билдер-функция** (как [`launch`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html){:target="_blank"}, [`async`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html){:target="_blank"} и т.д.) является расширением [`CoroutineScope`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html){:target="_blank"} и наследует [`coroutineContext`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/coroutine-context.html){:target="_blank"} оттуда.

`CoroutineScope` это интерфейс с единственным полем `coroutineContext`.

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

Билдер-функция `launch` принимает три аргумента:

* _context_ - дополнительный к [`CoroutineScope.coroutineContext`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/coroutine-context.html){:target="_blank"} контекст корутины

* _start_ - опция запуска корутины. Значение по умолчанию [`CoroutineStart.DEFAULT`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-d-e-f-a-u-l-t/index.html){:target="_blank"} 

* _block_ - блок кода (лямбда), который будет вызван в контексте предоставленной scope. Компилятор создает для лямбды внутренний класс наследник от `SuspendLambda` и реализующий интерфейс [`Function2`](https://github.com/JetBrains/kotlin/blob/master/spec-docs/function-types.md){:target="_blank"}:

```kotlin
final class me/aleksandarzekovic/exploringcoroutines/MainActivity$onCreate$1 
  extends kotlin/coroutines/jvm/internal/SuspendLambda 
    implements kotlin/jvm/functions/Function2
```

Это значит, что Kotlin создает для каждой корутины внутренний анонимный класс `SuspendLambda` при компиляции. Это класс "тела" корутины. Внутри реализованы два метода:

* _`invokeSuspend()`_ - содержит код в теле корутины, который обрабатывает внутри состояние корутины. Наиболее важная часть логики по обработке состояний находится здесь, это называется **машина состояний** (state machine);

* _`create()`_ - получает объект `Continuation`, затем создает и возвращает объект класса "тела" корутины.

# **Машина состояний корутины**

> Kotlin реализует `suspend` функции как машины состояний, так как такая реализация не требует специальной поддержки в рантайме. Из-за этого нужна обязательная маркировка корутин как `suspend` функций (окраска функций): компилятор должен знать, какая функция может быть приостанавливаемой, чтобы превратить ее в машину состояний.

<br/>

Машина состояний соотвествует точками приостановки. В нашем примере это:

![точки приостановки](/images/design_coroutines/8.jpeg)

<br/>

На рисунке изображены три состояния для этого кода:

* **L0** до точки приостановки **#1**
* **L1** до точки приостановки **#2**
* **L2** конечное


![точки приостановки](/images/design_coroutines/9.jpeg)

<br/>

Псевод-код сгенерированной машины из трех состояний должен выглядеть примерно так: 
```kotlin
// The initial state of the state machine
int label = 0
A a = null
B b = null

void resumeWith(Object result) {
    if (label == 0) goto L0
    if (label == 1) goto L1
    if (label == 2) goto L2
    else throw IllegalStateException("call to 'resume' before 'invoke' with coroutine")

  L0:
    // result is expected to be `null` at this invocation
    label = 1
    // 'this' is passed as a continuation
    result = getRandomNum(this) 
    if (result == COROUTINE_SUSPENDED) return
  L1:
    A a = (A) result
    label = 2
    val result = getSqrt(a, this) // 'this' is passed as a continuation
    if (result == COROUTINE_SUSPENDED) return
  L2:
    B b = (B) result
    log(String.valueOf(b))
    label = -1 // No more steps are allowed
    return
}    
```

<br/>

Логика корутины работает в методе `invokeSuspend`, мы его уже упоминали.

Иерархия наследования `SuspendLambda` показана ниже: 

![SuspendLambda наследование](/images/design_coroutines/10.jpeg)

<br/>

# **Continuation**

> Интерфейс "продолжение" для возобновления работы после точки приостановки, которая возвращает значение **T**.

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
view raw
```

Continuation объекты также важны, они дают возможность продолжить работу корутины. Каждая suspend функция связана с сгенерированным экземпляром `Continuation`, который обрабатывает приостановку. 

* _context_ - контекст корутины, соответствует текущему Continuation;

- _resumeWith()_ - используется для передачи результатов между точками приостановки. Вызывается с результатом (или исключением) последней точки приостановки и возобновляет работу корутины.

<br/>

# **BaseContinuationImpl**

Ключевой код `BaseContinuationImpl`:

```kotlin
internal abstract  class  BaseContinuationImpl (...) {
     // Implement resumeWith of Continuation
     // It is final and cannot be overridden!
    public  final override fun resumeWith (result: Result<Any?>) {
        // ...
        val  outcome  = invokeSuspend(param)
        // ...
    }
    // For implementation 
    protected abstract fun invokeSuspend (result: Result<Any?>) : Any?
}
```

`invokeSuspend()` - абстрактный метод, реализующий в корутине класс-тело при компиляции.

Метод `resumeWith()` всегда вызывает `invokeSuspend()`.

<br/>

# **ContinuationImpl**
`ContinuationImpl` наследуется от `BaseContinuationImpl`. Его задача генерировать объект `DispatchedContinuation` с помощью перехватчика, который также является `Continuation`. Мы поговорим подробнее об этом в разделе **3.2.4**.

Давайте продолжим анализировать тело функции `launch()`:

![launch()](/images/design_coroutines/11.webp)

<br/>

# **newCoroutineContext()**

`newCoroutineContext` - создает контекст для новой корутины. Контекст создается по-умолчаню с [`Dispatchers.Default`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html){:target="_blank"}, если не указаны другие или [`ContinuationInterceptor`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html){:target="_blank"}, также добавляет опционально подддежку отладки (если она включена) и копируемые thread-local средства JVM.

`newCoroutineContext()` - это функция-расширение CoroutineScope. Ее назначение это образование нового контекста из родительского (из CoroutineScope) и контекста, переданного как аргумент.

![ух я потерялся](/images/design_coroutines/12.webp)

Давайте вкратце посмотрим на `CoroutineContext`.

<br/>

# **CoroutineContext**

CoroutineContext это неизменяемое индексированное множество _Element_, таких как [`CoroutineName`](https://www.google.com/search?client=safari&rls=en&q=Coroutinename&ie=UTF-8&oe=UTF-8){:target="_blank"}, `CoroutineId`, [`CoroutineExceptionHandler`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/){:target="_blank"}, [`ContinuationIntercepter`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/){:target="_blank"}, [`CoroutineDispatcher`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/){:target="_blank"}, [`Job`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/){:target="_blank"}. Каждый элемент этого множества имеет уникальный ключ `Key`.


Представьте, что мы хотим контролировать на каком потоке или пуле потоков будет выполняться корутина. В зависимости от того, хотим ли мы запустить задачу на главном потоке, связанную с интенсивными вычислениям или операциями ввода/вывода (IO) мы будем использовать разные типы `dispatchers` (диспетчера).

`Dispatchers` - планировщики потоков работающие с корутинами, используются чтобы переключать потоки и определять группу потоков на которых работает корутина. Есть четыре вида диспетчеров:

* [**Dispatchers.Default**](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html){:target="_blank"}.

* [**Dispatchers.IO**](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html){:target="_blank"}.

* [**Dispatchers.Main**](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html){:target="_blank"}.

* [**Dispatchers.Unconfined**](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html){:target="_blank"}.

Все эти диспетчеры являются `CoroutineDispatcher`. Как мы говорили выше, `CoroutineDispatcher` один из элементов `CoroutineContext`, а значит все эти диспетчеры также элементы `CoroutineContext`.

Мы говорили, что `CoroutineContext` похож на коллекцию и эта коллекция может содержать разные типы `Element`. Мы можем создать новый контекст добавляя/удаляя элементы или через слияние двух контекстов. Оператор [_plus_](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/plus.html){:target="_blank"} работает как расширение [_Set.plus_](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html){:target="_blank"}, он возвращает комбинацию из двух контекстов, где элементы правой части, заменяются на элементы с такими же ключами из левой части.

Экземпляр `EmptyCoroutineContext` создает контекст без элементов.

<br/>

![примеры](/images/design_coroutines/13.webp)

<br/>

Пример 1:
```kotlin
import kotlinx.coroutines.*

fun main() {
    val coroutineName = CoroutineName("C#1") + CoroutineName("C#2")
    println(coroutineName)
}

// result
// CoroutineName(C#2)
```

Пример 2:
```kotlin
import kotlinx.coroutines.*

fun main() {
    val coroutineContext = CoroutineName("C#1") + Dispatchers.Default
    println(coroutineContext)
}

// result
// [CoroutineName(C#1), Dispatchers.Default]
```

Пример 3:
```kotlin
import kotlinx.coroutines.*

fun main() {
    val firstCoroutineContext = CoroutineName("C#1") + Dispatchers.Default
    println(firstCoroutineContext)
    
    val secondCoroutineContext = Job() + Dispatchers.IO
    println(secondCoroutineContext)
    
    val finalCoroutineContext = firstCoroutineContext + secondCoroutineContext
    println(finalCoroutineContext)
}

// result
// [CoroutineName(C#1), Dispatchers.Default]
// [JobImpl{Active}@39a054a5, Dispatchers.IO]
// [CoroutineName(C#1), JobImpl{Active}@39a054a5, Dispatchers.IO]
```

![примеры](/images/design_coroutines/14.webp)


`CoroutineContext` никогда не переопределяется, а объединяется с существующим. Теперь, когда мы узнали несколько вещей о `CoroutineContext`, мы можем вернутся немного назад, туда где остановились с `newCoroutineContext` в билдер-функции `launch`.

<br/>

Давайте определим различные контексты чтобы легче все понимать:

* **_scope context_** - контекст определенный в `CoroutineScope`

* **_passed context_** - экземпляр `CoroutineContext`, полученный как первый аргумент в билдер-функции

* **_parent context_** - suspend блок в билдер-функции имеет получателя `CoroutineScope`, который также предоставляет `CoroutineContext`. Этот контекст не новый контекст корутины!

![scope](/images/design_coroutines/15.webp)

* Новая корутина создает собственный дочерний экземпляр `Job` (используя `Job` из этого контекста как родительскую) и определяет свой контекст корутины (дочерний контекст) как родительский плюс свой `Job`. Мы рассмотрим детально, как это делается, чуть позже.


После определения контекста новой корутины (из родительского контекста), ее можно создавать.

![coroutine code](/images/design_coroutines/16.webp)

<br/>

# **CoroutineContext**

Мы используем новый (родительский) контекст для создания корутины. Аргумент `start` по умолчанию равен `CoroutineStart.DEFAULT`. В этом случае, мы создаем `StandaloneCoroutine` (наследуется от `AbstractCoroutine`) с возвращаемым типом [_Job_](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/){:target="_blank"}. `StandaloneCoroutine` это экземпляр корутины.

> Если мы установим ленивый запуск, то у нас будет экземпляр `LazyStandaloneCoroutine`, который наследуется от `StandaloneCoroutine` и в конечном итоге от `AbstractCoroutine`.

```kotlin
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}
```















