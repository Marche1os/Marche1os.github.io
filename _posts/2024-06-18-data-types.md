**Про типы данных и value классы**

Внедрил в проекте практику использования такого типа классов в Котлине, как value класс.
Такой тип класса специально создан для создания конкретного типа данных в проекте. 
Теперь на законных основаниях можно требовать на пулл-реквестах замены примитивных типов на конкретный тип данных. 

Выступил на команду с докладом, в котором охватил следующие темы:
- История типов данных в программировании; 
- value классы в Котлине и как их использовать в контексте создания конкретных типов данных; 
- возможности, которые мы получаем, задавая конкретный тип.  

В этой теме рассмотрим эти темы.

### История типов в программировании

Знание истории развития типов данных и то, какими были типы на заре программирования, позволяет взглянуть на проблему
шире и понять, что применение value классов является не каким-то стилем программирования, на который нас хотят
подтолкнуть разработчики языка или какие-то люди, выступающие с докладом. Нет, применение value классов
по-сути является следующим шагом в эволюции типов, который ведет нас к точности и безопасности типов данных.

[История типов. Параграф 1.3](http://www.k-press.ru/cs/2006/2/onunderstand/OnUnderstand.asp)


### value классы в Котлине 

Такой класс создать легко, вот как он выглядит в коде:
```kotlin
@JvmInline
value class MyValueClass(val value: String)
```

Одна из главных особенностей таких классов в том, что в Runtime происходит unboxing. 
Тип-обертка уходит, в код вызова подставляется обернутое значение.
Стоит отметить, что такое возможно не всегда и в целом нельзя прямо на 100% гарантировать unboxing. 
Например, value классы могут реализовывать интерфейсы и в таком случае никакого unboxing в Runtime не произойдет, наш value класс будет обычным классом.

Ниже перечислил основные моменты, касающиеся value классов: 
1. Можно сравнивать между собой по значение, но не по ссылке
2. На данный момент должны содержать только 1 параметр. Возможно, в будущих версиях языка это изменится
3. Инлайнятся в места вызова, когда это возможно 
4. Могут быть Serializable/Parcelable 
5. Можно использовать как параметр в DTO 
6. Могут содержать функции внутри себя. Такие функции компилируются в статичные методы, которые принимают инстанс value класса. 
7. Могут реализовывать интерфейсы. От классов наследоваться нельзя. В таком случае мы точно теряем инлайнинг. 
8. Могут содержать properties, но не переменные класса.
9. Вызов toString по умолчанию возвращает сигнатуру класса 
10. Могут быть Generic типом

### Возможности использования конкретного типа данных 

Создав конкретный тип данных, мы сразу получаем следующие возможности, которыми стоит воспользоваться:
- Документирование типа;
- задание области допустимых значений;
- compile-time type check;
- мышление на уровне данных.

### 1. Документирование типа данных

Вместе с созданием своего типа (через value классы, data классы - неважно) мы сразу получаем возможность описать, задокументировать тип данных, задать ограничения, указать область допустимых значений, связи - все что угодно. 

Кроме того упрощается навигация между файлами в проекте, по типу мы можем определить место инициализации.

Пример:

```kotlin
/**
 * Представляет собой имя [User] в системе
 * 
 * Должен удовлетворять условиям:
 * - Длина не должна быть короче 2 символов
 * - Формат строки: первый символ в upperCase, остальные в lowerCase
 */
@JvmInline
value class Username(val value: String)

data class User(
  val username: Username,
)
```
### 2. Область допустимых значений

Также в момент инициализации типа данных мы можем накладывать ограничение на его множество значений. 
Это позволяет быть уверенным при работе с типом в проекте.
Во-первых, уходит необходимость проваливаться по местам вызовов, чтобы отследить, откуда приходит значение, где оно инициализируется, с каким значением инициализируется и так далее. 
Достаточно взглянуть на тип, что узнать о нем все. 
Также уходит необходимость в проверках на то, что тип данных содержит удовлетворяющие нас значения, что ведет к уменьшению проверок. 
Например, на пустоту. А значит и к уменьшению цикломатической сложности кода. 
И вдобавок уходит необходимость проверять типы в тестах, сами тесты становятся легче в тех случаях, когда нужно проверять, что параметры одного типа не перепутаны местами.

Пример 1:

```kotlin
@JvmInline
value class Password private constructor(val value: String) {
  init {
      require(passwordIsStrong(value))
  }
  
  companion object Factory {
    fun create(password: String): Password = TODO("implementation")
  }
}
```

Пример 2:

```kotlin
/**
 * Громкость колонки
 */
@JvmInline
value class SpeakerVolume private constructor(val value: Int) {
  init {
      require(value in MIN_LEVEL..MAX_LEVEL)
  }
  
  companion object Factory {
    private const val MIN_LEVEL = 0
    private const val MAX_LEVEL = 10
    
    fun create(volume: Int): Result<SpeakerVolume> = TODO("implementation")
  }
}
```


### 3. Compile-time type check

Следующее, что мы получаем - это проверку соответствия типов на этапе компиляции. 
Здесь все просто, мы не сможем передать параметр одного типа, если функция ожидает параметр другого типа.
Или не сможем работать с одним типом, как если бы это был другой тип.

Популярные проблемы "перепутывания" параметров одного типа будут решены.

```kotlin
@JvmInline
value class Id(val value: String)

@JvmInline
value class StudentId(val value: String)

@JvmInline
value class ProductId(val value: String)

fun main() {
    val someId = Id("someId")
    val studentId = StudentId("studentId")
    val productId = ProductId("productId")
    
    // Получим на этапе компиляции: Type mismatch: inferred type is ProductId but StudentId was expected
    someFun(someId, productId, studentId)
}

fun someFun(id: Id, studentId: StudentId, productId: ProductId) {
    // no-op
}
```

### 4. Мышление на уровне данных

В целом, это довольно размытая вещь, пока не перейдешь на проработанную систему типов в проекте,
На уровне идеи ожидается следующее: видя данные, мы видим связь этих данных с другими сущностями, 
что позволит выйти на новый уровень рассуждения о программе, на рассуждение в терминах данных и связей между данными.

Погружение в новую предметную область для разработчиков будет более эффективным, будет снижено количество логических ошибок, которые часто возникают из-за отсутствия глубокого понимания предметной области.

Кроме этого, имея конкретные типы данных предметной области мы можем более эффективно создавать DSL и задавать допустимые операции над этим типом.