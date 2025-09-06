[Вопросы для собеседования](README.md)

# Шаблоны проектирования
+ [Что такое _«шаблон проектирования»_?](#Что-такое-шаблон-проектирования)
+ [Назовите основные характеристики шаблонов.](#Назовите-основные-характеристики-шаблонов)
+ [Типы шаблонов проектирования.](#Типы-шаблонов-проектирования)
+ [Приведите примеры основных шаблонов проектирования.](#Приведите-примеры-основных-шаблонов-проектирования)
+ [Приведите примеры порождающих шаблонов проектирования.](#Приведите-примеры-порождающих-шаблонов-проектирования)
+ [Приведите примеры структурных шаблонов проектирования.](#Приведите-примеры-структурных-шаблонов-проектирования)
+ [Приведите примеры поведенческих шаблонов проектирования.](#Приведите-примеры-поведенческих-шаблонов-проектирования)
+ [Что такое _«антипаттерн»_? Какие антипаттерны вы знаете?](#Что-такое-антипаттерн-Какие-антипаттерны-вы-знаете)
+ [Что такое _Dependency Injection_?](#Что-такое-dependency-injection)

## Что такое _«шаблон проектирования»_?
__Шаблон (паттерн) проектирования (design pattern)__ — это проверенное и готовое к использованию решение. Это не класс и не библиотека, которую можно подключить к проекту, это нечто большее - он не зависит от языка программирования, не является законченным образцом, который может быть прямо преобразован в код и может быть реализован по-разному в разных языках программирования.

Плюсы использования шаблонов:
+ снижение сложности разработки за счёт готовых абстракций для решения целого класса проблем.
+ облегчение коммуникации между разработчиками, позволяя ссылаться на известные шаблоны.
+ унификация деталей решений: модулей и элементов проекта.
+ возможность отыскав удачное решение, пользоваться им снова и снова.
+ помощь в выборе выбрать наиболее подходящего варианта проектирования.

Минусы:
+ слепое следование некоторому выбранному шаблону может привести к усложнению программы.
+ желание попробовать некоторый шаблон в деле без особых на то оснований.

[к оглавлению](#Шаблоны-проектирования)

## Назовите основные характеристики шаблонов.
+ __Имя__ - все шаблоны имеют уникальное имя, служащее для их идентификации;
+ __Назначение__	назначение данного шаблона;
+ __Задача__ - задача, которую шаблон позволяет решить;
+ __Способ решения__ - способ, предлагаемый в шаблоне для решения задачи в том контексте, где этот шаблон был найден;
+ __Участники__	- сущности, принимающие участие в решении задачи;
+ __Следствия__	- последствия от использования шаблона как результат действий, выполняемых в шаблоне;
+ __Реализация__ - возможный вариант реализации шаблона.

[к оглавлению](#Шаблоны-проектирования)

## Типы шаблонов проектирования.
+ Основные (Fundamental) - основные строительные блоки других шаблонов. Большинство других шаблонов использует эти шаблоны в той или иной форме.
+ Порождающие шаблоны (Creational) — шаблоны проектирования, которые абстрагируют процесс создание экземпляра. Они позволяют сделать систему независимой от способа создания, композиции и представления объектов. Шаблон, порождающий классы, использует наследование, чтобы изменять созданный объект, а шаблон, порождающий объекты, делегирует создание объектов другому объекту.
+ Структурные шаблоны (Structural) определяют различные сложные структуры, которые изменяют интерфейс уже существующих объектов или его реализацию, позволяя облегчить разработку и оптимизировать программу.
+ Поведенческие шаблоны (Behavioral) определяют взаимодействие между объектами, увеличивая таким образом его гибкость.

[к оглавлению](#Шаблоны-проектирования)

## Приведите примеры основных шаблонов проектирования.
+ __Делегирование (Delegation pattern)__ - Сущность внешне выражает некоторое поведение, но в реальности передаёт ответственность за выполнение этого поведения связанному объекту.
+ __Функциональный дизайн (Functional design)__ - Гарантирует, что каждая сущность имеет только одну обязанность и исполняет её с минимумом побочных эффектов на другие.
+ __Неизменяемый интерфейс (Immutable interface)__ - Создание неизменяемого объекта.
+ __Интерфейс (Interface)__ - Общий метод структурирования сущностей, облегчающий их понимание. 
+ __Интерфейс-маркер (Marker interface)__ - В качестве атрибута (как пометки объектной сущности) применяется наличие или отсутствие реализации интерфейса-маркера. В современных языках программирования вместо этого применяются атрибуты или аннотации.
+ __Контейнер свойств (Property container)__ - Позволяет добавлять дополнительные свойства сущности в контейнер внутри себя, вместо расширения новыми свойствами.
+ __Канал событий (Event channel)__ - Создаёт централизованный канал для событий. Использует сущность-представитель для подписки и сущность-представитель для публикации события в канале. Представитель существует отдельно от реального издателя или подписчика. Подписчик может получать опубликованные события от более чем одной сущности, даже если он зарегистрирован только на одном канале.

[к оглавлению](#Шаблоны-проектирования)

## Приведите примеры порождающих шаблонов проектирования.
+ __Абстрактная фабрика (Abstract factory)__ - Класс, который представляет собой интерфейс для создания других классов.
+ __Строитель (Builder)__ - Класс, который представляет собой интерфейс для создания сложного объекта.
+ __Фабричный метод (Factory method)__ - Делегирует создание объектов наследникам родительского класса. Это позволяет использовать в коде программы не специфические классы, а манипулировать абстрактными объектами на более высоком уровне.
+ __Прототип (Prototype)__ - Определяет интерфейс создания объекта через клонирование другого объекта вместо создания через конструктор.
+ __Одиночка (Singleton)__ - Класс, который может иметь только один экземпляр.

[к оглавлению](#Шаблоны-проектирования)

## Приведите примеры структурных шаблонов проектирования.
+ __Адаптер (Adapter)__ - Объект, обеспечивающий взаимодействие двух других объектов, один из которых использует, а другой предоставляет несовместимый с первым интерфейс. 
+ __Мост (Bridge)__ - Структура, позволяющая изменять интерфейс обращения и интерфейс реализации класса независимо. 
+ __Компоновщик (Composite)__ - Объект, который объединяет в себе объекты, подобные ему самому. 
+ __Декоратор (Decorator)__ - Класс, расширяющий функциональность другого класса без использования наследования. 
+ __Фасад (Facade)__ - Объект, который абстрагирует работу с несколькими классами, объединяя их в единое целое. 
+ __Приспособленец (Flyweight)__ - Это объект, представляющий себя как уникальный экземпляр в разных местах программы, но по факту не являющийся таковым. 
+ __Заместитель (Proxy)__ - Объект, который является посредником между двумя другими объектами, и который реализует/ограничивает доступ к объекту, к которому обращаются через него.

[к оглавлению](#Шаблоны-проектирования)

## Приведите примеры поведенческих шаблонов проектирования.
+ __Цепочка обязанностей (Chain of responsibility)__ - Предназначен для организации в системе уровней ответственности.
+ __Команда (Command)__ - Представляет действие. Объект команды заключает в себе само действие и его параметры.
+ __Интерпретатор (Interpreter)__ - Решает часто встречающуюся, но подверженную изменениям, задачу.
+ __Итератор (Iterator)__ - Представляет собой объект, позволяющий получить последовательный доступ к элементам объекта-агрегата без использования описаний каждого + __из объектов, входящих в состав агрегации.
+ __Посредник (Mediator)__ - Обеспечивает взаимодействие множества объектов, формируя при этом слабую связанность и избавляя объекты от необходимости явно ссылаться друг на друга.
+ __Хранитель (Memento)__ - Позволяет, не нарушая инкапсуляцию зафиксировать и сохранить внутренние состояния объекта так, чтобы позднее восстановить его в этих состояниях.
+ __Наблюдатель (Observer)__ - Определяет зависимость типа «один ко многим» между объектами таким образом, что при изменении состояния одного объекта все зависящие от него оповещаются об этом событии.
+ __Состояние (State)__ - Используется в тех случаях, когда во время выполнения программы объект должен менять своё поведение в зависимости от своего состояния.
+ __Стратегия (Strategy)__ - Предназначен для определения семейства алгоритмов, инкапсуляции каждого из них и обеспечения их взаимозаменяемости.
+ __Шаблонный метод (Template method)__ - Определяет основу алгоритма и позволяет наследникам переопределять некоторые шаги алгоритма, не изменяя его структуру в целом.
+ __Посетитель (Visitor)__ - Описывает операцию, которая выполняется над объектами других классов. При изменении класса Visitor нет необходимости изменять обслуживаемые классы.

[к оглавлению](#Шаблоны-проектирования)

## Что такое _«антипаттерн»_? Какие антипаттерны вы знаете?
__Антипаттерн (anti-pattern)__ — это распространённый подход к решению класса часто встречающихся проблем, являющийся неэффективным, рискованным или непродуктивным.

__Poltergeists (полтергейсты)__ - это классы с ограниченной ответственностью и ролью в системе, чьё единственное предназначение — передавать информацию в другие классы. Их эффективный жизненный цикл непродолжителен. Полтергейсты нарушают стройность архитектуры программного обеспечения, создавая избыточные (лишние) абстракции, они чрезмерно запутанны, сложны для понимания и трудны в сопровождении. Обычно такие классы задумываются как классы-контроллеры, которые существуют только для вызова методов других классов, зачастую в предопределенной последовательности.

Признаки появления и последствия антипаттерна
+ Избыточные межклассовые связи.
+ Временные ассоциации.
+ Классы без состояния (содержащие только методы и константы).
+ Временные объекты и классы (с непродолжительным временем жизни).
+ Классы с единственным методом, который предназначен только для создания или вызова других классов посредством временной ассоциации.
+ Классы с именами методов в стиле «управления», такие как startProcess.

Типичные причины
+ Отсутствие объектно-ориентированной архитектуры (архитектор не понимает объектно-ориентированной парадигмы).
+ Неправильный выбор пути решения задачи.
+ Предположения об архитектуре приложения на этапе анализа требований (до объектно-ориентированного анализа) могут также вести к проблемам на подобии этого антипаттерна.

__Внесенная сложность (Introduced complexity)__: Необязательная сложность дизайна. Вместо одного простого класса выстраивается целая иерархия интерфейсов и классов. Типичный пример «Интерфейс - Абстрактный класс - Единственный класс реализующий интерфейс на основе абстрактного».

__Инверсия абстракции (Abstraction inversion)__: Сокрытие части функциональности от внешнего использования, в надежде на то, что никто не будет его использовать.

__Неопределённая точка зрения (Ambiguous viewpoint)__: Представление модели без спецификации её точки рассмотрения.

__Большой комок грязи (Big ball of mud)__: Система с нераспознаваемой структурой.

__Божественный объект (God object)__: Концентрация слишком большого количества функций в одной части системы (классе).

__Затычка на ввод данных (Input kludge)__: Забывчивость в спецификации и выполнении поддержки возможного неверного ввода.

__Раздувание интерфейса (Interface bloat)__: Разработка интерфейса очень мощным и очень сложным для реализации.

__Волшебная кнопка (Magic pushbutton)__: Выполнение результатов действий пользователя в виде неподходящего (недостаточно абстрактного) интерфейса. Например, написание прикладной логики в обработчиках нажатий на кнопку.

__Перестыковка (Re-Coupling)__: Процесс внедрения ненужной зависимости.

__Дымоход (Stovepipe System)__: Редко поддерживаемая сборка плохо связанных компонентов.

__Состояние гонки (Race hazard)__: непредвидение возможности наступления событий в порядке, отличном от ожидаемого.

__Членовредительство (Mutilation)__: Излишнее «затачивание» объекта под определенную очень узкую задачу таким образом, что он не способен будет работать с никакими иными, пусть и очень схожими задачами.

__Сохранение или смерть (Save or die)__: Сохранение изменений лишь при завершении приложения.

[к оглавлению](#Шаблоны-проектирования)

## Что такое _Dependency Injection_?
__Dependency Injection (внедрение зависимости)__ - это набор паттернов и принципов разработки програмного обеспечения, которые позволяют писать слабосвязный код. В полном соответствии с принципом единой обязанности объект отдаёт заботу о построении требуемых ему зависимостей внешнему, специально предназначенному для этого общему механизму.

[к оглавлению](#Шаблоны-проектирования)
# Введение — что такое **порождающие (creational)** паттерны
Порождающие паттерны проектирования решают, **как правильно создавать объекты**. Они помогают отделить логику создания от использования, делают код гибче, проще для тестирования и расширения. На собеседовании от вас ожидают не только определения, но и понимание **когда** и **почему** применять каждый паттерн, плюсы/минусы и практические примеры.

Ниже — подробные объяснения (простым языком), реальные сценарии и готовые Java-примеры с подробными комментариями на русском. После каждого блока — полностью рабочий пример в одном файле (один public класс + вспомогательные), чтобы вы могли скопировать и вставить в IDE.

---

# 1. Фабрика (Simple Factory) и Фабричный метод (Factory Method)
### Идея (простыми словами)
- **Simple Factory** — это просто класс/метод, который создаёт объекты разных типов по входным параметрам. Неофициальный паттерн (не в GOF), но часто используется.
- **Factory Method** — переопределяемый метод (обычно в абстрактном классе или интерфейсе), который делегирует создание конкретных объектов подклассам. Это официальный GOF-паттерн.

### Когда и зачем использовать
- Нужна централизованная логика создания объектов (когда создание — не тривиально).
- Когда нужно скрыть конкретные классы от клиента (инверсия зависимостей).
- Когда требуется расширяемость: добавление нового продукта не должно ломать код клиента.
- Simple Factory — подходит для небольших случаев; Factory Method — при потребности в расширяемости через наследование.

### Плюсы / минусы
+ Снижает связанность кода (client не знает конкретных классов).  
+ Упрощает замену/добавление новых типов.  
− Может привести к большому числу классов (особенно Factory Method).  
− Simple Factory может стать «God class», если он слишком большой.

### Пример: Factory Method — транспорт (road/ship/air)
```java
// FactoryMethodDemo.java
// Простой демонстрационный пример паттерна Factory Method.
// Все классы находятся в одном файле для удобства копирования в IDE.
// Public-класс соответствует имени файла.

// Добавлены аннотации Lombok для демонстрации их использования (хотя в простых классах Lombok не обязателен).
import lombok.NoArgsConstructor;
import lombok.ToString;

public class FactoryMethodDemo {
    public static void main(String[] args) {
        // Клиент работает с создателем, не зная конкретного транспорта.
        Creator roadCreator = new RoadLogistics();
        Transport t1 = roadCreator.createTransport();
        t1.deliver();

        Creator seaCreator = new SeaLogistics();
        Transport t2 = seaCreator.createTransport();
        t2.deliver();

        // Добавление нового транспорта = добавление нового класса Creator + Product,
        // клиентский код не изменяется.
    }
}

// --- Product интерфейс ---
interface Transport {
    void deliver(); // метод для доставки
}

// --- Concrete Products ---
@NoArgsConstructor
@ToString
class Truck implements Transport {
    @Override
    public void deliver() {
        // Конкретная реализация доставки для грузовика
        System.out.println("Доставка грузовиком по дороге");
    }
}

@NoArgsConstructor
@ToString
class Ship implements Transport {
    @Override
    public void deliver() {
        System.out.println("Доставка кораблем по морю");
    }
}

// --- Creator (абстрактный) ---
abstract class Creator {
    // Фабричный метод — возвращает интерфейс Transport.
    // Подклассы решают, какой конкретно транспорт создать.
    public abstract Transport createTransport();

    // Допустим у Creator есть общая логика:
    public void planDelivery() {
        Transport transport = createTransport();
        // общая логика использования продукта
        System.out.println("Планирование доставки...");
        transport.deliver();
    }
}

// --- Concrete Creators ---
@NoArgsConstructor
class RoadLogistics extends Creator {
    @Override
    public Transport createTransport() {
        // можно добавить логику выбора конфигурации Truck
        return new Truck();
    }
}

@NoArgsConstructor
class SeaLogistics extends Creator {
    @Override
    public Transport createTransport() {
        return new Ship();
    }
}
```

---

# 2. Абстрактная фабрика (Abstract Factory)
### Идея (простыми словами)
Абстрактная фабрика предоставляет интерфейс для создания **семейств связанных объектов** (например, виджеты для разных платформ), не указывая их конкретных классов. Это удобно, когда объекты должны работать вместе (совместимы по интерфейсу/стилю).

### Когда и зачем использовать
- Когда нужно создать семейства взаимосвязанных объектов (например, темы UI — кнопка + чекбокс + поле ввода).
- Когда важно, чтобы продукты одного семейства были совместимы.
- Когда нужно менять семейство продуктов во время выполнения (поддержка тем/платформ).

### Плюсы / минусы
+ Хорошо обеспечивает единообразие продуктов (семейства совместимы).  
+ Упрощает замену семейства продуктов целиком.  
− Усложняет добавление новых типов продуктов (добавая новый продукт — нужно изменить интерфейсы/все фабрики).

### Пример: семейства виджетов (Windows / Mac)
```java
// AbstractFactoryDemo.java
// Демонстрация Abstract Factory на примере UI-виджетов (Button + Checkbox).
// Все классы в одном файле для удобного копирования.

// Добавлены аннотации Lombok (@ToString) для демонстрации, где это уместно.
import lombok.ToString;

public class AbstractFactoryDemo {
    public static void main(String[] args) {
        UIFactory factory = new WindowsFactory();
        Button winButton = factory.createButton();
        Checkbox winCheckbox = factory.createCheckbox();
        winButton.paint();
        winCheckbox.paint();

        // Теперь сменим семью на Mac
        UIFactory macFactory = new MacFactory();
        macFactory.createButton().paint();
        macFactory.createCheckbox().paint();
    }
}

// --- Abstract Products ---
interface Button {
    void paint();
}

interface Checkbox {
    void paint();
}

// --- Concrete Products for Windows ---
@ToString
class WindowsButton implements Button {
    @Override
    public void paint() {
        System.out.println("Рисуем кнопку в стиле Windows");
    }
}

@ToString
class WindowsCheckbox implements Checkbox {
    @Override
    public void paint() {
        System.out.println("Рисуем чекбокс в стиле Windows");
    }
}

// --- Concrete Products for Mac ---
@ToString
class MacButton implements Button {
    @Override
    public void paint() {
        System.out.println("Рисуем кнопку в стиле Mac");
    }
}

@ToString
class MacCheckbox implements Checkbox {
    @Override
    public void paint() {
        System.out.println("Рисуем чекбокс в стиле Mac");
    }
}

// --- Abstract Factory ---
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// --- Concrete Factories ---
class WindowsFactory implements UIFactory {
    @Override
    public Button createButton() { return new WindowsButton(); }
    @Override
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

class MacFactory implements UIFactory {
    @Override
    public Button createButton() { return new MacButton(); }
    @Override
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}
```

---

# 3. Builder (Строитель)
### Идея (простыми словами)
Builder отделяет процесс построения сложного объекта от его представления. Особенно полезен, когда конструктор класса принимает много параметров (особенно опциональных), и хочется избежать "конструкторского ада" (много конструкторов с разными комбинациями параметров).

### Когда и зачем использовать
- Объект имеет много полей (часто опциональных).
- Нужна читабельная и безопасная конструкция (иммутабельность).
- Требуется пошаговая конфигурация и валидация перед созданием.
- Полезен в DSL-подходе (цепочки вызовов).

### Плюсы / минусы
+ Улучшает читаемость кода при создании сложных объектов.  
+ Позволяет делать объект immutable.  
− Небольшой дополнительный код (класс Builder), но обычно оправдан.

### Пример: построение объекта `User` с множеством опциональных полей
```java
// BuilderDemo.java
// Пример паттерна Builder на классе User.
// Внутри статический вложенный Builder: класс User становится иммутабельным.

// Здесь заменяем ручной Builder на Lombok @Builder для сокращения кода.
// Lombok сгенерирует класс Builder автоматически, а также геттеры и toString.
import lombok.Builder;
import lombok.Getter;
import lombok.ToString;

public class BuilderDemo {
    public static void main(String[] args) {
        // Пример использования билдера.
        User user = User.builder()
                .email("ivan@example.com") // обязательное поле
                .firstName("Иван")
                .lastName("Иванов")
                .age(35)
                .phone("+7-900-123-45-67")
                .build();

        System.out.println(user);
    }
}

@Getter
@ToString
@Builder
final class User {
    // обязательное
    private final String email;
    // опциональные
    private final String firstName;
    private final String lastName;
    private final Integer age;
    private final String phone;
}
```

---

# 4. Singleton (Одиночка)
### Идея (простыми словами)
Паттерн обеспечивает наличие **только одного экземпляра** класса и глобальную точку доступа к нему.

### Когда и зачем использовать
- Нужно глобальное, совместно используемое состояние (например, конфигурация приложения, пул соединений, логгер).  
- Важно, чтобы экземпляр был один в рамках JVM.

### Важно на интервью
- Обсуждают проблемы многопоточности и сериализации.
- Покажите разные реализации (eager, lazy с double-check, Enum) и объясните плюсы/минусы.

### Плюсы / минусы
+ Удобен для глобального состояния.  
− Может стать глобальной переменной (anti-pattern) — усложняет тестирование (мокирование).  
− Ошибки при сериализации, клонировании и многопоточности — нужно быть аккуратным.

### Примеры: Eager, Lazy (Double-checked), Enum
```java
// SingletonDemo.java
// Три варианта реализации Singleton — в одном файле для демонстрации.
// В реальном проекте выберите один подход (часто рекомендуют enum-одиночку).

// Lombok можно использовать для генерации геттеров/логирования, например @Getter или @Slf4j.
// Здесь добавим @Getter в качестве примера.
import lombok.Getter;

public class SingletonDemo {
    public static void main(String[] args) {
        System.out.println("Eager: " + EagerSingleton.getInstance());
        System.out.println("LazyDCL: " + LazyDCLSingleton.getInstance());
        System.out.println("Enum: " + EnumSingleton.INSTANCE);

        // Проверка, что ссылки одинаковы
        System.out.println("Eager same? " + (EagerSingleton.getInstance() == EagerSingleton.getInstance()));
        System.out.println("LazyDCL same? " + (LazyDCLSingleton.getInstance() == LazyDCLSingleton.getInstance()));
        System.out.println("Enum same? " + (EnumSingleton.INSTANCE == EnumSingleton.INSTANCE));
    }
}

// --- Eager initialization (простая и потокобезопасная при загрузке класса) ---
@Getter
class EagerSingleton {
    // Экземпляр создаётся при загрузке класса
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    // Приватный конструктор — нельзя создать извне
    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}

// --- Lazy initialization с double-checked locking (DCL) ---
@Getter
class LazyDCLSingleton {
    // volatile важен для корректности double-checked locking
    private static volatile LazyDCLSingleton instance;

    private LazyDCLSingleton() {}

    public static LazyDCLSingleton getInstance() {
        // Сначала без синхронизации (быстрая проверка)
        if (instance == null) {
            synchronized (LazyDCLSingleton.class) {
                // Проверяем ещё раз внутри synchronized
                if (instance == null) {
                    instance = new LazyDCLSingleton();
                }
            }
        }
        return instance;
    }
}

// --- Enum singleton (лучший вариант в большинстве случаев) ---
enum EnumSingleton {
    INSTANCE; // единственный экземпляр

    // Можно хранить состояние и методы
    public void doSomething() {
        System.out.println("EnumSingleton doing something");
    }
}
```

**Рекомендация:** в большинстве случаев `enum`-singleton — самый безопасный и простой способ, потому что он автоматически защищён от проблем с сериализацией и отражением (reflection).

---

# 5. Prototype (Прототип)
### Идея (простыми словами)
Prototype — создаём новые объекты путём клонирования (копирования) существующего экземпляра-прототипа. Удобно, когда создание объекта «дорогое» (сложная инициализация), либо когда нужно быстро получить копию с небольшими изменениями.

### Когда и зачем использовать
- Стоимость создания нового объекта высокая (ресурсоёмкая инициализация).
- Нужно много схожих объектов, которые отличаются небольшими деталями.
- Желание получать копии без знания точного конкретного класса (через интерфейс clone).

### Важные нюансы в Java
- `Cloneable` и метод `clone()` — исторически спорны (поведение дефолтного `Object.clone()` делает поверхностное копирование).
- Для глубокого копирования часто нужно вручную клонировать вложенные объекты или использовать сериализацию/копирующие конструкторы.
- На интервью можно показать и `clone()` и альтернативный способ — копирующий конструктор/фабрика копий.

### Пример: shallow & deep clone
```java
// PrototypeDemo.java
// Демонстрация shallow и deep clone на объекте Document, содержащем Metadata.
// Используем Cloneable для примера, и также покажем копирующий конструктор для deep copy.

// Lombok используется для сокращения шаблонного кода (геттеры/сеттеры/toString)
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

public class PrototypeDemo {
    public static void main(String[] args) throws CloneNotSupportedException {
        Metadata meta = new Metadata("Иван", "2025-09-06");
        Document original = new Document("Отчёт", "Тело отчёта", meta);

        // shallow clone (через clone) — metadata будет общим (shared reference)
        Document shallow = original.clone();
        System.out.println("original metadata author: " + original.getMetadata().getAuthor());
        System.out.println("shallow metadata author:  " + shallow.getMetadata().getAuthor());

        // Изменим метаданные в shallow
        shallow.getMetadata().setAuthor("Пётр");
        System.out.println("После изменения shallow:");
        System.out.println("original metadata author: " + original.getMetadata().getAuthor()); // изменилось — демонстрация shallow

        // Deep clone через копирующий конструктор
        Document deep = new Document(original); // copy constructor
        deep.getMetadata().setAuthor("Мария");
        System.out.println("После изменения deep:");
        System.out.println("original metadata author: " + original.getMetadata().getAuthor()); // не изменилось
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
class Metadata implements Cloneable {
    private String author;
    private String date;

    @Override
    protected Metadata clone() throws CloneNotSupportedException {
        // Для метаданных shallow clone достаточно (если все поля примитивы/immutable).
        return (Metadata) super.clone();
    }
}

@Data
class Document implements Cloneable {
    private String title;
    private String body;
    private Metadata metadata;

    public Document(String title, String body, Metadata metadata) {
        this.title = title;
        this.body = body;
        this.metadata = metadata;
    }

    // Копирующий конструктор для deep copy
    public Document(Document other) {
        this.title = other.title;
        this.body = other.body;
        // Для глубокого копирования создаём новый объект Metadata (который клонируем)
        try {
            this.metadata = other.metadata.clone();
        } catch (CloneNotSupportedException e) {
            // fallback — создать вручную
            this.metadata = new Metadata(other.metadata.getAuthor(), other.metadata.getDate());
        }
    }

    @Override
    protected Document clone() throws CloneNotSupportedException {
        // По умолчанию суперкласс делает поверхностное копирование
        return (Document) super.clone();
    }

    // геттер для metadata
    public Metadata getMetadata() { return metadata; }
}
```

**Практический совет:** на интервью можно сказать, что `Cloneable` в Java спорен, и предпочитаете контролируемое копирование (копирующие конструкторы / фабрики), особенно для глубокого копирования.

---

# Заключение — как выбирать паттерн
- Нужен один глобальный экземпляр → **Singleton** (с осторожностью; лучше enum).
- Нужна простая централизация создания → **Simple Factory**.
- Требуется расширяемость создания через подклассы → **Factory Method**.
- Нужна гибкая смена семейства совместимых продуктов → **Abstract Factory**.
- Объект создаётся с большим количеством опций / хотите immutable → **Builder**.
- Нужно клонирование/быстро создавать похожие объекты / избежать дорогой инициализации → **Prototype** (предпочтительно через контролируемые копии).

---

# Структурные паттерны проектирования (с подробным объяснением и примерами на Java + Lombok)

Структурные паттерны помогают **организовать классы и объекты в более крупные структуры**, сохраняя при этом гибкость.  
Они фокусируются на том, **как связать классы и объекты между собой**.

---

# 1. Adapter (Адаптер)
### Идея (простыми словами)
Адаптер работает как **переходник между несовместимыми интерфейсами**. Если у нас есть существующий класс с одним интерфейсом, а клиент ожидает другой, адаптер "переводит" вызовы.

### Когда использовать
- Есть готовый класс, но его интерфейс не подходит клиентскому коду.
- Не хочется или нельзя менять существующий класс.
- Нужно использовать стороннюю библиотеку с неудобным интерфейсом.

### Плюсы / минусы
+ Позволяет переиспользовать существующие классы без изменения их кода.  
+ Изолирует изменения интерфейсов.  
− Может привести к избыточному числу классов (каждый адаптер отдельно).  
− Иногда ухудшает читаемость из-за "лишнего уровня".

### Пример: `SquarePeg` и `RoundHole`
```java
// AdapterDemo.java
// Пример: у нас есть КруглоеОтверстие (RoundHole) и КруглыйКолышек (RoundPeg).
// Но появляется КвадратныйКолышек (SquarePeg), который не подходит.
// Используем адаптер, чтобы заставить квадратный колышек работать с круглым отверстием.

import lombok.AllArgsConstructor;
import lombok.Getter;

public class AdapterDemo {
    public static void main(String[] args) {
        RoundHole hole = new RoundHole(5);
        RoundPeg roundPeg = new RoundPeg(5);
        System.out.println("Круглый колышек помещается? " + hole.fits(roundPeg));

        SquarePeg smallSqPeg = new SquarePeg(5);
        SquarePeg largeSqPeg = new SquarePeg(10);

        // Адаптируем квадратные колышки
        RoundPeg smallAdapter = new SquarePegAdapter(smallSqPeg);
        RoundPeg largeAdapter = new SquarePegAdapter(largeSqPeg);

        System.out.println("Маленький квадратный колышек помещается? " + hole.fits(smallAdapter));
        System.out.println("Большой квадратный колышек помещается? " + hole.fits(largeAdapter));
    }
}

@Getter
@AllArgsConstructor
class RoundHole {
    private double radius;

    public boolean fits(RoundPeg peg) {
        return this.radius >= peg.getRadius();
    }
}

@Getter
@AllArgsConstructor
class RoundPeg {
    private double radius;
}

@Getter
@AllArgsConstructor
class SquarePeg {
    private double width;
}

// Адаптер: заставляет SquarePeg работать как RoundPeg
@AllArgsConstructor
class SquarePegAdapter extends RoundPeg {
    private SquarePeg peg;

    public double getRadius() {
        // Диагональ квадрата / 2 = радиус вписанной окружности
        return (peg.getWidth() * Math.sqrt(2)) / 2;
    }
}
```

---

# 2. Composite (Компоновщик)
### Идея
Позволяет работать с **отдельными объектами и их составами одинаково**.  
Используется для представления древовидных структур (например, файловая система).

### Когда использовать
- Когда объекты образуют иерархию "часть-целое".
- Когда нужно, чтобы клиент одинаково работал как с одиночными объектами, так и с их комбинациями.

### Плюсы / минусы
+ Унифицированный способ работы с отдельными объектами и группами.  
+ Легко строить сложные иерархии.  
− Может быть сложно ограничить доступные операции для "листьев" и "композитов".  
− Может быть трудно контролировать структуру (например, запретить пустые папки).

### Пример: файловая система (Файл и Папка)
```java
// CompositeDemo.java
// Демонстрация: Папка содержит как файлы, так и другие папки.
// Оба реализуют общий интерфейс FileSystemComponent.

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.ToString;

import java.util.ArrayList;
import java.util.List;

public class CompositeDemo {
    public static void main(String[] args) {
        File file1 = new File("file1.txt");
        File file2 = new File("file2.txt");
        Directory dir = new Directory("docs");
        dir.add(file1);
        dir.add(file2);

        Directory root = new Directory("root");
        root.add(dir);
        root.add(new File("readme.md"));

        root.show();
    }
}

interface FileSystemComponent {
    void show();
}

@Getter
@AllArgsConstructor
@ToString
class File implements FileSystemComponent {
    private String name;

    @Override
    public void show() {
        System.out.println("Файл: " + name);
    }
}

@Getter
@AllArgsConstructor
class Directory implements FileSystemComponent {
    private String name;
    private List<FileSystemComponent> children = new ArrayList<>();

    public Directory(String name) {
        this.name = name;
    }

    public void add(FileSystemComponent component) {
        children.add(component);
    }

    @Override
    public void show() {
        System.out.println("Папка: " + name);
        for (FileSystemComponent child : children) {
            child.show();
        }
    }
}
```

---

# 3. Proxy (Заместитель)
### Идея
Proxy выступает как "заместитель" объекта, контролируя доступ к нему.  
Он может добавлять к основному объекту **ленивую инициализацию, безопасность, кэширование, логирование**.

### Когда использовать
- Ленивое создание "тяжёлого" объекта (виртуальный прокси).
- Добавление доступа/логирования без изменения реального объекта.
- Удалённые прокси (работа через сеть).

### Плюсы / минусы
+ Позволяет контролировать доступ и добавлять функциональность.  
+ Поддержка ленивой загрузки.  
− Усложняет код (ещё один уровень абстракции).  
− Может быть лишним, если дополнительных функций нет.

### Пример: ленивое создание "тяжёлого" объекта
```java
// ProxyDemo.java
// Пример: загружаем "тяжёлое" изображение только тогда, когда оно реально нужно.

public class ProxyDemo {
    public static void main(String[] args) {
        Image image = new ProxyImage("photo.jpg");

        // Файл ещё не загружен
        image.display();

        // Повторный вызов: файл уже загружен
        image.display();
    }
}

interface Image {
    void display();
}

// Реальный объект
class RealImage implements Image {
    private String filename;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();
    }

    private void loadFromDisk() {
        System.out.println("Загрузка изображения: " + filename);
    }

    @Override
    public void display() {
        System.out.println("Отображение: " + filename);
    }
}

// Прокси: откладывает создание RealImage
class ProxyImage implements Image {
    private String filename;
    private RealImage realImage;

    public ProxyImage(String filename) {
        this.filename = filename;
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}
```

---

# 4. Flyweight (Легковес)
### Идея
Разделяем общее состояние объектов (intrinsic state), а уникальное (extrinsic state) передаём извне.  
Это сильно экономит память, если объектов очень много.

### Когда использовать
- Нужно создать огромное количество похожих объектов (например, символы текста, частицы в игре).
- Внутреннее состояние объектов одинаковое, а уникальное можно вынести наружу.

### Плюсы / минусы
+ Сильно экономит память.  
+ Ускоряет создание новых объектов (берём из кэша).  
− Усложняет код (нужно разделять состояние).  
− Сложно поддерживать, если много разных вариантов состояний.

### Пример: символы в тексте
```java
// FlyweightDemo.java
// Демонстрация: у нас есть много "символов".
// Общая часть (глиф) хранится в Flyweight, а позиция — внешний контекст.

import java.util.HashMap;
import java.util.Map;

public class FlyweightDemo {
    public static void main(String[] args) {
        GlyphFactory factory = new GlyphFactory();

        Glyph g1 = factory.getGlyph('a');
        Glyph g2 = factory.getGlyph('a');
        Glyph g3 = factory.getGlyph('b');

        g1.draw(1, 1);
        g2.draw(2, 2);
        g3.draw(3, 3);
    }
}

interface Glyph {
    void draw(int x, int y);
}

// Конкретный легковес
class CharacterGlyph implements Glyph {
    private final char symbol;

    public CharacterGlyph(char symbol) {
        this.symbol = symbol;
    }

    @Override
    public void draw(int x, int y) {
        System.out.println("Рисуем '" + symbol + "' в позиции (" + x + "," + y + ")");
    }
}

// Фабрика легковесов
class GlyphFactory {
    private final Map<Character, Glyph> cache = new HashMap<>();

    public Glyph getGlyph(char c) {
        return cache.computeIfAbsent(c, CharacterGlyph::new);
    }
}
```

---

# 5. Facade (Фасад)
### Идея
Фасад предоставляет **простой интерфейс к сложной системе**.  
Клиенту не нужно знать детали — он использует удобный "фасад".

### Когда использовать
- Сложная система, много классов и зависимостей.
- Хотите упростить работу клиента и скрыть детали.

### Плюсы / минусы
+ Упрощает интерфейс системы.  
+ Скрывает детали реализации.  
− Может превратиться в "God object", если в него тянуть слишком много функций.  
− Иногда скрывает слишком много и ограничивает гибкость.

### Пример: Домашний кинотеатр
```java
// FacadeDemo.java
// Демонстрация: клиент включает кинотеатр одной командой, а не кучей.

public class FacadeDemo {
    public static void main(String[] args) {
        HomeTheaterFacade theater = new HomeTheaterFacade(
                new Amplifier(),
                new Projector(),
                new Screen()
        );

        theater.watchMovie("Inception");
        theater.endMovie();
    }
}

// Подсистемы
class Amplifier {
    public void on() { System.out.println("Усилитель включен"); }
    public void off() { System.out.println("Усилитель выключен"); }
}

class Projector {
    public void on() { System.out.println("Проектор включен"); }
    public void off() { System.out.println("Проектор выключен"); }
}

class Screen {
    public void down() { System.out.println("Экран опущен"); }
    public void up() { System.out.println("Экран поднят"); }
}

// Фасад
class HomeTheaterFacade {
    private Amplifier amp;
    private Projector projector;
    private Screen screen;

    public HomeTheaterFacade(Amplifier amp, Projector projector, Screen screen) {
        this.amp = amp;
        this.projector = projector;
        this.screen = screen;
    }

    public void watchMovie(String movie) {
        System.out.println("Готовим кинотеатр к просмотру...");
        screen.down();
        amp.on();
        projector.on();
        System.out.println("Запускаем фильм: " + movie);
    }

    public void endMovie() {
        System.out.println("Выключаем кинотеатр...");
        screen.up();
        projector.off();
        amp.off();
    }
}
```

---

# 6. Bridge (Мост)
### Идея
Разделяет абстракцию и реализацию, позволяя изменять их независимо.  
Клиент работает с абстракцией, а реализация передаётся в неё как зависимость.

### Когда использовать
- Нужно разделить уровни (например, разные виды форм + разные способы отрисовки).
- Когда абстракция и реализация должны изменяться независимо.

### Плюсы / минусы
+ Абстракция и реализация независимы.  
+ Легко добавлять новые реализации и новые абстракции.  
− Усложняет структуру кода (дополнительный слой).  
− Может быть избыточен для маленьких систем.

### Пример: фигуры и способы их рисования
```java
// BridgeDemo.java
// Демонстрация: фигура (Shape) использует интерфейс Renderer для рисования.
// Можно легко добавлять новые фигуры или новые способы рендера.

public class BridgeDemo {
    public static void main(String[] args) {
        Renderer vector = new VectorRenderer();
        Renderer raster = new RasterRenderer();

        Shape circle1 = new Circle(vector);
        Shape circle2 = new Circle(raster);

        circle1.draw();
        circle2.draw();
    }
}

interface Renderer {
    void renderCircle(float radius);
}

class VectorRenderer implements Renderer {
    public void renderCircle(float radius) {
        System.out.println("Рисуем круг в векторном виде с радиусом " + radius);
    }
}

class RasterRenderer implements Renderer {
    public void renderCircle(float radius) {
        System.out.println("Рисуем пиксельный круг с радиусом " + radius);
    }
}

// Абстракция
abstract class Shape {
    protected Renderer renderer;

    public Shape(Renderer renderer) {
        this.renderer = renderer;
    }

    public abstract void draw();
}

class Circle extends Shape {
    private float radius = 5;

    public Circle(Renderer renderer) {
        super(renderer);
    }

    @Override
    public void draw() {
        renderer.renderCircle(radius);
    }
}
```

---

# 7. Decorator (Декоратор)
### Идея
Позволяет **динамически добавлять функциональность объекту**, оборачивая его в другие объекты-декораторы.  
В отличие от наследования, можно гибко комбинировать несколько декораторов.

### Когда использовать
- Нужно добавлять новые обязанности объектам на лету.
- Не хочется создавать кучу подклассов.

### Плюсы / минусы
+ Позволяет гибко добавлять функциональность без изменения исходного кода.  
+ Можно комбинировать несколько декораторов.  
− Много мелких классов, которые могут усложнить понимание.  
− Не всегда очевидно, в каком порядке применяются декораторы.

### Пример: добавление функциональности кофе
```java
// DecoratorDemo.java
// Демонстрация: есть базовый кофе, и мы можем добавлять "ингредиенты" (молоко, сахар).
// Каждый декоратор оборачивает исходный объект.

public class DecoratorDemo {
    public static void main(String[] args) {
        Coffee coffee = new SimpleCoffee();
        System.out.println(coffee.getDescription() + " = " + coffee.getCost());

        coffee = new MilkDecorator(coffee);
        System.out.println(coffee.getDescription() + " = " + coffee.getCost());

        coffee = new SugarDecorator(coffee);
        System.out.println(coffee.getDescription() + " = " + coffee.getCost());
    }
}

interface Coffee {
    String getDescription();
    double getCost();
}

// Базовый класс
class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() {
        return "Обычный кофе";
    }

    @Override
    public double getCost() {
        return 2.0;
    }
}

// Декоратор
abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;

    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
}

// Конкретные декораторы
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", молоко";
    }

    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.5;
    }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", сахар";
    }

    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.2;
    }
}
```

---


# Заключение
- **Adapter** — подключаем несовместимые интерфейсы.  
- **Composite** — дерево объектов (часть-целое).  
- **Proxy** — контролируем доступ к объекту.  
- **Flyweight** — экономим память, разделяя общее состояние.  
- **Facade** — упрощаем доступ к сложной системе.  
- **Bridge** — разделяем абстракцию и реализацию.  
- **Decorator** — добавляем функциональность объекту динамически.

# Поведенческие паттерны проектирования

## 1. Template Method (Шаблонный метод)
**Описание:** Шаблонный метод позволяет определить общий алгоритм в виде последовательности шагов в базовом классе, при этом оставляя конкретную реализацию отдельных шагов подклассам. Такой подход помогает повторно использовать код и обеспечивает единообразие алгоритмов, при этом позволяя подклассам менять только детали без изменения общей структуры.

**Плюсы:**
- Централизованное управление алгоритмом.
- Возможность переопределять отдельные шаги алгоритма.

**Минусы:**
- Подклассы жестко зависят от структуры базового класса.
- Меньшая гибкость по сравнению с Strategy.

**Когда применять:**
- Когда есть повторяющаяся структура алгоритма и необходимо менять только отдельные шаги.

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
abstract class DataProcessor {
    public final void process() {
        readData();  // шаг 1: чтение данных
        processData(); // шаг 2: обработка данных
        saveData();    // шаг 3: сохранение данных
    }

    protected abstract void readData();
    protected abstract void processData();
    protected abstract void saveData();
}

@Slf4j
class CSVDataProcessor extends DataProcessor {
    @Override
    protected void readData() {
        log.info("Чтение CSV данных");
    }

    @Override
    protected void processData() {
        log.info("Обработка CSV данных");
    }

    @Override
    protected void saveData() {
        log.info("Сохранение CSV данных");
    }
}

public class Main {
    public static void main(String[] args) {
        DataProcessor processor = new CSVDataProcessor();
        processor.process();
    }
}
```

---

## 2. Mediator (Посредник)
**Описание:** Посредник централизует взаимодействие между объектами. Вместо того чтобы компоненты напрямую ссылались друг на друга, они общаются через посредника. Это снижает связность и упрощает поддержку сложных систем с большим количеством взаимозависимых объектов.

**Плюсы:**
- Уменьшение связности компонентов.
- Упрощение изменений и добавления новых взаимодействий.

**Минусы:**
- Посредник может стать слишком сложным и тяжёлым для поддержки.

**Когда применять:**
- Когда много объектов взаимодействуют друг с другом и необходимо централизовать это взаимодействие.

```java
import lombok.Data;
import lombok.extern.slf4j.Slf4j;

@Slf4j
class Mediator {
    public void notify(Component sender, String event) {
        if (event.equals("A")) {
            log.info("Mediator реагирует на событие A и инициирует действия B");
        } else if (event.equals("B")) {
            log.info("Mediator реагирует на событие B и инициирует действия A");
        }
    }
}

@Data
class Component {
    private Mediator mediator;

    public void trigger(String event) {
        log.info("Component триггерит событие: {}", event);
        mediator.notify(this, event);
    }
}

public class Main {
    public static void main(String[] args) {
        Mediator mediator = new Mediator();
        Component comp1 = new Component();
        Component comp2 = new Component();

        comp1.setMediator(mediator);
        comp2.setMediator(mediator);

        comp1.trigger("A");
        comp2.trigger("B");
    }
}
```

---

## 3. Chain of Responsibility (Цепочка обязанностей)
**Описание:** Позволяет передавать запрос по цепочке объектов до тех пор, пока один из них не обработает его. Отправитель запроса не знает, какой объект его обработает, что делает систему гибкой и легко расширяемой.

**Плюсы:**
- Гибкость в обработке запросов.
- Разделение обязанностей.

**Минусы:**
- Нет гарантии, что запрос будет обработан.
- Цепочка может стать слишком длинной.

**Когда применять:**
- Когда запрос может быть обработан несколькими объектами и нужно избегать жёсткой зависимости.

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
abstract class Handler {
    protected Handler next;

    public void setNext(Handler next) {
        this.next = next;
    }

    public abstract void handle(String request);
}

@Slf4j
class ConcreteHandlerA extends Handler {
    @Override
    public void handle(String request) {
        if (request.equals("A")) {
            log.info("Handler A обработал запрос");
        } else if (next != null) {
            next.handle(request);
        }
    }
}

@Slf4j
class ConcreteHandlerB extends Handler {
    @Override
    public void handle(String request) {
        if (request.equals("B")) {
            log.info("Handler B обработал запрос");
        } else if (next != null) {
            next.handle(request);
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Handler handlerA = new ConcreteHandlerA();
        Handler handlerB = new ConcreteHandlerB();
        handlerA.setNext(handlerB);

        handlerA.handle("B");
    }
}
```

---

## 4. Observer (Наблюдатель)
**Описание:** Наблюдатель позволяет объектам подписываться на события другого объекта и автоматически получать уведомления об изменениях. Основная цель — слабая связка между объектами: субъекты уведомляют наблюдателей без знания их внутренней реализации.

**Плюсы:**
- Слабая связка между объектами.
- Легкое добавление новых наблюдателей.

**Минусы:**
- Сложно отследить всех подписчиков.
- Массовое уведомление может снизить производительность.

**Когда применять:**
- Когда один объект должен уведомлять множество других о своих изменениях.

```java
import lombok.extern.slf4j.Slf4j;
import java.util.ArrayList;
import java.util.List;

@Slf4j
interface Observer {
    void update(String message);
}

@Slf4j
class ConcreteObserver implements Observer {
    private String name;

    public ConcreteObserver(String name) {
        this.name = name;
    }

    @Override
    public void update(String message) {
        log.info("Observer {} получил сообщение: {}", name, message);
    }
}

@Slf4j
class Subject {
    private List<Observer> observers = new ArrayList<>();

    public void addObserver(Observer observer) {
        observers.add(observer);
    }

    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Subject subject = new Subject();
        subject.addObserver(new ConcreteObserver("A"));
        subject.addObserver(new ConcreteObserver("B"));

        subject.notifyObservers("Событие произошло");
    }
}
```

---

## 5. Strategy (Стратегия)
**Описание:** Стратегия позволяет менять алгоритм или поведение объекта во время выполнения. Основная идея — вынести алгоритмы в отдельные классы и использовать их через общий интерфейс, что обеспечивает гибкость и слабую связку.

**Плюсы:**
- Легкая замена алгоритмов.
- Слабая связка между контекстом и стратегиями.

**Минусы:**
- Увеличение количества классов.

**Когда применять:**
- Когда требуется менять поведение объекта во время выполнения.

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
interface PaymentStrategy {
    void pay(int amount);
}

@Slf4j
class CreditCardPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        log.info("Оплата {} кредитной картой", amount);
    }
}

@Slf4j
class PayPalPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        log.info("Оплата {} через PayPal", amount);
    }
}

@Slf4j
class ShoppingCart {
    private PaymentStrategy strategy;

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void checkout(int amount) {
        strategy.pay(amount); // делегирование оплаты выбранной стратегии
    }
}

public class Main {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        cart.setPaymentStrategy(new CreditCardPayment());
        cart.checkout(500);

        cart.setPaymentStrategy(new PayPalPayment());
        cart.checkout(1000);
    }
}

---

## 6. Command (Команда)
**Описание:** Паттерн Команда инкапсулирует запрос как объект, позволяя параметризовать объекты действиями, ставить их в очередь или журналировать. Он отделяет отправителя запроса от объекта, который выполняет действие. Такой подход облегчает реализацию отмены и повтора операций, а также поддержку макросов. Паттерн обеспечивает гибкость и расширяемость системы, позволяя легко добавлять новые команды без изменения существующих классов. Команды могут быть сохранены и выполнены позже, что делает их удобными для отложенных действий.

**Плюсы:**
- Инкапсуляция действий.
- Возможность реализовать отмену и повтор операций.
- Упрощает поддержку макросов и отложенного выполнения.

**Минусы:**
- Увеличение числа классов.
- Может усложнить структуру проекта при большом количестве команд.

**Когда применять:**
- Когда нужно параметризовать объекты действиями.
- Когда требуется сохранять, журналировать или откатывать действия.

```java
import lombok.extern.slf4j.Slf4j;

// Интерфейс команды с методом execute
@Slf4j
interface Command {
    void execute();
}

// Класс устройства, на котором выполняются команды
@Slf4j
class Light {
    public void turnOn() {
        log.info("Свет включен");
    }

    public void turnOff() {
        log.info("Свет выключен");
    }
}

// Конкретная команда включения света
@Slf4j
class TurnOnCommand implements Command {
    private Light light;

    public TurnOnCommand(Light light) {
        this.light = light;
    }

    @Override
    public void execute() {
        light.turnOn(); // делегируем действие устройству
    }
}

// Пульт управления, который вызывает команды
@Slf4j
class RemoteControl {
    private Command command;

    public void setCommand(Command command) {
        this.command = command;
    }

    public void pressButton() {
        command.execute(); // выполняем команду
    }
}

public class Main {
    public static void main(String[] args) {
        Light light = new Light();
        Command turnOn = new TurnOnCommand(light);
        RemoteControl remote = new RemoteControl();
        remote.setCommand(turnOn);
        remote.pressButton(); // включение света через команду
    }
}
```

---

## 7. State (Состояние)
**Описание:** Паттерн Состояние позволяет объекту изменять поведение при изменении его внутреннего состояния. Каждый объект состояния инкапсулирует определённое поведение, что заменяет громоздкие конструкции if/else или switch. Паттерн повышает читаемость и расширяемость кода. Объект делегирует выполнение методов своему текущему состоянию, что делает систему более гибкой. Легко добавлять новые состояния без изменения существующих классов.

**Плюсы:**
- Упрощает код с множеством условий.
- Легко добавлять новые состояния.
- Повышает читаемость и поддержку.

**Минусы:**
- Увеличение числа классов.
- Сложнее отследить все состояния в больших системах.

**Когда применять:**
- Когда поведение объекта зависит от его состояния.
- Когда необходимо легко расширять систему новыми состояниями.

```java
import lombok.extern.slf4j.Slf4j;

// Интерфейс состояния
@Slf4j
interface State {
    void handle();
}

// Конкретное состояние включенного устройства
@Slf4j
class OnState implements State {
    @Override
    public void handle() {
        log.info("Состояние ON: устройство работает");
    }
}

// Конкретное состояние выключенного устройства
@Slf4j
class OffState implements State {
    @Override
    public void handle() {
        log.info("Состояние OFF: устройство выключено");
    }
}

// Контекст, который меняет состояния
@Slf4j
class Device {
    private State state;

    public void setState(State state) {
        this.state = state; // изменение текущего состояния
    }

    public void request() {
        state.handle(); // делегирование поведения текущему состоянию
    }
}

public class Main {
    public static void main(String[] args) {
        Device device = new Device();
        device.setState(new OnState());
        device.request(); // устройство работает
        device.setState(new OffState());
        device.request(); // устройство выключено
    }
}
```

---

## 8. Visitor (Посетитель)
**Описание:** Паттерн Visitor позволяет добавлять новые операции для группы объектов, не изменяя их классы. Он отделяет алгоритмы от объектов, над которыми эти алгоритмы выполняются. Это особенно полезно для сложных иерархий, где часто добавляются новые операции. Паттерн делает код более открытым к расширению и закрытым к модификации. Он упрощает поддержку и тестирование различных операций.

**Плюсы:**
- Легко добавлять новые операции.
- Подходит для сложных структур объектов.
- Снижает связность между классами элементов и операциями.

**Минусы:**
- Трудно добавлять новые типы элементов.
- Требуется строгая структура иерархии.

**Когда применять:**
- Когда нужно выполнять разные операции над объектами сложной структуры.
- Когда операции часто меняются или добавляются.

```java
import lombok.extern.slf4j.Slf4j;

// Интерфейс посетителя
@Slf4j
interface Visitor {
    void visit(Book book);
    void visit(CD cd);
}

// Конкретный посетитель для вывода цены
@Slf4j
class PriceVisitor implements Visitor {
    @Override
    public void visit(Book book) {
        log.info("Цена книги: {}", book.getPrice());
    }

    @Override
    public void visit(CD cd) {
        log.info("Цена CD: {}", cd.getPrice());
    }
}

// Элемент структуры
@Slf4j
interface Item {
    void accept(Visitor visitor);
}

// Конкретные элементы
@Slf4j
class Book implements Item {
    private int price;

    public Book(int price) {
        this.price = price;
    }

    public int getPrice() {
        return price;
    }

    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}

@Slf4j
class CD implements Item {
    private int price;

    public CD(int price) {
        this.price = price;
    }

    public int getPrice() {
        return price;
    }

    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}

public class Main {
    public static void main(String[] args) {
        Item[] items = {new Book(100), new CD(200)};
        Visitor visitor = new PriceVisitor();
        for (Item item : items) {
            item.accept(visitor);
        }
    }
}
```

---

## 9. Interpreter (Интерпретатор)
**Описание:** Паттерн Интерпретатор определяет грамматику простого языка и интерпретирует предложения этого языка. Он позволяет создавать интерпретаторы для специализированных языков и выражений. Паттерн упрощает анализ и обработку выражений, делая код более структурированным. Легко расширять новые правила и синтаксис. Особенно полезен для вычисления, проверки и интерпретации формул или DSL.

**Плюсы:**
- Легко расширять синтаксис и правила.
- Четкая структура грамматики.

**Минусы:**
- Сложность поддержки для сложного языка.
- Может создавать много небольших классов.

**Когда применять:**
- Для интерпретации выражений специализированного языка.
- Для построения компиляторов, парсеров или простых DSL.

```java
import lombok.extern.slf4j.Slf4j;

// Интерфейс выражения
@Slf4j
interface Expression {
    boolean interpret(String context);
}

// Конкретное терминальное выражение
@Slf4j
class TerminalExpression implements Expression {
    private String data;

    public TerminalExpression(String data) {
        this.data = data;
    }

    @Override
    public boolean interpret(String context) {
        return context.contains(data);
    }
}

// Операция ИЛИ над выражениями
@Slf4j
class OrExpression implements Expression {
    private Expression expr1;
    private Expression expr2;

    public OrExpression(Expression expr1, Expression expr2) {
        this.expr1 = expr1;
        this.expr2 = expr2;
    }

    @Override
    public boolean interpret(String context) {
        return expr1.interpret(context) || expr2.interpret(context);
    }
}

public class Main {
    public static void main(String[] args) {
        Expression expr1 = new TerminalExpression("Java");
        Expression expr2 = new TerminalExpression("Python");
        Expression orExpression = new OrExpression(expr1, expr2);

        log.info("Результат интерпретации: {}", orExpression.interpret("Java"));
    }
}
```

---

## 10. Iterator (Итератор)
**Описание:** Итератор позволяет последовательно обходить элементы коллекции, не раскрывая её внутреннюю структуру. Паттерн предоставляет единый интерфейс для обхода разных типов коллекций. Он делает код обхода универсальным и повторно используемым. Позволяет реализовать несколько способов обхода для одной коллекции. Упрощает работу с комплексными структурами данных.

**Плюсы:**
- Универсальный способ обхода.
- Слабая связка между коллекцией и обходом.

**Минусы:**
- Дополнительные объекты-итераторы.
- Могут усложнить код при очень больших коллекциях.

**Когда применять:**
- Когда требуется последовательный доступ к элементам коллекции.
- Когда нужно скрыть внутреннюю структуру коллекции.

```java
import lombok.extern.slf4j.Slf4j;
import java.util.ArrayList;
import java.util.List;
import java.util.Iterator;

@Slf4j
public class Main {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("A");
        list.add("B");
        list.add("C");

       

---

## 10. Iterator (Итератор)
**Описание:** Итератор позволяет последовательно обходить элементы коллекции, не раскрывая её внутреннюю структуру. Он предоставляет единый интерфейс обхода для различных типов коллекций. Это упрощает работу с комплексными структурами данных и делает код более универсальным и повторно используемым. Паттерн поддерживает несколько способов обхода одной коллекции. Он также позволяет безопасно перебирать элементы, не зависимо от внутренней реализации коллекции.

**Плюсы:**
- Универсальный способ обхода элементов.
- Слабая связка между коллекцией и обходом.
- Возможность реализовать разные стратегии обхода.

**Минусы:**
- Дополнительные объекты-итераторы.
- Может увеличивать использование памяти при больших коллекциях.

**Когда применять:**
- Когда требуется последовательный доступ к элементам коллекции.
- Когда нужно скрыть внутреннюю структуру коллекции.

```java
import lombok.extern.slf4j.Slf4j;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

@Slf4j
public class Main {
    public static void main(String[] args) {
        // Создаем коллекцию элементов
        List<String> list = new ArrayList<>();
        list.add("A");
        list.add("B");
        list.add("C");

        // Получаем итератор для обхода коллекции
        Iterator<String> iterator = list.iterator();

        // Проходим по всем элементам, не зная внутренней структуры коллекции
        while (iterator.hasNext()) {
            String item = iterator.next();
            log.info("Элемент коллекции: {}", item); // выводим текущий элемент
        }
    }
}
```

---

## 11. Memento (Хранитель)
**Описание:** Memento позволяет сохранять и восстанавливать прошлое состояние объекта, не нарушая инкапсуляцию. Это полезно для реализации undo/redo и отката изменений. Хранитель отделяет состояние объекта от его поведения, обеспечивая сохранение истории изменений без вмешательства в код объекта. Паттерн повышает гибкость системы, позволяя легко восстанавливать прежние состояния. Он также упрощает тестирование и откат ошибок в сложных объектах.

**Плюсы:**
- Сохраняет инкапсуляцию объекта.
- Позволяет реализовать undo/redo и откат состояний.
- Упрощает тестирование и управление сложными объектами.

**Минусы:**
- Может потреблять много памяти при частом сохранении состояний.
- Требует дополнительного кода для управления хранителями.

**Когда применять:**
- Когда необходимо сохранять и восстанавливать состояния объектов.
- При реализации undo/redo и сохранении истории изменений.

```java
import lombok.Getter;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;

// Хранитель, содержащий состояние
@Slf4j
@Getter
class Memento {
    private final String state;

    public Memento(String state) {
        this.state = state;
    }
}

// Объект, состояние которого нужно сохранять
@Slf4j
class Originator {
    @Setter
    private String state;

    // Сохраняем текущее состояние
    public Memento save() {
        log.info("Сохраняем состояние: {}", state);
        return new Memento(state);
    }

    // Восстанавливаем предыдущее состояние
    public void restore(Memento memento) {
        state = memento.getState();
        log.info("Восстановлено состояние: {}", state);
    }
}

@Slf4j
public class Main {
    public static void main(String[] args) {
        Originator originator = new Originator();
        originator.setState("Состояние1");
        Memento saved = originator.save(); // сохраняем текущее состояние

        originator.setState("Состояние2");
        log.info("Текущее состояние изменилось: {}", originator.getState());

        originator.restore(saved); // откат к предыдущему состоянию
    }
}




[Вопросы для собеседования](README.md)
