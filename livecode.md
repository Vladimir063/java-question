
# 10 задач для собеседований Senior Java — разбор и корректные реализации

Ниже — тщательно отформатированная шпаргалка по 10 задачам. Для каждой:  
**(1) задание, (2) ошибки, (3) как сделать лучше, (4) корректная реализация.**  
Код — на Java/Spring, где уместно добавлены комментарии и best practices.

> Примечание: вы прислали 9 исходных задач. Чтобы получить «круглые» 10, я добавил **задачу №10 (антипаттерн “создание потоков вручную” → `@Async`/пул потоков)** — классическую тему для сеньорских собесов.

---

## Задача 1. REST + БД + Kafka (создание заказа и публикация события)

### 1) Задание
Есть контроллер и сервис, которые сохраняют заказ в БД и отправляют событие в Kafka. Требуется найти проблемы и предложить корректную реализацию (надежность, транзакционность, валидность данных, идемпотентность и т. п.).

### 2) Ошибки в существующем коде
- Полевая инъекция `@Autowired` — хрупко, сложнее тестировать. Лучше конструкторная инъекция.
- Контроллер всегда возвращает `200 OK "ok"` — неинформативно. Должно быть `201 Created` и/или вернуть `id` ресурса.
- Отправляется тупая строка `"order_created"` — без полезной нагрузки. Нет ключа, нет версии события.
- Нет валидации входного `OrderRequest` (например, `total >= 0`, `userId` не пустой).
- Отсутствует гарантированная согласованность между записью в БД и сообщением в Kafka. При падении между `save` и `send` возможны «зомби»-состояния (запись есть, события нет).
- Нет обработки ошибок публикации, ретраев, идемпотентности (повторная доставка).
- Сущность `Order` не указывает стратегию `GenerationType` и не закрывает детали типа денег (`BigDecimal` scale/precision).

### 3) Что сделать лучше
- Перейти на **конструкторную инъекцию**.
- Валидировать входные данные (`javax.validation`).
- Возвращать `201 Created` с `Location` и/или DTO с `id`.
- Событие — полноценный JSON с ключом `orderId`, `eventType`, `schemaVersion`, полезной нагрузкой.
- Для надежности — **Outbox pattern**: событие сначала пишем в таблицу `outbox` в рамках одной транзакции с заказом, а отдельный publisher (транзакционно/периодически) читает `outbox` и шлет в Kafka с маркой «отправлено».
- Либо настраивать **Kafka transactions** + `ChainedTransactionManager` (сложнее и хрупче), поэтому чаще рекомендуют **Outbox**.

### 4) Корректная реализация (вариант с Outbox)

```java
// DTO
public record OrderRequest(
    @NotBlank String userId,
    @NotNull @PositiveOrZero BigDecimal total
) {}

public record OrderResponse(Long id) {}

// Entity
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String userId;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal total;

    @Column(nullable = false)
    private String status;

    // getters/setters/constructors
}

// Outbox entity
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String aggregateType; // "Order"

    @Column(nullable = false)
    private Long aggregateId; // orderId

    @Column(nullable = false)
    private String eventType; // "ORDER_CREATED"

    @Column(nullable = false, columnDefinition = "text")
    private String payload; // JSON

    @Column(nullable = false)
    private Instant createdAt = Instant.now();

    @Column(nullable = false)
    private boolean sent = false;

    // getters/setters
}

// Repositories
public interface OrderRepository extends JpaRepository<Order, Long> {}
public interface OutboxRepository extends JpaRepository<OutboxEvent, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select e from OutboxEvent e where e.sent = false order by e.id asc")
    List<OutboxEvent> findBatch(Pageable pageable);
}

// Service
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    @Transactional
    public Long createOrder(OrderRequest request) {
        Order order = new Order();
        order.setUserId(request.userId());
        order.setTotal(request.total());
        order.setStatus("NEW");
        order = orderRepository.save(order);

        Map<String, Object> event = Map.of(
            "orderId", order.getId(),
            "eventType", "ORDER_CREATED",
            "schemaVersion", 1,
            "userId", order.getUserId(),
            "total", order.getTotal()
        );
        String payload;
        try {
            payload = objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize event", e);
        }

        OutboxEvent outbox = new OutboxEvent();
        outbox.setAggregateType("Order");
        outbox.setAggregateId(order.getId());
        outbox.setEventType("ORDER_CREATED");
        outbox.setPayload(payload);
        outboxRepository.save(outbox);

        return order.getId();
    }
}

// Controller
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody OrderRequest request, UriComponentsBuilder uri) {
        Long id = orderService.createOrder(request);
        URI location = uri.path("/orders/{id}").build(id);
        return ResponseEntity.created(location).body(new OrderResponse(id));
    }
}

// Outbox publisher (отдельный бин/планировщик/consumer)
@Component
@RequiredArgsConstructor
public class OutboxPublisher {
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Transactional
    @Scheduled(fixedDelayString = "${outbox.publisher.delay-ms:1000}")
    public void publishBatch() {
        List<OutboxEvent> batch = outboxRepository.findBatch(PageRequest.of(0, 100));
        for (OutboxEvent e : batch) {
            String key = e.getAggregateId().toString();
            kafkaTemplate.send("orders", key, e.getPayload()).completable().whenComplete((md, ex) -> {
                if (ex == null) {
                    // помечаем как отправленное
                    markSent(e.getId());
                } else {
                    // лог + ретраи (оставляем sent=false)
                }
            });
        }
    }

    @Transactional
    public void markSent(Long outboxId) {
        outboxRepository.findById(outboxId).ifPresent(o -> {
            o.setSent(true);
            outboxRepository.save(o);
        });
    }
}
```

---

## Задача 2. Транзакции и переименование файла на диске (вариант 1)

### 1) Задание
Метод: выбрать `id` по старому имени, переименовать файл на диске, обновить имя в БД. Нужно избежать несогласованности БД/файловой системы, SQL‑инъекции и багов транзакций.

### 2) Ошибки
- Конкатенация SQL → **SQL‑инъекция**.
- Переименование файла внутри одной БД‑транзакции: файловая система **не откатывается** при rollback.
- Отсутствие блокировок/проверок конкурентных изменений.
- Нет обработки ошибок и компенсации.
- Непонятный метод `insert` (опечатка? лишняя строка).

### 3) Что сделать лучше
- Использовать **параметризованные запросы** (`JdbcTemplate`/`JPA`).
- Разделить БД‑транзакцию и I/O: сначала безопасный поиск + валидация, затем **атомарно** переименовать на диске, после — **короткая транзакция обновления** в БД. При ошибке обновления — **компенсация** (переименовать обратно).
- Добавить блокировку строки `SELECT ... FOR UPDATE` (или optimistic lock) на время операции.

### 4) Корректная реализация (JdbcTemplate + компенсация)

```java
@Service
@RequiredArgsConstructor
public class FileService {
    private final JdbcTemplate jdbc;

    public void process(String oldName, String newName) {
        Long id = jdbc.queryForObject(
            "select id from file where name = ? for update skip locked",
            Long.class, oldName
        );
        if (id == null) throw new IllegalArgumentException("File not found: " + oldName);

        Path oldPath = Path.of("/data", oldName);
        Path newPath = Path.of("/data", newName);

        try {
            Files.move(oldPath, newPath, StandardCopyOption.ATOMIC_MOVE);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to rename on FS", e);
        }

        try {
            jdbc.update("update file set name = ? where id = ?", newName, id);
        } catch (DataAccessException e) {
            // компенсация
            try { Files.move(newPath, oldPath, StandardCopyOption.ATOMIC_MOVE); }
            catch (IOException ignored) {}
            throw e;
        }
    }
}
```

---

## Задача 3. Класс `Cat4` (конкурентность, JDBC, finalize, toString)

### 1) Задание
Исправить ошибки потокобезопасности, JDBC, `finalize`, логгирования, равенства/строки.

### 2) Ошибки
- `private static final int jumpsCount = 0;` — **final** и инкремент → компиляция не пройдет; синглтон‑счетчик для всех котов.
- Создание `new Thread` в цикле без пула — **антипаттерн**.
- `doQuery` — **SQL‑инъекция**, небезопасная конкатенация байт.
- `toString()` лезет в БД → побочные эффекты, `SQLException`.
- `finalize()` — **устаревший** API, не гарантируется вызов, не нужен.
- `getJumpsCount()` и инкремент не атомарны.
- Логи без статики/конфигурации.
- `equals()` без `hashCode()`.

### 3) Что сделать лучше
- Использовать `AtomicInteger` для счетчика (или инстанс‑поле).
- Пул потоков (`ExecutorService`) или `@Async`.
- `PreparedStatement` и безопасная кодировка параметров.
- `toString()` без запросов к БД.
- Удалить `finalize()`.
- Определить `equals`/`hashCode` по **стабильному** полю (`name`).

### 4) Корректная реализация

```java
public final class Cat4 {
    private static final Logger log = Logger.getLogger(Cat4.class.getName());

    private final DataSource dataSource;
    private final String name;
    private final AtomicInteger jumps = new AtomicInteger(0);
    private final ExecutorService executor;

    public Cat4(String name, DataSource dataSource, ExecutorService executor) {
        this.name = Objects.requireNonNull(name);
        this.dataSource = Objects.requireNonNull(dataSource);
        this.executor = Objects.requireNonNull(executor);
    }

    public void doJumps(int jumpsCount) {
        for (int i = 0; i < jumpsCount; i++) {
            executor.submit(this::doJump);
        }
    }

    private void doJump() {
        jumps.incrementAndGet();
        log.fine("Jump!");
    }

    public void doMeow() { log.fine("Meow!"); }

    public void doQuery(byte[] parameters) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement("insert into cats(name) values (?)")) {
            ps.setString(1, new String(parameters, StandardCharsets.UTF_8));
            ps.executeUpdate();
        }
    }

    protected int getAndResetJumps() {
        return jumps.getAndSet(0);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Cat4)) return false;
        Cat4 cat4 = (Cat4) o;
        return name.equals(cat4.name);
    }

    @Override
    public int hashCode() { return Objects.hash(name); }

    @Override
    public String toString() {
        return "Cat4{name='%s'}".formatted(name);
    }
}
```

---

## Задача 4. Класс «Банкомат»: номиналы, остаток, выдача суммы

### 1) Задание
Исправить синтаксические ошибки и реализовать методы **загрузки**, **выгрузки**, **расчета остатка**, **выдачи суммы** по купюрам.

### 2) Ошибки
- Много опечаток/синтаксиса: `denominations,keySet()`, запятые, комментарии.
- Использование устаревшего `Hashtable` и сырых массивов ключей.
- `getRest()` считает через автоупаковку `Integer`, возвращает `Integer`, перемножая short*int без контроля переполнения.
- Нет неизменяемости/копий, потенциальные внешние модификации.
- Нет алгоритма выдачи суммы (жадный разбор по убыванию).

### 3) Что сделать лучше
- Использовать `NavigableMap<Integer, Integer>` (номинал → количество), `TreeMap` по убыванию.
- Возвращать **копии**, не давая внешнему коду сломать состояние.
- В `getRest()` использовать `long`/`BigInteger` по необходимости.
- Реализовать **жадный алгоритм** выдачи (или динамику, если номиналы «плохие»).

### 4) Корректная реализация

```java
public class Atm {
    // ключ: номинал, значение: количество
    private final NavigableMap<Integer, Integer> banknotes = new TreeMap<>(Comparator.reverseOrder());

    // загрузка: слияние остатков
    public synchronized void load(Map<Integer, Integer> add) {
        Objects.requireNonNull(add);
        add.forEach((denom, count) -> {
            if (denom <= 0 || count <= 0) throw new IllegalArgumentException("Bad input");
            banknotes.merge(denom, count, Integer::sum);
        });
    }

    // выгрузка всех купюр (копия) и очистка ячеек
    public synchronized Map<Integer, Integer> unload() {
        Map<Integer, Integer> copy = new LinkedHashMap<>(banknotes);
        banknotes.clear();
        return copy;
    }

    // остаток
    public synchronized long getRest() {
        long sum = 0L;
        for (var e : banknotes.entrySet()) {
            sum += (long) e.getKey() * e.getValue();
        }
        return sum;
    }

    // выдача суммы (жадный алгоритм)
    public synchronized Map<Integer, Integer> dispense(int amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount <= 0");
        if (amount > getRest()) throw new IllegalArgumentException("Insufficient funds");

        Map<Integer, Integer> toGive = new LinkedHashMap<>();
        int remaining = amount;

        for (var e : banknotes.entrySet()) {
            int denom = e.getKey();
            int available = e.getValue();
            int need = remaining / denom;
            if (need > 0) {
                int give = Math.min(need, available);
                if (give > 0) {
                    toGive.put(denom, give);
                    remaining -= give * denom;
                }
            }
        }

        if (remaining != 0) {
            throw new IllegalStateException("Cannot compose exact amount");
        }

        // списываем из кассет
        toGive.forEach((d, c) -> banknotes.computeIfPresent(d, (k, v) -> v - c));
        banknotes.entrySet().removeIf(e -> e.getValue() == 0);

        return toGive;
    }
}
```

---

## Задача 5. Production‑класс `Cat4` (кэш, конкурентность, Feign/JDBC)

### 1) Задание
В классе есть поле `ConcurrentHashMap<byte[], BigDecimal> CACHE`, методы `doRandomJumps`, `getCat4Name`, `doQuery`, `doQueryCached`, `getJumpsCount` и др. Найти и исправить проблемы.

### 2) Ошибки
- Ключи `byte[]` в `ConcurrentHashMap` — сравнение **по ссылке**, кэш не работает. Нужно иммутабельный ключ (`String`, `ByteBuffer.wrap(..)`, `MessageDigest`).
- Создание `new Thread()` в цикле — вместо пула.
- Непотокобезопасный `jumpsCount` (int, инкремент с гонками).
- `getCat4Name()` ловит `NullPointerException` как норму — заменяем на `Optional`/валидацию/дефолт.
- `doQuery()` — SQL‑инъекция + лишняя `)` в SQL + нет проверки `resultSet.next()`.
- `doQueryCached()` — двойной `get` без атомики, неиспользует `computeIfAbsent`.
- Потенциальная **утечка памяти** из-за бесконечного роста кэша.
- Нет таймаута/лимитов внешних вызовов.

### 3) Что сделать лучше
- Ключ кэша — `String` (UTF‑8) или `ByteBuffer`/`Hex` хэша.
- Использовать пул потоков/`@Async`.
- `AtomicInteger` для счетчика.
- Нормальная обработка `null` для профиля.
- `PreparedStatement` и корректный SQL.
- `computeIfAbsent` с контролем ошибок.
- Ограничить кэш: `Caffeine`/`Guava`/`Spring Cache` с `@Cacheable` и TTL.

### 4) Корректная реализация (с Spring Cache)

```java
@Service
@RequiredArgsConstructor
public class Cat4 {
    private static final Logger log = Logger.getLogger(Cat4.class.getName());

    private final DataSource dataSource;
    private final TaskExecutor taskExecutor; // ThreadPoolTaskExecutor
    private final AtomicInteger jumps = new AtomicInteger(0);

    @Setter private Cat4Profile cat4Profile;

    public void doRandomJumps(int maxJumps) {
        int jumpsToDo = ThreadLocalRandom.current().nextInt(maxJumps + 1);
        for (int i = 0; i < jumpsToDo; i++) {
            taskExecutor.execute(this::doJump);
        }
    }

    public String getCat4Name() {
        return Optional.ofNullable(cat4Profile)
                .map(Cat4Profile::getCatName)
                .orElse("Max");
    }

    private void doJump() {
        jumps.incrementAndGet();
        log.fine("Jump!");
    }

    public void doMeow() { log.fine("Meow!"); }

    public BigDecimal doQuery(String name) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "select weight from Cat where name = ?")) {
            ps.setString(1, name);
            try (ResultSet rs = ps.executeQuery()) {
                if (!rs.next()) throw new IllegalArgumentException("Cat not found");
                return rs.getBigDecimal("weight");
            }
        }
    }

    @Cacheable(value = "catWeight", key = "#name", cacheManager = "redisCacheManager")
    public BigDecimal doQueryCached(String name) throws SQLException {
        return doQuery(name);
    }

    public int getAndResetJumps() { return jumps.getAndSet(0); }
}
```

---

## Задача 6. Обработка документов по типу (enum → стратегия)

### 1) Задание
Есть `DocumentService.process(Document[])`, где `switch` по `DocumentType`. Исправить ошибки и показать более гибкий, расширяемый подход.

### 2) Ошибки
- В `switch` по enum писать `case PDF:`, а не `case DocumentType.PDF:`.
- Нет default/валидации null.
- Жесткая связка логики со `switch`; при добавлении типа — менять `switch`.

### 3) Что сделать лучше
- Вынести специализацию в **стратегии**: интерфейс `DocumentProcessor`, карта `type → processor`.  
- Простое добавление новых обработчиков без изменения `DocumentService`.

### 4) Корректная реализация

```java
public enum DocumentType { XML, PDF, DOCX }

public class Document {
    String id;
    DocumentType type;
    String content;
}

public interface DocumentProcessor {
    void process(Document doc);
    DocumentType supports();
}

@Component
class PdfProcessor implements DocumentProcessor {
    public void process(Document doc) { /* ... */ }
    public DocumentType supports() { return DocumentType.PDF; }
}

@Component
class DocxProcessor implements DocumentProcessor {
    public void process(Document doc) { /* ... */ }
    public DocumentType supports() { return DocumentType.DOCX; }
}

@Component
class XmlProcessor implements DocumentProcessor {
    public void process(Document doc) { /* ... */ }
    public DocumentType supports() { return DocumentType.XML; }
}

@Service
@RequiredArgsConstructor
class DocumentService {
    private final Map<DocumentType, DocumentProcessor> registry;

    public DocumentService(List<DocumentProcessor> processors) {
        this.registry = processors.stream()
            .collect(Collectors.toUnmodifiableMap(DocumentProcessor::supports, p -> p));
    }

    public void process(Document[] docs) {
        for (Document d : docs) {
            Objects.requireNonNull(d, "doc is null");
            DocumentProcessor p = registry.get(d.type);
            if (p == null) throw new IllegalArgumentException("Unsupported type: " + d.type);
            p.process(d);
        }
    }
}
```

---

## Задача 7. Транзакции и переименование файла (вариант 2)

### 1) Задание
Вариант кода обновляет БД до переименования файлов на диске. Нужно исправить порядок и устранить инъекции.

### 2) Ошибки
- Сначала `update` в БД, потом `processFile` → при падении `processFile` возникает **несогласованность**.
- Конкатенация SQL → **инъекции**.
- Нет проверок существования/конкурентности.

### 3) Что сделать лучше
- Сначала безопасно переименовать файл (атомарно), затем — однократное обновление БД (короткая транзакция).
- Использовать параметризованные запросы (`JdbcTemplate`/`NamedParameterJdbcTemplate`).
- Добавить компенсацию при неудачном обновлении БД.

### 4) Корректная реализация (Spring JDBC)

```java
@Service
@RequiredArgsConstructor
public class FileRenameService {
    private final JdbcTemplate jdbc;

    public void process(String oldName, String newName) {
        Long id = jdbc.queryForObject("select id from file where name = ?", Long.class, oldName);
        if (id == null) throw new IllegalArgumentException("Not found: " + oldName);

        Path oldPath = Path.of("/files", oldName);
        Path newPath = Path.of("/files", newName);

        try {
            Files.move(oldPath, newPath, StandardCopyOption.ATOMIC_MOVE);
        } catch (IOException e) {
            throw new IllegalStateException("FS rename failed", e);
        }

        try {
            jdbc.update("update file set name = ? where id = ?", newName, id);
        } catch (DataAccessException e) {
            try { Files.move(newPath, oldPath, StandardCopyOption.ATOMIC_MOVE); } catch (IOException ignored) {}
            throw e;
        }
    }
}
```

---

## Задача 8. Контракт/ценообразование: состояние в бине‑сервисе

### 1) Задание
Контроллер грузит контракт из репозитория, сеттит его в поле синглтон‑сервиса `ContractService` и вызывает `updatePrice()`. Надо исправить.

### 2) Ошибки
- Держать **состояние** (`contract`) в `@Service` (синглтон) — **гонки** между параллельными запросами.
- `updatePrice()` ничего не сохраняет (`repository.save(...)` отсутствует).
- Нет транзакции для сохранения.
- Контроллер возвращает `void`, не сообщает статус.
- Не обрабатываются `404`, валидация, логирование.

### 3) Что сделать лучше
- Сделать сервис **без состояния**: передавать `Contract`/`id` как параметры.
- Добавить транзакцию, сохранение в репозитории, возврат DTO/статуса.
- Разделить ответственность: контроллер — минимум логики.

### 4) Корректная реализация

```java
@RestController
@RequestMapping("/contract")
@RequiredArgsConstructor
public class ContractController {
    private final ContractService contractService;

    @PostMapping("/{contractId}/price")
    public ResponseEntity<Void> updatePrice(@PathVariable Long contractId) {
        contractService.updatePrice(contractId);
        return ResponseEntity.noContent().build();
    }
}

@Service
@RequiredArgsConstructor
public class ContractService {
    private final ContractRepository repository;
    private final PriceService priceService;

    @Transactional
    public void updatePrice(Long id) {
        Contract contract = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        BigDecimal price = priceService.calcPrice(contract);
        contract.setPrice(price);
        repository.save(contract);
    }
}

@Entity
class Contract {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal price;
    // getters/setters
}
```

---

## Задача 9. Высоконагруженный микросервис чеков (cron, AOP прокси, Optional, async)

### 1) Задание
- Метод `processJobRunning()` — раз в час читает необработанные возвратные чеки, отправляет событие в аналитику, сохраняет чек.
- Метод `sendReceipt()` — принимает DTO и синхронно дергает внешний `taxClient.sendReceipt()` (до 60 сек). Нужно сделать неблокирующим.

### 2) Ошибки
- `@Scheduled(cron = ""${cron.expression}"")` — лишние кавычки → должно быть `@Scheduled(cron = "${cron.expression}")`.
- `eventListener.sendEventToAnalytics(new SomeEvent(receipt.getId());` — **пропущена закрывающая скобка** + неизвестно, откуда `eventListener`.
- `mapper.map(receipt)` и потом `mapReceipt(mapper.map(receipt))` — подозрительные двойные маппинги (entity→dto→entity?). Вероятно нужна одна операция.
- `findById` используется как будто возвращает `Receipt`, а он возвращает **`Optional<Receipt>`**.
- Комментарий SQL: `'REFUNC'` — опечатка (`REFUND`).
- Вызов кэшируемого метода через `this.getReceiptSource(...)` не сработает — нужен **бин‑прокси** (self/`AopContext`).
- Отсутствуют ретраи, try/catch в цикле — одна ошибка прерывает всю пачку.
- `sendReceipt()` блокирует поток на долгом внешнем вызове.

### 3) Что сделать лучше
- Исправить cron и мелкие ошибки/опечатки.
- В `processJobRunning()` работать **батчами** (пагинация), помечать чек `processed` **после** успешной отправки события и сохранения. Ошибку логировать, чек оставить непомеченным для следующего запуска/ретрая.
- `@Transactional(readOnly = true)` для чтений, для записи — отдельная транзакция на чек.
- `sendReceipt()` сделать **асинхронным**: либо `@Async` с пулом, либо отправлять DTO в **очередь** (Kafka/RabbitMQ) и отдельный consumer вызывает налоговую. Для высоких нагрузок лучше **очередь**.
- Обработать `Optional` и AOP‑прокси (`self.getReceiptSource(...)`), либо перенести кэш на внешний `@Service`.
- Добавить таймауты/ретраи (`Resilience4j`) при вызовах.

### 4) Корректная реализация (вариант с очередью для sendReceipt)

```java
@Service
@RequiredArgsConstructor
public class SomeServiceImpl implements SomeService {
    private final TaxClient taxClient;
    private final ReceiptDao receiptDao;
    private final ApplicationEventPublisher eventPublisher; // для аналитики
    private final KafkaTemplate<String, ReceiptDto> kafka; // очередь для налоговой
    private final SomeService self; // прокси для AOP (@Cacheable)

    // раз в час: "0 0 * * * *" (пример), либо из настроек
    @Scheduled(cron = "${cron.expression}")
    public void processJobRunning() {
        int page = 0;
        int size = 500;
        Page<Receipt> receipts;
        do {
            receipts = receiptDao.findAllBySourceAndProcessedFalse(ReceiptSource.REFUND, PageRequest.of(page++, size));
            for (Receipt r : receipts) {
                try {
                    // бизнес-логика отправки события
                    eventPublisher.publishEvent(new SomeEvent(r.getId()));
                    // пометить обработанным
                    r.setProcessed(true);
                    receiptDao.save(r);
                } catch (Exception e) {
                    // логируем и продолжаем
                    // оставляем processed=false, чтобы ретраиться
                }
            }
        } while (!receipts.isEmpty());
    }

    // приходит HTTP-запрос из другого сервиса
    public void sendReceipt(ReceiptDto dto) {
        ReceiptSource source = self.getReceiptSource(dto); // через прокси, чтобы @Cacheable сработал
        dto.setSource(source);
        // вместо долгого синхронного вызова — кладем в Kafka
        kafka.send("tax-receipts", dto.getId().toString(), dto);
    }

    @Transactional(readOnly = true)
    @Cacheable(value = "receipt_type", cacheManager = "redisCacheManager", key = "#dto.id")
    public ReceiptSource getReceiptSource(ReceiptDto dto) {
        return receiptDao.findById(dto.getId())
                .map(Receipt::getSource)
                .orElseThrow(() -> new IllegalArgumentException("Receipt not found: " + dto.getId()));
    }
}

// DAO с пагинацией
public interface ReceiptDao extends JpaRepository<Receipt, Long> {
    Page<Receipt> findAllBySourceAndProcessedFalse(ReceiptSource source, Pageable pageable);
}

// Consumer очереди, который реально дергает налоговую (масштабируется независимо)
@Component
@RequiredArgsConstructor
class TaxConsumer {
    private final TaxClient taxClient;

    @KafkaListener(topics = "tax-receipts", groupId = "tax-sender")
    public void onReceipt(ReceiptDto dto) {
        // здесь могут быть ретраи/таймауты/резилиенс
        taxClient.sendReceipt(dto, dto.getSource());
    }
}
```

Если хочется быстро и просто — можно сделать `@Async`:

```java
@Service
@RequiredArgsConstructor
public class AsyncTaxService {
    private final TaxClient taxClient;

    @Async("taxExecutor") // ThreadPoolTaskExecutor bean
    public CompletableFuture<Void> sendAsync(ReceiptDto dto, ReceiptSource source) {
        taxClient.sendReceipt(dto, source);
        return CompletableFuture.completedFuture(null);
    }
}
```

---

## Задача 10. Антипаттерн «создание потоков вручную» → `@Async`/пул потоков

### 1) Задание
Дан сервис, который для каждой задачи делает `new Thread(...).start()` в цикле. Нужно исправить на `@Async`/пул потоков с управляемыми размерами и back‑pressure.

### 2) Ошибки
- Создание «голых» потоков → лишняя нагрузка на планировщик, отсутствует управление числом потоков, нет мониторинга.
- Нет таймаутов/ограничений очереди → риск OOM.
- Исключения теряются.

### 3) Что сделать лучше
- Конфигурировать `ThreadPoolTaskExecutor` с метриками, очередью, политикой отказа.
- Пометить «фоновые» методы `@Async("executorName")`.
- Возвращать `CompletableFuture`/`Mono<Void>` (reactive), чтобы можно было связывать/ждать при необходимости.

### 4) Корректная реализация

```java
@Configuration
public class AsyncConfig {
    @Bean(name = "workExecutor")
    public ThreadPoolTaskExecutor workExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(8);
        ex.setMaxPoolSize(32);
        ex.setQueueCapacity(10000);
        ex.setThreadNamePrefix("work-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.initialize();
        return ex;
    }
}

@Service
public class WorkService {
    @Async("workExecutor")
    public CompletableFuture<Void> doWorkAsync(String payload) {
        // ... работа
        return CompletableFuture.completedFuture(null);
    }

    public void submitMany(List<String> items) {
        List<CompletableFuture<Void>> futures = items.stream()
                .map(this::doWorkAsync)
                .toList();
        CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();
    }
}
```


# 10 новых задач (№10–19) — код‑ревью и корректные реализации

Формат для каждой задачи:  
**1) задание · 2) ошибки · 3) что сделать лучше · 4) корректная реализация**

---

## 11) DocumentService: if‑else → Стратегия (+ поддержка PDF)

### 1) Задание
Сделать код‑ревью текущего `DocumentService`, выделить плюсы/минусы, затем отрефакторить на паттерн **Стратегия**, добавив реализацию для чтения PDF.

### 2) Ошибки/наблюдения
**Плюсы**: простота и быстрое чтение логики.  
**Минусы**:
- Жёсткая привязка к строковым типам (`"PDF"`, `"DOCX"`, ...), риск опечаток/регистра букв.
- Нарушение **OCP**: добавление нового типа требует правки `if/else`.
- `new DocumentReader()` внутри метода — сложно тестировать/заменять.
- Возврат `null` — NPE у вызывающего кода.
- Нулевая расширяемость (нет интерфейсов/стратегий).

### 3) Что сделать лучше
- Ввести `DocumentType` (enum) с `fromString()`.
- Определить интерфейс `DocumentReaderStrategy` и реализации под каждый тип.
- Сформировать `Map<DocumentType, DocumentReaderStrategy>` через Spring (внедрение списка бинов).
- Возвращать `InputStream`/бросать осмысленное исключение (никаких `null`).

### 4) Корректная реализация

```java
public enum DocumentType {
    PDF, DOCX, XLSX;

    public static DocumentType fromString(String type) {
        return Arrays.stream(values())
            .filter(t -> t.name().equalsIgnoreCase(type))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Unsupported type: " + type));
    }
}

public interface DocumentReaderStrategy {
    DocumentType supports();
    InputStream read();
}

@Component
class PdfDocumentReader implements DocumentReaderStrategy {
    @Override public DocumentType supports() { return DocumentType.PDF; }
    @Override public InputStream read() {
        // реальная реализация (пример-заглушка):
        return new ByteArrayInputStream("PDF bytes".getBytes(StandardCharsets.UTF_8));
    }
}

@Component
class DocxDocumentReader implements DocumentReaderStrategy {
    @Override public DocumentType supports() { return DocumentType.DOCX; }
    @Override public InputStream read() {
        return new ByteArrayInputStream("DOCX bytes".getBytes(StandardCharsets.UTF_8));
    }
}

@Component
class XlsxDocumentReader implements DocumentReaderStrategy {
    @Override public DocumentType supports() { return DocumentType.XLSX; }
    @Override public InputStream read() {
        return new ByteArrayInputStream("XLSX bytes".getBytes(StandardCharsets.UTF_8));
    }
}

@Component
public class DocumentService {
    private final Map<DocumentType, DocumentReaderStrategy> readers;

    public DocumentService(List<DocumentReaderStrategy> strategies) {
        this.readers = strategies.stream()
                .collect(Collectors.toUnmodifiableMap(DocumentReaderStrategy::supports, s -> s));
    }

    public InputStream readDocument(String type) {
        DocumentType t = DocumentType.fromString(type);
        DocumentReaderStrategy s = readers.get(t);
        if (s == null) {
            throw new IllegalArgumentException("Reader not found for: " + type);
        }
        return s.read();
    }
}
```

---

## 12) Поиск человека по имени с Optional

### 1) Задание
Реализовать метод `findPersonByName(List<Person> persons, String name)`, возвращающий `Optional<Person>` первого совпадения.

### 2) Ошибки/нюансы
- В исходнике метод возвращает всегда `Optional.empty()`.
- Нужно аккуратно обработать `null`-ы (список/имя/имя у Person).
- Уточнить сравнение: строгий `equals` (без учета регистра) — зависит от требований. Оставим строгий `equals`.

### 3) Что сделать лучше
- Проверить входные параметры, не кидать NPE.
- Использовать `stream().filter(...).findFirst()`.

### 4) Корректная реализация

```java
public class PersonService {
    public Optional<Person> findPersonByName(List<Person> persons, String name) {
        if (persons == null || name == null) return Optional.empty();
        return persons.stream()
                .filter(p -> p != null && name.equals(p.getName()))
                .findFirst();
    }
}
```

---

## 13) Бронирование номера (тесты ок, в проде — косяки)

### 1) Задание
Почему код проходит тестовые стенды, но фейлится в проде? Исправить.

### 2) Ошибки
- Сравнение статуса строкой и **через `==`** (в примере «двойные кавычки в кавычках») — должно быть `equals`, лучше использовать **enum**.
- `findById(roomId)` у Spring Data обычно возвращает `Optional<Room>`, не сам `Room`.
- Нет **транзакции** и **блокировок**: гонка на уровне БД при одновременных бронированиях.
- Нет **versioning** (`@Version`) → возможны «двойные продажи» на высоких нагрузках/нескольких подах.
- Полевая инъекция `@Autowired`.
- Нет проверки, что `room` вообще найден.

### 3) Что сделать лучше
- Ввести `enum RoomStatus { VACANT, BOOKED }`.
- Использовать **пессимистическую** блокировку (`@Lock(PESSIMISTIC_WRITE)`) или **оптимистическую** (`@Version`) и ретраи.
- Обернуть в **транзакцию**.
- Фиксировать клиента из `SecurityContext` безопасно.
- Возвращать понятный результат/исключение.

### 4) Корректная реализация (вариант с пессимистической блокировкой)

```java
@Entity
class Room {
    @Id Integer id;
    @Enumerated(EnumType.STRING)
    RoomStatus status;
    String clientId;
    String roomNumber;
    @Version Long version;
}

enum RoomStatus { VACANT, BOOKED }

public interface RoomRepository extends JpaRepository<Room, Integer> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select r from Room r where r.id = :id")
    Optional<Room> findForUpdate(@Param("id") Integer id);
}

@Service
@RequiredArgsConstructor
class BookingService {
    private final RoomRepository roomRepository;

    @Transactional
    public boolean bookRoom(Integer roomId) {
        Room room = roomRepository.findForUpdate(roomId)
            .orElseThrow(() -> new IllegalArgumentException("Room not found: " + roomId));

        if (room.getStatus() == RoomStatus.VACANT) {
            room.setStatus(RoomStatus.BOOKED);
            room.setClientId(SecurityContext.getClientId());
            roomRepository.save(room);
            return true;
        }
        return false;
    }
}
```

---

## 14) Большой сервис заказов: ревью и рефакторинг

### 1) Задание
Ответить на вопросы по валидации, транзакциям, SRP и рискам; отрефакторить.

### 2) Ошибки/ответы по пунктам
- **Валидация DTO**: использовать Bean Validation (`@Valid`) на входе контроллера, аннотации на `OrderDto`/`OrderItemDto` (`@NotNull`, `@Positive`, `@Size`, `@NotBlank` и т. п.). Доменная валидация — в домене/сервисе.
- **Проверки на null**: на границах (контроллер/фасад). Внутри сервиса — вход уже валиден.
- **Что оставить в `OrderService`**: оркестрацию случаев — валидация домена, расчёт цены (лучше вынести в `PricingService`), сохранение, обновление инвентаря (лучше — `InventoryService`), уведомления — `NotificationService` асинхронно/через события.
- **Когда откатится транзакция**: по **unchecked** исключению или по чекed, если явно сконфигурировано `rollbackFor`. Сейчас откат при `RuntimeException`/`Error`.
- **SRP**: разделить на `OrderService` (оркестрация), `PricingService`, `InventoryService`, `NotificationService`.
- **Что писать в таблицу**: агрегат `Order` + `OrderItem` и их статусы/суммы/скидки. Аудит — отдельно/Outbox.
- **Риски при сохранении**: частичная запись, конкурентные изменения инвентаря, двойное уведомление. Нужна идемпотентность, блокировки/проверки, outbox для событий.
- **`saveOrder` кинул исключение**: транзакция откатится, уведомлений/изменения инвентаря не будет, если вызовы после сохранения — но лучше сохранять перед уведомлением.
- **`updateInventory` увеличиваем или уменьшаем?** — для заказа **уменьшаем** остаток.
- **`decreaseStock` делает**: уменьшает складские остатки; нужна проверка достаточного количества и атомарность.
- **`countByCustomerType` будет работать?** — да, если в `Order` есть поле `customerType`. Но вызывать из ценообразования — **плохая идея** (нагрузка). Лучше кэшировать/предрассчитывать.
- **Где писать SQL в интерфейсе?** — через `@Query` у репозитория; derive‑методы — если достаточно.
- **`findByOrderId` сформируется?** — если поле называется `orderId`. Но у JPA обычно `id`. Если поле — `id`, нужно `findById`.

### 3) Что сделать лучше
- Конструкторная инъекция, DTO‑валидация, разделение на сервисы, `enum CustomerType`, убрать «магические строки», outbox для нотификаций, транзакции вокруг модификаций, учесть конкурентность инвентаря.

### 4) Корректная реализация (упрощённый пример)

```java
enum CustomerType { REGULAR, VIP, LOYALTY }

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PricingService pricingService;
    private final NotificationService notificationService;

    @Transactional
    public void processOrder(@Valid Order order) {
        // доменная валидация (например, что есть items)
        if (order.getItems() == null || order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order items required");
        }

        BigDecimal total = pricingService.calculateTotal(order.getItems(), order.getCustomerType());
        order.setTotal(total);

        orderRepository.save(order); // 1) сохраняем заказ

        inventoryService.decreaseFor(order); // 2) уменьшаем остатки (можно в той же ТХ)

        // 3) уведомление (через outbox/событие -> async consumer)
        notificationService.notifyAsync(order.getCustomerId(), "Order accepted: " + order.getId());
    }
}

@Service
public class PricingService {
    public BigDecimal calculateTotal(List<OrderItem> items, CustomerType type) {
        BigDecimal total = items.stream()
                .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        // скидки
        switch (type) {
            case VIP -> total = total.multiply(new BigDecimal("0.90"));
            case LOYALTY -> total = total.multiply(new BigDecimal("0.85"));
            default -> {}
        }
        return total;
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    int countByCustomerType(CustomerType customerType);
    // findById уже есть у JpaRepository
}

@Repository
public interface InventoryRepository extends JpaRepository<Inventory, Long> {
    @Modifying
    @Query("update Inventory i set i.stock = i.stock - :q where i.productId = :pid and i.stock >= :q")
    int decreaseStock(@Param("pid") Long productId, @Param("q") int quantity);
}

@Service
@RequiredArgsConstructor
public class InventoryService {
    private final InventoryRepository repo;

    public void decreaseFor(Order order) {
        for (OrderItem item : order.getItems()) {
            int updated = repo.decreaseStock(item.getProductId(), item.getQuantity());
            if (updated == 0) {
                throw new IllegalStateException("Insufficient stock for product " + item.getProductId());
            }
        }
    }
}
```

---

## 15) OrderService: audit + save + исключения

### 1) Задание
Сделать ревью: транзакции, порядок операций, логирование/исключения, корректность DI, типы репозиториев.

### 2) Ошибки
- Смешение `@Autowired` и `@Inject` без нужды.
- `@Transactional` на **private** методе не сработает (self‑invocation/AOP). Должно быть на public‑методе/внешнем бине.
- `auditRepository.save(order)` — вероятно не тот тип; аудит лучше писать **после** успешного сохранения заказа (или через outbox/событие).
- Ловим `Throwable`, делаем `printStackTrace()` — плохая практика. Нужен нормальный логгер + корректные типы исключений.
- `OrderValidationException` (checked) из `@Transactional` — по умолчанию **не откатывает**. Нужен `rollbackFor` ИЛИ сделать runtime‑исключение.
- Метод `save` выбрасывает `OrderValidationException`, но также ловит `Throwable` и заворачивает в `OrderSaveException` — двусмысленность.

### 3) Что сделать лучше
- Конструкторная инъекция, логгер.
- Валидацию выполнить **до** сохранения.
- Транзакцию на public‑методе, откатывать по любым исключениям домена.
- Аудит — через событие (после коммита) или в конце успешной ТХ.

### 4) Корректная реализация

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
    private final OrderRepository orderRepository;
    private final AuditRepository auditRepository;

    @Transactional(rollbackFor = Exception.class)
    public void save(Order order) {
        validate(order); // может бросать RuntimeException/DomainException
        orderRepository.save(order);
        // аудит после успешной записи (можно вынести в @TransactionalEventListener(AFTER_COMMIT))
        auditRepository.save(new AuditRecord("ORDER_SAVED", order.getId(), Instant.now()));
    }

    private void validate(Order order) {
        if (order == null || order.getClient() == null || isBlank(order.getClient().getName())) {
            throw new OrderValidationRuntimeException("Поле клиент не заполнено");
        }
    }
}
```

---

## 16) Обновление статуса заказа и уведомления

### 1) Задание
Привести метод к чистому коду: Optional, enum статусов, транзакция, единый путь сохранения и уведомлений.

### 2) Ошибки
- `findById` возвращает `Optional`, не `Order`.
- Дублирование веток if‑else и `orderRepository.save(order)` непоследовательно.
- «Магические строки» статусов.
- Нет транзакции; уведомление шлётся до/без сохранения.
- Полевая инъекция.

### 3) Что сделать лучше
- `enum OrderStatus` и switch.
- Единственный путь `save` после установки статуса.
- Уведомление после успешного сохранения.
- Конструкторная инъекция, логирование/ошибки.

### 4) Корректная реализация

```java
enum OrderStatus { COMPLETED, CANCELLED, PENDING, IN_PROGRESS }

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    @Transactional
    public void updateOrderStatus(Long orderId, OrderStatus newStatus) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("Order not found: " + orderId));

        order.setStatus(newStatus.name());
        orderRepository.save(order);

        switch (newStatus) {
            case COMPLETED -> notificationService.notify(order.getUserId(), "Your order is completed");
            case CANCELLED -> notificationService.notify(order.getUserId(), "Your order is cancelled");
            default -> {}
        }
    }
}
```

---

## 17) Проверка возможности продажи по времени/категории

### 1) Задание
Реализовать `checkSale(Product item, Integer hour)` с правилами:
- 13:00–14:00 — запрет на продажу всего.
- 23:00–02:59 — запрет на продажу любого алкоголя.
- 03:00–07:59 — запрет на продажу крепкого алкоголя (`abv >= 40`).

### 2) Ошибки/нюансы
- Часы в диапазоне `0..23`. Нужны аккуратные интервалы, включая «переход через сутки».
- Класс `Alcohol` — абстрактный, у него есть `getAbv()`.

### 3) Что сделать лучше
- Нормализовать вход (`Objects.requireNonNull`), проверить диапазон.
- Чёткие условия интервалов, особенно «23:00–02:59».

### 4) Корректная реализация

```java
public boolean checkSale(Product item, Integer hour) {
    if (item == null || hour == null) return false;
    if (hour < 0 || hour > 23) throw new IllegalArgumentException("hour must be 0..23");

    // 13:00–13:59 — обед, запрет всем
    if (hour == 13) return false;

    boolean isAlcohol = item instanceof Alcohol;
    int abv = isAlcohol ? ((Alcohol) item).getAbv() : 0;

    // 23:00–23:59 и 00:00–02:59 — запрет для любого алкоголя
    if (isAlcohol && (hour == 23 || hour <= 2)) return false;

    // 03:00–07:59 — запрет для крепкого алкоголя (>= 40)
    if (isAlcohol && (hour >= 3 && hour <= 7) && abv >= 40) return false;

    return true;
}
```

---

## 18) Поиск товаров при большом RPS и частом `count=0`

### 1) Задание
Метод делает `count` → если >0, то `find`. 200 RPS, в 50% случаев `count=0`. Снизить нагрузку/RT.

### 2) Ошибки
- Два запроса к БД на каждый вызов, лишняя работа при `count=0`.
- Отсутствие индексов/лимитов.
- Нет таймаутов/кэша.

### 3) Что сделать лучше
- Заменить `count` на **`exists`** (нам достаточно факта наличия).
- Либо сразу выполнять `find` с **лимитом** и, если пусто — возвращать пустой список.
- Индексы по полям фильтра обязательны.
- Добавить кэширование «пустых» результатов на короткий TTL (если фильтров много и запросы повторяются).

### 4) Корректная реализация (через `exists`)

```java
class Service {
    private final Repo repo;

    public List<Product> find(Filter filter) {
        if (!repo.existsByFilter(filter)) {
            return Collections.emptyList();
        }
        return repo.findByFilter(filter); // уже полноценная выборка
    }
}
```

Либо «один запрос» с пагинацией:

```java
public List<Product> find(Filter filter) {
    List<Product> firstPage = repo.findByFilter(filter, PageRequest.of(0, 50));
    return firstPage; // если пусто — сразу пустой список
}
```

---

## 19) Синхронизация: разные мониторы, неверный main

### 1) Задание
Исправить создание монитора при каждом вызове, синтаксис вызова, сделать `main` статическим.

### 2) Ошибки
- `new Object()` внутри `doSome()` → разные мониторы → нет защиты.
- `new Service.doSome()` — синтаксическая ошибка, нужно `new Service().doSome()`.
- `main` должен быть `static`.
- `execute` не определён — для примера используем `ExecutorService`.

### 3) Что сделать лучше
- Единый `final Object lock` или `synchronized (this)`.
- Использовать пул потоков.

### 4) Корректная реализация

```java
class Main {
    public static void main(String[] args) {
        Service s = new Service();
        ExecutorService ex = Executors.newFixedThreadPool(2);
        ex.submit(s::doSome);
        ex.submit(s::doSome);
        ex.shutdown();
    }
}

class Service {
    private final Object lock = new Object();

    void doSome() {
        // ... code before
        synchronized (lock) {
            // критическая секция
        }
        // ... code after
    }
}
```

---

## 20) In‑memory хранилище пользователей под нагрузкой

### 1) Задание
Быстро поднять хранение пользователей в памяти, учесть потокобезопасность и нагрузку.

### 2) Ошибки
- `HashMap` не потокобезопасен под конкурентной записью/чтением.
- Возврат мутабельных объектов без копий → можно сломать кэш извне.
- Нет ограничений по памяти/TTL/метрик.

### 3) Что сделать лучше
- Использовать `ConcurrentHashMap`.
- Сделать `User` **иммутабельным** (record) или возвращать копии.
- Возвращать `Optional<User>`.
- Для прод‑нагрузки: `Caffeine`/`Spring Cache` с лимитом по размеру и TTL, метрики, эвикшены.
- При росте требований — вынести в внешнее хранилище (Redis/DB) + кэш.

### 4) Корректная реализация (базовая, потокобезопасная)

```java
@Service
public class UserService {
    private final ConcurrentMap<Integer, User> users = new ConcurrentHashMap<>();

    public void addUser(User user) {
        Objects.requireNonNull(user);
        users.put(user.getId(), user);
    }

    public Optional<User> getUser(int id) {
        return Optional.ofNullable(users.get(id));
    }

    public List<User> getAllUsers() {
        // защитная копия
        return List.copyOf(users.values());
    }

    public void removeUser(int id) {
        users.remove(id);
    }
}
```

Для продакшена — вариант с Caffeine:

```java
@Configuration
public class CacheConfig {
    @Bean
    public Cache<Integer, User> userCache() {
        return Caffeine.newBuilder()
                .maximumSize(100_000)
                .expireAfterAccess(Duration.ofMinutes(30))
                .recordStats()
                .build();
    }
}

@Service
@RequiredArgsConstructor
public class UserService {
    private final Cache<Integer, User> cache;

    public void addUser(User u) { cache.put(u.getId(), u); }
    public Optional<User> getUser(int id) { return Optional.ofNullable(cache.getIfPresent(id)); }
    public List<User> getAllUsers() { return cache.asMap().values().stream().toList(); }
}
```


Вот продолжение — **задачи №20–22** (нумерация продолжается от ваших прошлых 10–19). По каждой:  
**1) задание · 2) подробный разбор ошибок · 3) как сделать лучше · 4) корректная реализация (пример кода)**

---

# 20) `DocumentService`: проверка принят/завершён (строки, N+1 и кэширование)

### 1) Задание
Есть сервис с двумя методами:
- `isAcceptedDocument(Document)` — документ считается принятым, если *код текущего статуса* входит в список положительных кодов.
- `isFinishedDocument(Document)` — обработка завершена, если среди кодов документа есть статус, помеченный как финальный (берём из БД).

Нужно провести код-ревью, найти проблемы и отрефакторить.

### 2) Ошибки и узкие места
- **Сравнение строк через `==`**:  
  В `if (document.getCurrentStatus().getCode() == status)` сравниваются ссылки, а не значения. Это почти гарантированная логическая ошибка. Нужен `equals`.
- **«Магические значения» и смешение форматов**:  
  Список `["500","293","216","processed"]` смешивает числовые коды и строковые «говорящие» значения. Это повышает риск ошибок и несогласованности с БД/бизнес-правилами.
- **NPE-риски**:  
  `document`, `document.getCurrentStatus()`, `getCode()` и `document.getCodes()` могут быть `null`. Сейчас это никак не обрабатывается.
- **N+1 запрос к БД**:  
  В `isFinishedDocument` на каждый `docStatusCode` вызывается `statusRepository.findByCode(...)` → для документов с длинной историей статусов будет большое число запросов. Нужна пакетная загрузка `findAllByCodeIn(...)` (один запрос).
- **Плохая расширяемость**:  
  Локальный список «положительных» кодов вшит в метод. При смене бизнес-правил придётся перекомпилировать сервис. Лучше: конфигурация, справочник в БД, кэш.
- **Инъекция полями** (`@Autowired` на поле): хуже тестируемость и явность зависимостей. Предпочтительна конструкторная инъекция.
- **Нет кэширования статусов**:  
  Если `status.isFinal()` берётся из справочника, набор финальных кодов меняется редко — его можно кэшировать.

### 3) Что сделать лучше
- Перейти на **конструкторную инъекцию**.
- Для «принят» держать **набор кодов** из конфигурации/БД и сверять через `Set.contains`.  
  Ещё лучше — ввести **enum StatusCode** и приводить вход к enum (или хотя бы нормализовать строки).
- В `isFinishedDocument` — **пакетный запрос** `findAllByCodeIn(codes)` или кэшированный набор «финальных» кодов `findAllByIsFinalTrue()`.
- Обработать `null` безопасно и явно.
- (Опционально) Добавить кэш/TTL на «финальные» коды, если справочник меняется редко.

### 4) Корректная реализация (пример)

```java
@Service
@RequiredArgsConstructor
public class DocumentService {

    private final StatusRepository statusRepository;

    // Лучше вынести в конфиг/БД. Здесь — для примера.
    private static final Set<String> ACCEPTED_CODES = Set.of("500", "293", "216", "processed");

    /**
     * Документ принят, если код текущего статуса входит в набор положительных кодов.
     */
    public boolean isAcceptedDocument(Document document) {
        if (document == null) return false;
        Status current = document.getCurrentStatus();
        if (current == null) return false;

        String code = current.getCode();
        if (code == null) return false;

        return ACCEPTED_CODES.contains(code);
    }

    /**
     * Обработка завершена, если среди кодов документа есть финальный.
     * Важно: без N+1 — грузим статусы пакетно.
     */
    @Transactional(readOnly = true)
    public boolean isFinishedDocument(Document document) {
        if (document == null) return false;
        List<String> codes = document.getCodes();
        if (codes == null || codes.isEmpty()) return false;

        // Репозиторий должен уметь пакетно:
        // List<Status> findAllByCodeIn(Collection<String> codes);
        List<Status> statuses = statusRepository.findAllByCodeIn(codes);

        return statuses.stream().anyMatch(Status::isFinal);
    }

    // Альтернатива: кэшировать финальные коды (если справочник редкий)
    // @Cacheable("final-status-codes")
    // public Set<String> finalCodes() { ... }
    // public boolean isFinishedDocument(Document doc) {
    //     Set<String> finals = finalCodes();
    //     return doc.getCodes().stream().anyMatch(finals::contains);
    // }
}
```

---

# 21) `CachedPhotosService`: кэширование ресайза (Optional, ключ кэша, типы)

### 1) Задание
Сервис ресайза фото, результат кэшируем. Провести ревью и исправить.

### 2) Ошибки и узкие места
- **`findById` возвращает `Optional`** у Spring Data, а в коде — `Photo photo = photoRepository.findById(photoId);` (как будто возвращает сам объект). Необработанный «фото не найдено».
- **Несоответствие типов**: переменная `PhotoDTO` vs `PhotoDto`; также `ConvertionUtils` — вероятная опечатка (`ConversionUtils`).
- **Инъекция полями** → лучше конструкторная.
- **Ключ кэша**: по умолчанию Spring Cache использует все аргументы (`SimpleKey(photoId, width, height)`) — это ок. Но если `photoId` большой объект или есть шанс налёта на разные вызовы, можно задать явный ключ (строка).
- **Семантика ошибок/валидации**: если валидация не прошла — бросить понятное исключение. Если фото не найдено — тоже. `null` возвращать нельзя.
- **Жизненный цикл кэша**: нет политики TTL/объёма. Для продовой нагрузки нужны лимиты (например, Caffeine/Redis/TTL).
- **Сериализация**: если кэш — распределённый, убедиться, что `PhotoDto` сериализуемый или используем JSON/byte[].

### 3) Что сделать лучше
- Конструкторная инъекция, аккуратные типы и названия классов.
- Обрабатывать отсутствие фото через 404-подобное исключение (или `Optional` на уровне контроллера).
- Явный ключ кэша (опционально).
- Настроить кэш-менеджер: размер, TTL, сериализация.
- Добавить `@Transactional(readOnly = true)` (если есть ленивые связи и нам нужна их подгрузка в одном контексте чтения).

### 4) Корректная реализация (пример)

```java
@Service
@RequiredArgsConstructor
public final class CachedPhotosService {

    public static final String RESIZED_PHOTO_CACHE_NAME = "RESIZED_PHOTO_CACHE_NAME";

    private final PhotoRepository photoRepository;
    private final PhotoValidationService photoValidationService;
    private final PhotoOperations photoOperations;

    @Cacheable(
        cacheNames = RESIZED_PHOTO_CACHE_NAME,
        key = "#photoId + ':' + #width + 'x' + #height"
        // condition / unless можно добавить по требованиям
    )
    @Transactional(readOnly = true)
    public PhotoDto resizedPhoto(String photoId, int width, int height) {
        photoValidationService.validateSize(width, height);

        Photo photo = photoRepository.findById(photoId)
            .orElseThrow(() -> new PhotoNotFoundException("Photo not found: " + photoId));

        PhotoDto photoDto = ConversionUtils.convert(photo);
        return photoOperations.resize(photoDto, width, height);
    }
}
```

> В проде дополнительно: `@CacheEvict` при обновлении/удалении фото; `Caffeine`/`Redis` с TTL/максимальным размером; метрики кэша.

---

# 22) `PizzaService`: транзакции, логирование, JPA-сущность

### 1) Задание
Сервис оформляет заявку на пиццу и проверяет готовность. Провести ревью, объяснить почему «проходит стенды, но косячит в проде», и исправить.

### 2) Ошибки и причины прод-косяков
- **Опечатки и стиль**: `PizzaSevice` (должно быть `PizzaService`), строковая конкатенация в логах вместо плейсхолдеров.
- **`getOne` устарел**: `userRepository.getOne(userId)` в новых Spring Data заменяется на `getReferenceById` или чаще на `findById` (с `Optional`). В текущем коде есть проверка `if (user == null)`, но `getOne` возвращал прокси и кидал `EntityNotFoundException` при обращении — поведение *отличается* между стендами/БД и продом.
- **Возврат `null` как сигнал об ошибке**: крайне нежелательно. Лучше `Optional`/исключение.
- **Переменная `userId` в `fillValues`**: метода нет аргумента `userId`, но он используется — это не скомпилируется.
- **`@Transactional` на `private` методе**: Spring AOP не применит транзакцию при self-вызове приватного метода. На тестах это легко не заметить (меньше конкуренции, другая БД), а в проде возникнут частичные записи: `save()` выполнится, затем исключение («нет пепперони») — и запись не откатится.
- **Плохая модель сущности**:
  - Внутренний не-static класс `PizzaRequest` с `@Entity` — JPA так нельзя (должен быть top-level или `static` nested).
  - `@Id` отсутствует, `final UUID pizzaId` без сеттера и без `@Id`.
  - Нет `@NoArgsConstructor` — JPA требует.
- **Грубое глушение ошибок**: в `isPizzaReady` ловится `Exception` и возвращается `false` без логирования/дифференциации — теряются причины, трудно поддерживать.
- **Точность денежных сумм**: `double price` — для денег лучше `BigDecimal`.

### 3) Что сделать лучше
- Переименовать в `PizzaService`, использовать **конструкторную инъекцию** и параметризованные логи `log.info("...", arg)`.
- Заменить `getOne` на `findById().orElseThrow(...)`.
- Сделать **публичную транзакцию** вокруг всего процесса заявки (`requestPizza`) — тогда исключение про пепперони откатит `save`. Либо проверять «пепперони недоступна» **до** сохранения (дешевле для БД).
- Вынести сущность `PizzaRequest` в **отдельный top-level класс**, добавить `@Id`, `@NoArgsConstructor`, сделать поля совместимыми с JPA.
- Для денег — `BigDecimal`.
- В `isPizzaReady` — логировать и решать: либо бросать исключение наверх, либо возвращать `Optional<Boolean>`/статус с деталями.
- Проверки на `null` — на границе API (контроллер). В сервис вход должен быть валиден.

### 4) Корректная реализация (пример)

```java
// Сущность — отдельным классом
@Entity
@Table(name = "pizza_request")
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
public class PizzaRequest {

    @Id
    private UUID id;

    @Column(nullable = false)
    private UUID userId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PizzaType pizzaType;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    private PizzaMaker pizzaMaker;
}

// Репозиторий
public interface PizzaRequestRepository extends JpaRepository<PizzaRequest, UUID> {}

// Сервис
@Component
@RequiredArgsConstructor
@Slf4j
public class PizzaService {

    private final UserRepository userRepository;
    private final PizzaRequestRepository pizzaRequestRepository;
    private final KitchenService kitchenService;

    /**
     * Создаём заявку. Если выбрана недоступная пицца — бросаем исключение,
     * ТХ откатывается (ниже — RuntimeException).
     */
    @Transactional
    public UUID requestPizza(UUID userId, PizzaType pizzaType, BigDecimal price) {
        log.info("request pizza: userId={}", userId);

        var user = userRepository.findById(userId)
            .orElseThrow(() -> new IllegalArgumentException("User not found: " + userId));

        // Можно «отсечь» сразу — до сохранения (дешевле для БД)
        if (pizzaType == PizzaType.PEPPERONI) {
            throw new IllegalStateException("Pepperoni is not available");
        }

        var pizzaRequest = new PizzaRequest();
        pizzaRequest.setId(UUID.randomUUID());
        pizzaRequest.setUserId(user.getId());
        pizzaRequest.setPizzaType(pizzaType);
        pizzaRequest.setPrice(price);

        pizzaRequestRepository.save(pizzaRequest);

        log.info("request created: pizzaId={}", pizzaRequest.getId());
        return pizzaRequest.getId();
    }

    @Transactional(readOnly = true)
    public boolean isPizzaReady(UUID pizzaId) {
        log.info("check pizza: id={}", pizzaId);

        var pizzaRequest = pizzaRequestRepository.findById(pizzaId)
            .orElseThrow(() -> new IllegalArgumentException("PizzaRequest not found: " + pizzaId));

        try {
            PizzaStatus status = kitchenService.getStatus(pizzaRequest);
            log.info("status={}", status);
            return status == PizzaStatus.READY;
        } catch (KitchenServiceException e) {
            log.warn("Kitchen status failed for {}: {}", pizzaId, e.getMessage(), e);
            // либо пробрасываем дальше, либо возвращаем false по бизнес-требованию:
            return false;
        }
    }
}
```

> Почему на тестах «всё ок», а в проде — нет?  
> Часто на тестах:
> - используется `@Transactional` на самих тестовых методах (всё откатывается «за кадром»);
> - другая БД (H2), где поведение ленивых прокси/исключений другое;
> - низкая конкуренция потоков.  
> В проде `@Transactional` на `private` методе реально **не сработает** (self-invocation), поэтому сохранение пройдёт, а исключение после — не откатит запись.

---






