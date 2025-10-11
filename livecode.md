
# Разбор 22 задач для собеседований Senior Java (с исходными кодами)
Формат по каждой задаче:  
**1) Задание** · **Исходный код** · **2) Ошибки/риски (подробно)** · **3) Что сделать лучше** · **4) Корректная реализация (пример)**

> Примечание: для первых **20 задач** описания ошибок и улучшений расширены и детализированы.

---

## 1) REST + БД + Kafka: создание заказа и публикация события

### 1) Задание
Контроллер принимает заказ, сервис сохраняет его в БД и отправляет событие в Kafka. Найти проблемы и показать корректный подход (надёжность, согласованность, валидация, идемпотентность).

### Исходный код
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public ResponseEntity<String> createOrder(@RequestBody OrderRequest request) {
        orderService.processOrder(request);
        return ResponseEntity.ok("ok");
    }
}

@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void processOrder(OrderRequest request) {
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setTotal(request.getTotal());
        order.setStatus("NEW");
        orderRepository.save(order);

        kafkaTemplate.send("orders", String.valueOf(order.getId()), "order_created");
    }
}

@Entity
@Data
public class Order {
    @Id
    @GeneratedValue
    private long id;

    private String userId;
    private String status;
    private BigDecimal total;
}

@Data
public class OrderRequest {
    private String userId;
    private BigDecimal total;
}
```

### 2) Ошибки/риски
- **Полевая инъекция** (`@Autowired` на полях) — ухудшает тестируемость и явность зависимостей.
- **Нет валидации входных данных**: `userId` пустой, `total` отрицательный — всё пройдёт.
- **Ответ `200 OK "ok"`** — неинформативно; правильнее `201 Created` + `Location`/DTO с `id`.
- **Событие в Kafka — просто строка `"order_created"`**: нет полезной нагрузки, схемы/версии, ключа; трудно эволюционировать и отлаживать.
- **Несогласованность БД ↔ Kafka**: при сбое между `save()` и `send()` возможны записи без события (или наоборот при ретраях). Нужен **Outbox** или транзакции Kafka (сложно).
- **Нет обработки ошибок/ретраев** при отправке; нет идемпотентности на потребителе.
- **Модель денег**: `BigDecimal` без контроля `scale/precision`; возможно хранить в minor units (копейки) или задать `precision/scale` в колонке.
- **Статус "NEW" строкой** — лучше `enum`.
- **Отсутствие аудита/корреляции** — для трассировки событий (idempotency key, correlationId).

### 3) Что сделать лучше
- Использовать **конструкторную инъекцию**.
- Добавить **Bean Validation** на `OrderRequest` и в контроллере (`@Valid`).
- Возвращать `201 Created` + DTO/Location.
- Событие — JSON с полями (`orderId`, `eventType`, `schemaVersion`, `userId`, `total`).
- Обеспечить согласованность: **Outbox pattern** (рекомендуется) — в одной транзакции с заказом записываем событие в таблицу `outbox`, отдельный паблишер отправляет в Kafka и помечает `sent=true`.
- Логировать ключевые шаги; добавить метрики.
- Подумать про идемпотентность на уровне потребителей.

### 4) Корректная реализация (Outbox, сокращённый пример)
```java
// DTO
public record OrderRequest(@NotBlank String userId,
                           @NotNull @PositiveOrZero BigDecimal total) {}
public record OrderResponse(Long id) {}

// Entity
@Entity @Table(name="orders")
public class Order {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
  @Column(nullable=false) private String userId;
  @Column(nullable=false, precision=19, scale=2) private BigDecimal total;
  @Enumerated(EnumType.STRING) @Column(nullable=false) private Status status;
  public enum Status { NEW, PAID }
  // getters/setters
}

@Entity @Table(name="outbox_events")
public class OutboxEvent {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
  @Column(nullable=false) private String aggregateType; // "Order"
  @Column(nullable=false) private Long aggregateId;     // orderId
  @Column(nullable=false) private String eventType;     // "ORDER_CREATED"
  @Column(nullable=false, columnDefinition="text") private String payload;
  @Column(nullable=false) private boolean sent = false;
  @Column(nullable=false) private Instant createdAt = Instant.now();
  // getters/setters
}

@Service
@RequiredArgsConstructor
public class OrderService {
  private final OrderRepository orderRepo;
  private final OutboxRepository outboxRepo;
  private final ObjectMapper mapper;

  @Transactional
  public Long createOrder(OrderRequest req) {
    Order o = new Order();
    o.setUserId(req.userId());
    o.setTotal(req.total());
    o.setStatus(Order.Status.NEW);
    o = orderRepo.save(o);

    Map<String,Object> evt = Map.of(
      "orderId", o.getId(), "eventType","ORDER_CREATED",
      "schemaVersion",1, "userId", o.getUserId(), "total", o.getTotal()
    );
    String payload = mapper.writeValueAsString(evt);
    OutboxEvent out = new OutboxEvent();
    out.setAggregateType("Order");
    out.setAggregateId(o.getId());
    out.setEventType("ORDER_CREATED");
    out.setPayload(payload);
    outboxRepo.save(out);
    return o.getId();
  }
}

@RestController @RequestMapping("/orders") @RequiredArgsConstructor
public class OrderController {
  private final OrderService svc;
  @PostMapping
  public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderRequest req, UriComponentsBuilder uri) {
    Long id = svc.createOrder(req);
    return ResponseEntity.created(uri.path("/orders/{id}").build(id)).body(new OrderResponse(id));
  }
}

// Паблишер батчами
@Component @RequiredArgsConstructor
public class OutboxPublisher {
  private final OutboxRepository repo;
  private final KafkaTemplate<String,String> kafka;
  @Scheduled(fixedDelayString="${outbox.delay-ms:1000}")
  @Transactional
  public void publishBatch() {
    List<OutboxEvent> batch = repo.findTop100BySentFalseOrderByIdAsc();
    for (OutboxEvent e: batch) {
      kafka.send("orders", e.getAggregateId().toString(), e.getPayload())
        .completable().whenComplete((md,ex)-> { if (ex==null) markSent(e.getId()); });
    }
  }
  @Transactional public void markSent(Long id) { repo.findById(id).ifPresent(o->{o.setSent(true); repo.save(o);}); }
}
```

---

## 2) Транзакции + ФС: переименование файла и обновление БД (вариант 1)

### 1) Задание
Нужно переименовать файл на диске и обновить запись в БД так, чтобы не было рассинхрона.

### Исходный код
```java
@Transactional
public void process(String oldName, String newName) { 
    Long id = exec("select id from file where name='" + oldName + "'"); //выполнение запроса к БД 
    insert 
    processFile(oldName, newName); //переименование файла на диске
    exec("update file set name='" + newName + "' where id = " + id);  
}
```

### 2) Ошибки/риски
- **SQL-инъекции**: конкатенация входных значений.
- **Лишний `insert`** — похоже на мусор/опечатку.
- **Файловая система не откатывается транзакцией БД**: при rollback файл уже переименован → несогласованность.
- **Порядок шагов**: сначала ФС, потом БД — при падении update БД останемся с неправильным именем.
- **Нет блокировок/конкуренции**: гонки при одновременных запросах.
- **Нет обработки ошибок и компенсации**.

### 3) Что сделать лучше
- Параметризованные запросы (`JdbcTemplate`, JPA).
- Сперва проверить и **заблокировать запись** (пессимистическая/оптимистическая блокировка).
- Сделать **атомарный move** на ФС; затем **короткую транзакцию** обновления БД; при неудаче — **компенсировать** rename обратно.
- Логирование, метрики.

### 4) Корректная реализация
```java
@Service @RequiredArgsConstructor
public class FileService {
  private final JdbcTemplate jdbc;

  public void process(String oldName, String newName) {
    Long id = jdbc.queryForObject(
      "select id from file where name = ? for update skip locked",
      Long.class, oldName);
    if (id == null) throw new IllegalArgumentException("File not found: " + oldName);

    Path oldP = Path.of("/data", oldName);
    Path newP = Path.of("/data", newName);

    try { Files.move(oldP, newP, StandardCopyOption.ATOMIC_MOVE); }
    catch (IOException e) { throw new IllegalStateException("FS rename failed", e); }

    try { jdbc.update("update file set name = ? where id = ?", newName, id); }
    catch (DataAccessException e) {
      try { Files.move(newP, oldP, StandardCopyOption.ATOMIC_MOVE); } catch (IOException ignored) {}
      throw e;
    }
  }
}
```

---

## 3) Класс `Cat4`: потокобезопасность, JDBC, finalize, toString

### 1) Задание
Исправить ошибки в многопоточности, работе с БД и вспомогательных методах.

### Исходный код
```java
public final class Cat4 {

    private static final int jumpsCount = 0;

    private final DataSource dataSource;
    private final String name;

    public Cat4(String name, DataSource dataSource) {
        this.name = name;
        this.dataSource = dataSource;
    }

    public void doJumps(int jumpsCount) {
        for (int i = 0; i < jumpsCount; i++) {
            new Thread(new Runnable() {
                public void run() {
                    doJump();
                }
            }).start();
        }
    }

    public void doJump() {
        jumpsCount++;
        Logger.getLogger(Cat4.class.getName()).fine("Jump!");
    }

    public void doMeow() {
        Logger.getLogger(Cat4.class.getName()).fine("Meow!");
    }

    public void doQuery(byte[] parameters) throws SQLException {
        Connection conn = null;
        Statement stmt = null;
        try {
            conn = dataSource.getConnection();
            stmt = conn.createStatement();
            stmt.execute("insert into cats (name) values(" + new String(parameters) + ")");
        } finally {
            if (stmt != null) {
                stmt.close();
            }
            if (conn != null) {
                conn.close();
            }
        }
    }

    protected int getJumpsCount() {
        int result = jumpsCount;
        jumpsCount = 0;
        return result;
    }

    public void finalize() {
        jumpsCount = 0;
    }

    @Override
    public boolean equals(Object otherCat) {
        if (otherCat == this) {
            return true;
        }
        if (!(otherCat instanceof Cat4)) {
            return false;
        }
        Cat4 cat4 = (Cat4) otherCat;
        return name.equals(cat4.name);
    }

    @Override
    public String toString() {
        try {
            return "Cat4{" +
                "name='" + name + '\'' +
                ", url=" + dataSource.getConnection().getMetaData().getURL() +
                '}';
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

### 2) Ошибки/риски
- `private static final int jumpsCount = 0;` — **final** и инкремент → не скомпилируется; и это **статик**-счётчик на все экземпляры.
- Создание «голых» потоков в цикле → антипаттерн, нет контроля пула/нагрузки/ошибок.
- SQL-инъекция: конкатенация `new String(parameters)` в SQL.
- Ресурсы лучше в `try-with-resources`.
- `toString()` не должен ходить в БД (побочные эффекты/исключения).
- `finalize()` устарел и бесполезен; не гарантирован к вызову.
- `getJumpsCount()` не атомарен; гонки.
- `equals()` без `hashCode()` — нарушает контракт.

### 3) Что сделать лучше
- Использовать `AtomicInteger` для счётчика (инстанс-поле).
- `ExecutorService`/`@Async` вместо `new Thread`.
- `PreparedStatement` и корректная кодировка (UTF-8).
- Убрать логику БД из `toString()`; удалить `finalize()`.
- Добавить `hashCode()`.
- Логгер как поле.

### 4) Корректная реализация
```java
public final class Cat4 {
  private static final Logger log = Logger.getLogger(Cat4.class.getName());

  private final DataSource dataSource;
  private final String name;
  private final AtomicInteger jumps = new AtomicInteger(0);
  private final ExecutorService executor;

  public Cat4(String name, DataSource ds, ExecutorService executor) {
    this.name = Objects.requireNonNull(name);
    this.dataSource = Objects.requireNonNull(ds);
    this.executor = Objects.requireNonNull(executor);
  }

  public void doJumps(int n) {
    for (int i=0; i<n; i++) executor.submit(this::doJump);
  }

  private void doJump() {
    jumps.incrementAndGet();
    log.fine("Jump!");
  }

  public void doMeow(){ log.fine("Meow!"); }

  public void doQuery(byte[] parameters) throws SQLException {
    try (Connection c = dataSource.getConnection();
         PreparedStatement ps = c.prepareStatement("insert into cats(name) values (?)")) {
      ps.setString(1, new String(parameters, StandardCharsets.UTF_8));
      ps.executeUpdate();
    }
  }

  protected int getAndResetJumps(){ return jumps.getAndSet(0); }

  @Override public boolean equals(Object o){
    if (this==o) return true;
    if (!(o instanceof Cat4)) return false;
    return name.equals(((Cat4)o).name);
  }
  @Override public int hashCode(){ return Objects.hash(name); }

  @Override public String toString(){ return "Cat4{name='%s'}".formatted(name); }
}
```

---

## 4) Класс «Банкомат»: загрузка/выгрузка/остаток/выдача

### 1) Задание
Исправить синтаксические ошибки и реализовать методы загрузки, выгрузки, расчёта остатка и выдачи суммы.

### Исходный код
```java
/*
Спроектировать класс "Банкомат", хранящий купюры в различных номиналах.,
Предусмотреть методы загрузки, выгрузки, расчета остатка, выдачи суммы клиенту (в номинальном разрезе)*/,
public class Atm {
    Hashtable<Short, Integer> denominations; // номиналы /
    public void load(Hashtable<Short, Integer> denominations) {
        this.denominations = new Hashtable<>();
        Object[]keys = denominations,keySet().toArray();
        for (int i = 0; i < keys.length; i++) {
            this.denominations.put((Short)keys[i], denominations.get((Short)keys[i]));
        }
    }

//Выгружает банкноты, оставляя пустую емкость.
//@return карта "номинал" -> число выгруженных купюр
public Hashtable<Short, Integer> unload() {
    Hashtable<Short, Integer> result = new Hashtable<>();
    Object[] keys = denominations.keySet().toArray();
        for (int i = 0; i < keys.length; i++) {
            result.put((Short)keys[i], denominations.get((Short)keys[i]));
        }
        return result;
    }


//Рассчитывает остаток в банкомате.
@return*/
public Integer getRest() {
    int result = new Integer(0);
    Object[] keys = denominations.keySet().toArray();,

        for (int i = 0; i < keys.length; i++) {
            result += (Integer)denominations.get((Short)keys[i]) * (Short)keys[i];
        }
        return result;
    }
}
public HashTable<Short, Integer> dispense (Integer amount) {
//TODO
return null;
}
```

### 2) Ошибки/риски
- Синтаксис (`denominations,keySet`, лишние запятые/комментарии).
- `Hashtable` — устаревшее; лучше `Map`/`ConcurrentMap` по необходимости.
- Нет неизменяемости: внешние ссылки могут менять состояние.
- `getRest()` — автоупаковка/переполнение, `Integer` вместо `long`.
- Нет алгоритма выдачи (жадный по убыванию номиналов, проверка достаточности).
- Не синхронизировано (если нужен потокобезопасный ATM).

### 3) Что сделать лучше
- `NavigableMap<Integer,Integer>` по убыванию номиналов.
- Возвращать копии структур.
- Остаток считать в `long`.
- Реализовать выдачу: жадно (если номиналы «правильные»), иначе динамику.
- (Опционально) `synchronized` методы или внешний лок.

### 4) Корректная реализация
```java
public class Atm {
  private final NavigableMap<Integer,Integer> notes = new TreeMap<>(Comparator.reverseOrder());

  public synchronized void load(Map<Integer,Integer> add){
    Objects.requireNonNull(add);
    add.forEach((d,c)->{
      if (d<=0 || c<=0) throw new IllegalArgumentException("Bad denom/count");
      notes.merge(d,c,Integer::sum);
    });
  }

  public synchronized Map<Integer,Integer> unload(){
    Map<Integer,Integer> copy = new LinkedHashMap<>(notes);
    notes.clear();
    return copy;
  }

  public synchronized long getRest(){
    long sum=0;
    for (var e: notes.entrySet()) sum += (long)e.getKey()*e.getValue();
    return sum;
  }

  public synchronized Map<Integer,Integer> dispense(int amount){
    if (amount<=0) throw new IllegalArgumentException("amount<=0");
    if (amount>getRest()) throw new IllegalStateException("Insufficient funds");

    Map<Integer,Integer> toGive = new LinkedHashMap<>();
    int rem = amount;
    for (var e: notes.entrySet()){
      int denom=e.getKey(), avail=e.getValue();
      int need = rem/denom;
      if (need>0){
        int give = Math.min(need, avail);
        if (give>0){
          toGive.put(denom, give);
          rem -= give*denom;
        }
      }
    }
    if (rem!=0) throw new IllegalStateException("Cannot compose exact amount");
    toGive.forEach((d,c)-> notes.computeIfPresent(d,(k,v)-> v-c));
    notes.entrySet().removeIf(e-> e.getValue()==0);
    return toGive;
  }
}
```

---

## 5) Production `Cat4`: кэш, потоки, JDBC

### 1) Задание
Найти проблемы (даже если «работает в проде») и предложить рефакторинг.

### Исходный код
```java
public class Cat4 {
    private final ConcurrentHashMap<byte[], BigDecimal> CACHE = new ConcurrentHashMap<>();
    public int jumpsCount = 0;
    private Cat4Profile cat4Profile;
    private final DataSource dataSource;

    public Cat4(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void doRandomJumps(int maxJumps) {
        Random rnd = new Random();
        int jumpsToDo = Math.abs(rnd.nextInt()) % maxJumps;
        for (int i = 0; i < jumpsToDo; i++) {
            new Thread(() -> {
                doJump();
            }).start();
        }
    }

    public void setCat4Profile(Cat4Profile cat4Profile) {
        this.cat4Profile = cat4Profile;
    }

    public String getCat4Name() {
        try {
            return this.cat4Profile.getCatName();
        } catch (NullPointerException e) {
            return "Max";
        }
    }

    public void doJump() {
        this.jumpsCount++;
        Logger.getLogger(Cat4.class.getName()).fine("Jump!");
    }

    public void doMeow() {
        Logger.getLogger(Cat4.class.getName()).fine("Meow!");
    }

    public BigDecimal doQuery(byte[] parameters) throws SQLException {
        Connection conn = null;
        Statement stmt = null;
        try {
            conn = dataSource.getConnection();
            stmt = conn.createStatement();
            ResultSet resultSet = stmt.executeQuery("select weight from Cat where name = '" + new
                String(parameters) + "')");

            resultSet.next();
            return resultSet.getBigDecimal("weight");
        } finally {
            if (stmt != null) {
                stmt.close();
            }
            if (conn != null) {
                conn.close();
            }
        }
    }

    public BigDecimal doQueryCached(byte[] parameters) throws SQLException {
        if (CACHE.get(parameters) != null)
            return CACHE.get(parameters);

        BigDecimal weight = doQuery(parameters);
        CACHE.put(parameters, weight);
        return weight;
    }

    public int getJumpsCount() {
        int result = jumpsCount;
        jumpsCount = 0;
        return result;
    }

    public void setJumpsCount() {
        this.jumpsCount++;
    }
}
```

### 2) Ошибки/риски
- **Ключ кэша `byte[]`** сравнивается по ссылке — кэш фактически бесполезен. Нужен иммутабельный ключ (`String`, `ByteBuffer`, хеш).
- **`new Thread` в цикле** — антипаттерн, нет управления ресурсами/нагрузкой/исключениями.
- Непотокобезопасный `jumpsCount` (гонки).
- `getCat4Name()` ловит `NullPointerException` как ветку логики — лучше `Optional`/явная проверка.
- SQL-инъекция и синтаксическая ошибка в запросе (лишняя `)`).
- `resultSet.next()` без проверки результата.
- Два чтения кэша подряд (`get`/`get`) → не атомарно; лучше `computeIfAbsent`.
- Потенциальная утечка памяти: кэш без лимитов.
- Нет логирования ошибок на уровне JDBC.

### 3) Что сделать лучше
- Ключ кэша — `String` (UTF-8) или хеш.
- Пул потоков/`@Async`.
- `AtomicInteger` для счётчика.
- Явная проверка профиля.
- `PreparedStatement`, параметризованные запросы, проверка `rs.next()`.
- `computeIfAbsent` или `@Cacheable` (Caffeine/Redis) с TTL/лимитами.
- Логирование исключений.

### 4) Корректная реализация (с Spring Cache-подходом)
```java
@Service @RequiredArgsConstructor
public class Cat4 {
  private static final Logger log = Logger.getLogger(Cat4.class.getName());
  private final DataSource dataSource;
  private final TaskExecutor taskExecutor;
  private final AtomicInteger jumps = new AtomicInteger(0);
  @Setter private Cat4Profile cat4Profile;

  public void doRandomJumps(int maxJumps){
    int n = ThreadLocalRandom.current().nextInt(maxJumps+1);
    for (int i=0;i<n;i++) taskExecutor.execute(this::doJump);
  }

  private void doJump(){ jumps.incrementAndGet(); log.fine("Jump!"); }

  public String getCat4Name(){
    return Optional.ofNullable(cat4Profile).map(Cat4Profile::getCatName).orElse("Max");
  }

  public BigDecimal doQuery(String name) throws SQLException {
    try (Connection c = dataSource.getConnection();
         PreparedStatement ps = c.prepareStatement("select weight from Cat where name = ?")) {
      ps.setString(1, name);
      try (ResultSet rs = ps.executeQuery()){
        if (!rs.next()) throw new IllegalArgumentException("Cat not found: "+name);
        return rs.getBigDecimal("weight");
      }
    }
  }

  @Cacheable(value = "catWeight", key = "#name", cacheManager = "redisCacheManager")
  public BigDecimal doQueryCached(String name) throws SQLException { return doQuery(name); }

  public int getAndResetJumps(){ return jumps.getAndSet(0); }
}
```

---

## 6) Обработка документов по типу: switch → стратегия

### 1) Задание
Вынести логику из `switch` в стратегии.

### Исходный код
```java
public class RefEx {
    public enum DocumentType { XML, PDF, DOCX }
    public static class Document { String id; DocumentType type; String content; }
    public static class DocumentService {
        public void process(Document[] d) {
            for (Document i : d) {
                // Общая логика обработки документа
                switch (i.type) {
                    case DocumentType.PDF: { /* ... */ break; }
                    case DocumentType.DOCX: { /* ... */ break; }
                    case DocumentType.XML: { /* ... */ break; }
                }
            }
        }
    }
}
```

### 2) Ошибки/риски
- Неверный синтаксис `case DocumentType.PDF:` — достаточно `case PDF:`.
- Жёсткая связка: при добавлении типа — менять `switch`, нарушает OCP.
- Нет обработки `null`.

### 3) Что сделать лучше
- Интерфейс `DocumentProcessor` и реализации на типы.
- Регистрация через Spring (список бинов → карта `type→processor`).

### 4) Корректная реализация
```java
public interface DocumentProcessor {
  DocumentType supports();
  void process(Document doc);
}

@Component class PdfProcessor implements DocumentProcessor { /* ... */ }
@Component class DocxProcessor implements DocumentProcessor { /* ... */ }
@Component class XmlProcessor  implements DocumentProcessor { /* ... */ }

@Service
public class DocumentService {
  private final Map<DocumentType,DocumentProcessor> registry;
  public DocumentService(List<DocumentProcessor> processors){
    this.registry = processors.stream().collect(Collectors.toUnmodifiableMap(DocumentProcessor::supports, p->p));
  }
  public void process(Document[] docs){
    for (Document d: docs){
      Objects.requireNonNull(d);
      var p = Objects.requireNonNull(registry.get(d.type), "Unsupported type: "+d.type);
      p.process(d);
    }
  }
}
```

---

## 7) Транзакции + ФС: порядок операций (вариант 2)

### 1) Задание
Обновляет БД, затем переименовывает файл на диске — исправить.

### Исходный код
```java
@Transactional
public void process(String oldName, String newName) {
    Long id = exec("select id from file where name='" + oldName + "'");
    exec("update file set name='" + newName + "' where id = " + id);
    processFile(oldName, newName);
}
```

### 2) Ошибки/риски
- SQL-инъекции.
- Сначала обновляется БД, потом ФС: при падении на ФС — рассинхрон.
- Нет компенсации и блокировок.

### 3) Что сделать лучше
- Как в задаче 2: атомарный move, затем короткий апдейт; при ошибке — откат ФС.

### 4) Корректная реализация
```java
@Service @RequiredArgsConstructor
public class FileRenameService {
  private final JdbcTemplate jdbc;
  public void process(String oldName, String newName) {
    Long id = jdbc.queryForObject("select id from file where name = ?", Long.class, oldName);
    if (id == null) throw new IllegalArgumentException("Not found: " + oldName);
    Path oldP = Path.of("/files", oldName), newP = Path.of("/files", newName);
    try { Files.move(oldP, newP, StandardCopyOption.ATOMIC_MOVE); }
    catch (IOException e){ throw new IllegalStateException("FS rename failed", e); }
    try { jdbc.update("update file set name = ? where id = ?", newName, id); }
    catch (DataAccessException e){ try { Files.move(newP, oldP, StandardCopyOption.ATOMIC_MOVE); } catch (IOException ignored) {} throw e; }
  }
}
```

---

## 8) Контракт/ценообразование: состояние в синглтоне

### 1) Задание
Контроллер кладёт `Contract` в поле бина `ContractService` и вызывает `updatePrice()` — исправить состояние и сохранение.

### Исходный код
```java
@RestController
@RequestMapping(path = "/contract")
public class ContractController {
    private final ContractRepository repository;
    private final ContractService contractService;
    public ContractController(ContractRepository repository, ContractService contractService) {
        this.repository = repository;
        this.contractService = contractService;
    }
    @PostMapping(path = "/{contractId}/price")
    public void updatePrice(@PathVariable Long contractId) {
        Contract contract = repository.findById(contractId).orElseThrow();
        contractService.setContract(contract);
        contractService.updatePrice();
    }
}

@Service
public class ContractService {
    private final PriceService priceService;
    private Contract contract;
    public ContractService(PriceService priceService) { this.priceService = priceService; }
    public void setContract(Contract contract) { this.contract = contract; }
    public void updatePrice() {
        if (contract != null) {
            BigDecimal price = priceService.calcPrice(contract);
            contract.setPrice(price);
        }
    }
}
class Contract { private BigDecimal price; /* getters/setters */ }
```

### 2) Ошибки/риски
- **Хранение состояния** (`contract`) в синглтоне → гонки между запросами.
- Нет транзакции и **сохранения** в репозиторий.
- Контроллер возвращает `void`, нет статуса.
- Не обрабатываются 404 и ошибки расчёта.

### 3) Что сделать лучше
- Сервис **без состояния**: передавать `id`/`Contract` в метод.
- `@Transactional`, `repository.save`, возврат 204/DTO.

### 4) Корректная реализация
```java
@RestController @RequestMapping("/contract") @RequiredArgsConstructor
public class ContractController {
  private final ContractService service;
  @PostMapping("/{id}/price")
  public ResponseEntity<Void> updatePrice(@PathVariable Long id){
    service.updatePrice(id);
    return ResponseEntity.noContent().build();
  }
}
@Service @RequiredArgsConstructor
public class ContractService {
  private final ContractRepository repo; private final PriceService price;
  @Transactional
  public void updatePrice(Long id){
    Contract c = repo.findById(id).orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    c.setPrice(price.calcPrice(c));
    repo.save(c);
  }
}
```

---

## 9) Высоконагруженный микросервис чеков: @Scheduled, AOP, Optional, async

### 1) Задание
Исправить ошибки и сделать `sendReceipt()` неблокирующим.

### Исходный код (фрагменты)
```java
@Service
@AllArgsConstructor
public class SomeServiceImpl implements Someservice {
    private final TaxClient taxClient;
    private final ReceiptDao receiptDao;
    private final ReceiptTypeDao receiptTypeDao;

    private final ReceiptMapper mapper;
    private final ReceiptTypeMapper typeMapper;
    private final RefundReceiptMapper refundReceiptMapper;
    private final SomeService self;

    @Scheduled(cron = ""${cron.expression}"") // ""30 30 * * *""
    public void processJobRunning() {
        List<Receipt> receipts = getRefundReceipts();

        for (Receipt receipt : receipts) {
            receipt.setProcessed(true);
            eventListener.sendEventToAnalytics(new SomeEvent(receipt.getId()); 
            receiptDao.save(mapReceipt(mapper.map(receipt)));
        }
    }

    @Transactional(readOnly = true)
    public List<Receipt> getRefundReceipts() {
        return receiptDao.findAllBySourceAndProcessedFalse(ReceiptSource.REFUND);
    }

    private Receipt mapReceipt(ReceiptDto receiptDto) {
        return mapper.map(receiptDto);
    }

    public void sendReceipt(ReceiptDto receiptDto) {
        ReceiptSource source = self.getReceiptSource(receiptDto);
        taxClient.sendReceipt(receiptDto, source);
    }

    @Transactional
    @Cacheable(value = ""receipt_type"", cacheManager = ""redisCacheManager"")
    public ReceiptSource getReceiptSource(ReceiptDto receiptDto) {
        Receipt receipt = receiptDao.findById(receiptDto.getId());
        ReceiptSource source = receipt.getSource();
        return source;
    }
}
```

### 2) Ошибки/риски
- `@Scheduled(cron = ""${cron.expression}"")` — двойные кавычки: должно быть `@Scheduled(cron = "${cron.expression}")`.
- Пропущена `)` в `sendEventToAnalytics(new SomeEvent(...))`.
- NPE/Optional: `findById` отдаёт `Optional<Receipt>`, а используется как `Receipt`.
- `mapper.map(receipt)` → `mapReceipt(mapper.map(receipt))` — подозрительный двойной маппинг.
- Self-invocation: `@Cacheable` на методе, но вызов через `this` не активирует прокси; нужен **бин-прокси** (`self` — ок, если правильно сконфигурирован) или вынос в отдельный бин.
- Сохранение/отправка в цикле без `try/catch` — одна ошибка прервёт весь цикл.
- `sendReceipt()` синхронно ждёт внешний сервис с таймаутом до 60 сек → блокировка потоков, деградация под нагрузкой.
- Нет батч-процессинга/пагинации в `processJobRunning`.

### 3) Что сделать лучше
- Исправить cron и синтаксис.
- В `processJobRunning()`: пагинация, `try/catch` на каждый чек, помечать `processed=true` после успешной отправки события/сохранения.
- `findById(...).orElseThrow(...)`.
- `sendReceipt()` отправлять **в очередь** (Kafka/RabbitMQ) и отдельным consumer вызывать налоговую; либо `@Async` + пул.
- Ретраи/таймауты (Resilience4j) для внешнего вызова.
- Логи/метрики.

### 4) Корректная реализация (вариант с очередью)
```java
@Service @RequiredArgsConstructor
public class SomeServiceImpl implements SomeService {
  private final TaxClient taxClient;
  private final ReceiptDao receiptDao;
  private final ApplicationEventPublisher eventPublisher;
  private final KafkaTemplate<String, ReceiptDto> kafka;
  private final SomeService self; // AOP proxy

  @Scheduled(cron="${cron.expression}")
  public void processJobRunning(){
    int page=0, size=500;
    Page<Receipt> batch;
    do {
      batch = receiptDao.findAllBySourceAndProcessedFalse(ReceiptSource.REFUND, PageRequest.of(page++, size));
      for (Receipt r: batch){
        try {
          eventPublisher.publishEvent(new SomeEvent(r.getId()));
          r.setProcessed(true);
          receiptDao.save(r);
        } catch (Exception e){
          // лог/метрики, чек останется processed=false → ретрай
        }
      }
    } while (!batch.isEmpty());
  }

  public void sendReceipt(ReceiptDto dto){
    ReceiptSource source = self.getReceiptSource(dto);
    dto.setSource(source);
    kafka.send("tax-receipts", dto.getId().toString(), dto);
  }

  @Transactional(readOnly=true)
  @Cacheable(value="receipt_type", cacheManager="redisCacheManager", key="#dto.id")
  public ReceiptSource getReceiptSource(ReceiptDto dto){
    return receiptDao.findById(dto.getId())
      .map(Receipt::getSource)
      .orElseThrow(() -> new IllegalArgumentException("Receipt not found: "+dto.getId()));
  }
}

public interface ReceiptDao extends JpaRepository<Receipt, Long> {
  Page<Receipt> findAllBySourceAndProcessedFalse(ReceiptSource source, Pageable pageable);
}

@Component @RequiredArgsConstructor
class TaxConsumer {
  private final TaxClient taxClient;
  @KafkaListener(topics="tax-receipts", groupId="tax-sender")
  public void onReceipt(ReceiptDto dto){ taxClient.sendReceipt(dto, dto.getSource()); }
}
```

---

## 10) DocumentService `readDocument(String type)`: if-else → Стратегия (+PDF)

### 1) Задание
Код-ревью текущей реализации и рефакторинг на паттерн **Стратегия**; добавить чтение PDF.

### Исходный код
```java
@Component
public class DocumentService {
    public InputStream readDocument(String type) {
        DocumentReader reader = new DocumentReader();

        if (type.equals("PDF")) {
            return reader.readPdf();
        } else if (type.equals("DOCX")) {
            return reader.readDocx();
        } else if (type.equals("XLSX")) {
            return reader.readXlsx();
        } else {
            return null;
        }
    }
}
```

### 2) Ошибки/риски
- Жёсткие строковые сравнения, риск опечаток/регистра.
- Возврат `null` → NPE у вызывающего.
- Нарушение OCP: добавление нового типа требует правки `if-else`.
- Создание `new DocumentReader()` внутри метода — сложно тестировать/заменять.
- Нет валидации `type` и логирования.

### 3) Что сделать лучше
- Ввести `enum DocumentType` и `fromString()`.
- Интерфейс `DocumentReaderStrategy` с реализациями (`PdfReader`, `DocxReader`, `XlsxReader`).
- Регистрация стратегий через Spring (список → карта).
- Исключения вместо `null`.

### 4) Корректная реализация
```java
public enum DocumentType {
  PDF, DOCX, XLSX;
  public static DocumentType fromString(String t){
    return Arrays.stream(values()).filter(v -> v.name().equalsIgnoreCase(t))
      .findFirst().orElseThrow(() -> new IllegalArgumentException("Unsupported type: " + t));
  }
}
public interface DocumentReaderStrategy {
  DocumentType supports();
  InputStream read();
}
@Component class PdfReader implements DocumentReaderStrategy { /* read PDF */ }
@Component class DocxReader implements DocumentReaderStrategy { /* read DOCX */ }
@Component class XlsxReader implements DocumentReaderStrategy { /* read XLSX */ }

@Component
public class DocumentService {
  private final Map<DocumentType, DocumentReaderStrategy> map;
  public DocumentService(List<DocumentReaderStrategy> list){
    this.map = list.stream().collect(Collectors.toUnmodifiableMap(DocumentReaderStrategy::supports, s->s));
  }
  public InputStream readDocument(String type){
    DocumentReaderStrategy s = map.get(DocumentType.fromString(type));
    if (s==null) throw new IllegalArgumentException("Reader not found: " + type);
    return s.read();
  }
}
```

---

## 11) Поиск Person по имени → Optional

### 1) Задание
Реализовать `findPersonByName(List<Person>, String)`.

### Исходный код
```java
class Person { /* name, age, getters */ }
public class PersonService {
    public Optional<Person> findPersonByName(List<Person> persons, String name) {
        // TODO
        return Optional.empty();
    }
}
```

### 2) Ошибки/риски
- Метод всегда возвращает `Optional.empty()`.
- Нет обработки `null` списка/имени/элементов.

### 3) Что сделать лучше
- Использовать `stream().filter(...).findFirst()` и аккуратно обрабатывать `null`.
- Решить про сравнение (с регистром/без).

### 4) Корректная реализация
```java
public Optional<Person> findPersonByName(List<Person> persons, String name) {
  if (persons == null || name == null) return Optional.empty();
  return persons.stream().filter(p -> p != null && name.equals(p.getName())).findFirst();
}
```

---

## 12) Бронирование номера (тесты ок, в проде косяки)

### 1) Задание
Почему код проходит стенды, но падает в проде? Исправить.

### Исходный код
```java
@Service
class BookingService {
    @Autowired
    private RoomRepository roomRepository;

    public boolean bookRoom(Integer roomId) {
        boolean roomBooked = false;

        Room room = roomRepository.findById(roomId);
        if(room.getStatus() == ""VACANT"") {
            room.setStatus(""BOOKED"");
            room.setClientId(SecurityContext.getClientId());
            roomRepository.save(room);
            roomBooked = true;
        }
        return roomBooked;
    }
}
@Entity
class Room {
    @Id Integer id;
    String status; // VACANT, BOOKED
    String clientId;
    String roomNumber;
}
public interface RoomRepository extends JpaRepository<Room, Integer> {}
```

### 2) Ошибки/риски
- `findById` у Spring Data возвращает `Optional`, а не `Room`.
- Сравнение строк через `==`. Лучше `enum RoomStatus`.
- Нет транзакции/блокировок → **двойные продажи** при гонках (несколько подов).
- Нет `@Version` (оптимистическая блокировка) или `@Lock(PESSIMISTIC_WRITE)`.
- Полевая инъекция.

### 3) Что сделать лучше
- `enum RoomStatus { VACANT, BOOKED }`.
- Пессимистическая блокировка при бронировании или оптимистическая с ретраями.
- `@Transactional`.
- Обработка `Optional`, возврат понятного результата/исключения.

### 4) Корректная реализация (пессимистическая блокировка)
```java
@Entity
class Room {
  @Id Integer id;
  @Enumerated(EnumType.STRING) RoomStatus status;
  String clientId; String roomNumber;
  @Version Long version;
}
enum RoomStatus { VACANT, BOOKED }

public interface RoomRepository extends JpaRepository<Room,Integer> {
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query("select r from Room r where r.id = :id")
  Optional<Room> findForUpdate(@Param("id") Integer id);
}

@Service @RequiredArgsConstructor
class BookingService {
  private final RoomRepository repo;
  @Transactional
  public boolean bookRoom(Integer roomId){
    Room r = repo.findForUpdate(roomId).orElseThrow();
    if (r.getStatus() == RoomStatus.VACANT){
      r.setStatus(RoomStatus.BOOKED);
      r.setClientId(SecurityContext.getClientId());
      repo.save(r);
      return true;
    }
    return false;
  }
}
```

---

## 13) Большой `OrderService`: вопросы по валидации, SRP, транзакциям

### 1) Задание
Ответить на вопросы (валидация, SRP, rollback, риски) и отрефакторить.

### Исходный код
```java
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;
    @Autowired private InventoryRepository inventoryRepository;
    @Autowired private NotificationService notificationService;

    @Transactional
    public void processOrder(Order order) {
        validateOrder(order);
        calculateTotalPrice(order.getItems(), order.getCustomerType());
        saveOrder(order);
        notifyCustomer(order);
        updateInventory(order);
    }

    public double calculateTotalPrice(List<OrderItem> items, String customerType) {
        double total = items.stream().mapToDouble(item -> item.getPrice() * item.getQuantity()).sum();
        if (""""VIP"""".equals(customerType)) total *= 0.9;
        else if (""""LOYALTY"""".equals(customerType)) total *= 0.85;

        if (orderRepository.countByCustomerType(customerType) > 10) {
            total *= 0.95;
        }
        return total;
    }

    private void saveOrder(Order order) { orderRepository.save(order); }

    private void updateInventory(Order order) {
        order.getItems().forEach(item -> {
            inventoryRepository.decreaseStock(item.getProductId(), item.getQuantity());
        });
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    int countByCustomerType(String customerType);
    Order findByOrderId(Long id);
}
```

### 2) Ошибки/риски и ответы
- **Валидация DTO**: на границе (контроллер), `@Valid` + аннотации в DTO (`@NotEmpty items`, `@Positive qty/price`, `@NotNull customerType`). Бизнес-валидация — в домене/сервисе.
- **Null-проверки**: на входе в сервис предполагаем валидные данные; в контроллере — проверяем и маппим.
- **SRP**: сервис делает и рассчёт, и уведомления, и инвентарь. Следует разделить на `PricingService`, `InventoryService`, `NotificationService`; `OrderService` — оркестратор.
- **Rollback**: по `RuntimeException` (или по любым, если `rollbackFor=Exception.class`). Если `saveOrder` кинет исключение — транзакция откатится, уведомления/инвентарь лучше делать после сохранения.
- **Деньги**: `double` — не подходит (погрешности). Нужен `BigDecimal` и округление.
- **Доп. скидка по `countByCustomerType`**: тяжёлый запрос в расчёте; кэшировать/агрегировать заранее; и лучше `enum CustomerType` вместо строк.
- **`findByOrderId`**: сгенерируется, если есть поле `orderId`; но обычно у сущности `id`. Если поля нет — метод не сработает как ожидается.
- **Инвентарь**: `decreaseStock` должен быть атомарным (SQL с условием `stock >= ?`), иначе гонки/отрицательные остатки.
- **Уведомления**: лучше **async/outbox** (после коммита).

### 3) Что сделать лучше
- Разделение ответственности, `BigDecimal`, DTO-валидация, enum-ы вместо строк, outbox для событий, атомарное изменение инвентаря, метрики.

### 4) Корректная реализация (сокращённо)
```java
enum CustomerType { REGULAR, VIP, LOYALTY }

@Service @RequiredArgsConstructor
public class OrderService {
  private final OrderRepository orders;
  private final InventoryService inventory;
  private final PricingService pricing;
  private final NotificationService notifications;

  @Transactional
  public void processOrder(@Valid Order order){
    if (order.getItems()==null || order.getItems().isEmpty()) throw new IllegalArgumentException("Items required");
    BigDecimal total = pricing.calculateTotal(order.getItems(), order.getCustomerType());
    order.setTotal(total);
    orders.save(order);
    inventory.decreaseFor(order); // атомарно и с проверками
    notifications.notifyAsync(order.getCustomerId(), "Order accepted: "+order.getId());
  }
}

@Service
public class PricingService {
  public BigDecimal calculateTotal(List<OrderItem> items, CustomerType type){
    BigDecimal total = items.stream()
      .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
      .reduce(BigDecimal.ZERO, BigDecimal::add);
    switch (type){
      case VIP -> total = total.multiply(new BigDecimal("0.90"));
      case LOYALTY -> total = total.multiply(new BigDecimal("0.85"));
      default -> {}
    }
    return total;
  }
}

@Repository
public interface InventoryRepository extends JpaRepository<Inventory,Long>{
  @Modifying
  @Query("update Inventory i set i.stock = i.stock - :q where i.productId=:pid and i.stock >= :q")
  int decreaseStock(@Param("pid") Long productId, @Param("q") int qty);
}

@Service @RequiredArgsConstructor
public class InventoryService {
  private final InventoryRepository repo;
  public void decreaseFor(Order order){
    for (OrderItem it: order.getItems()){
      int updated = repo.decreaseStock(it.getProductId(), it.getQuantity());
      if (updated==0) throw new IllegalStateException("Insufficient stock for "+it.getProductId());
    }
  }
}
```

---

## 14) OrderService: аудит, исключения, транзакции

### 1) Задание
Ревью и исправление — корректная транзакция, порядок действий, логирование.

### Исходный код
```java
public class OrderService {
@Autowired private OrderRepository orderRepository;
@Inject private AuditRepository auditRepository;

public void save(Order order) throws OrderValidationException {
    try { saveOrderInternal(order); }
    catch (Throwable e) {
        e.printStackTrace();
        throw new OrderSaveException(e, order);
    }
}

@Transactional
private void saveOrderInternal(Order order) throws OrderValidationException {
    auditRepository.save(order);
    if (order.getClient().getName() == """") {
        throw new OrderValidationException(""Поле клиент не заполнено."");
    }
    orderRepository.save(order);
}
}
```

### 2) Ошибки/риски
- `@Transactional` на `private` методе не сработает из-за self-invocation/AOP.
- Аудит до валидации — может записывать мусор.
- Ловим `Throwable` и `printStackTrace()` — плохая практика; используем логгер и корректные исключения.
- Checked `OrderValidationException` по умолчанию **не откатывает** транзакцию.
- Смешение `@Autowired` и `@Inject` без причины.

### 3) Что сделать лучше
- Поставить `@Transactional(rollbackFor = Exception.class)` на **public** метод.
- Валидация **до** сохранения, аудит после успешной записи (или `@TransactionalEventListener(AFTER_COMMIT)`).
- Нормальные логи.

### 4) Корректная реализация
```java
@Service @RequiredArgsConstructor @Slf4j
public class OrderService {
  private final OrderRepository orderRepository;
  private final AuditRepository auditRepository;

  @Transactional(rollbackFor = Exception.class)
  public void save(Order order){
    validate(order);
    orderRepository.save(order);
    auditRepository.save(new AuditRecord("ORDER_SAVED", order.getId(), Instant.now()));
  }

  private void validate(Order o){
    if (o==null || o.getClient()==null || o.getClient().getName()==null || o.getClient().getName().isBlank())
      throw new OrderValidationRuntimeException("Поле клиент не заполнено");
  }
}
```

---

## 15) Обновление статуса заказа и уведомления

### 1) Задание
Привести к чистому коду: enum статусов, транзакция, единый путь сохранения/уведомлений.

### Исходный код
```java
public class OrderService {
@Autowired private OrderRepository orderRepository;
private final NotificationService notificationService;
@Autowired
public OrderService(NotificationService notificationService) { this.notificationService = notificationService; }

public void updateOrderStatus(Long orderId, String newStatus) {
    Order order = orderRepository.findById(orderId);
    if (newStatus.equals(""COMPLETED"")) {
        order.setStatus(""COMPLETED"");
        notificationService.notify(order.getUserId(), ""Your order is completed"");
    } else if (newStatus.equals(""CANCELLED"")) {
        order.setStatus(""CANCELLED"");
        notificationService.notify(order.getUserId(), ""Your order is cancelled"");
    } else if (newStatus.equals(""PENDING"")) {
        order.setStatus(""PENDING"");
        orderRepository.save(order);
    } else if (newStatus.equals(""IN_PROGRESS"")) {
        order.setStatus(""IN_PROGRESS"");
        orderRepository.save(order);
    } else {
        throw new IllegalArgumentException(""Unsupported status: "" + newStatus);
    }
}
}
```

### 2) Ошибки/риски
- `findById` возвращает `Optional`, а не `Order`.
- Дублирование кода и непоследовательность сохранения (иногда сохраняем, иногда нет).
- Магические строки, нет enum.
- Нет транзакции, уведомление до сохранения.

### 3) Что сделать лучше
- Enum `OrderStatus`, один `save` после установки статуса, уведомления после сохранения.
- Обработка отсутствия заказа.

### 4) Корректная реализация
```java
enum OrderStatus { COMPLETED, CANCELLED, PENDING, IN_PROGRESS }

@Service @RequiredArgsConstructor
public class OrderService {
  private final OrderRepository repo;
  private final NotificationService notifications;

  @Transactional
  public void updateOrderStatus(Long id, OrderStatus newStatus){
    Order o = repo.findById(id).orElseThrow(() -> new IllegalArgumentException("Not found: "+id));
    o.setStatus(newStatus.name());
    repo.save(o);
    switch (newStatus){
      case COMPLETED -> notifications.notify(o.getUserId(), "Your order is completed");
      case CANCELLED -> notifications.notify(o.getUserId(), "Your order is cancelled");
      default -> {}
    }
  }
}
```

---

## 16) Проверка возможности продажи по времени/категории

### 1) Задание
Реализовать `checkSale(Product item, Integer hour)` согласно правилам.

### Исходный код (интерфейсы/описание)
```java
public class IceCream implements Product {};
public abstract class Alcohol implements Product { int getAbv(); };
/*
 * 13:00–14:00 — обед, продажа запрещена
 * 23:00–02:59 — запрет на продажу любого алкоголя
 * 03:00–07:59 — запрет на продажу крепкого алкоголя (>=40)
*/
public boolean checkSale(Product item, Integer hour) { /* ... */ }
```

### 2) Ошибки/риски
- Нужно аккуратно обработать интервалы, особенно переход суток (23:00–02:59).
- Проверить валидность `hour` (0..23).

### 3) Что сделать лучше
- Явные условия; алкоголю запрет ночью; крепкому — запрет ранним утром; обед — всем.

### 4) Корректная реализация
```java
public boolean checkSale(Product item, Integer hour) {
  if (item==null || hour==null) return false;
  if (hour<0 || hour>23) throw new IllegalArgumentException("hour 0..23");
  if (hour == 13) return false; // обед
  boolean isAlcohol = item instanceof Alcohol;
  int abv = isAlcohol ? ((Alcohol)item).getAbv() : 0;
  if (isAlcohol && (hour == 23 || hour <= 2)) return false; // 23:00–02:59
  if (isAlcohol && (hour >= 3 && hour <= 7) && abv >= 40) return false; // 03:00–07:59 крепкий
  return true;
}
```

---

## 17) Поиск товаров при высоком RPS и частом `count=0`

### 1) Задание
Оптимизировать метод, который делает `count` → `find`.

### Исходный код
```java
class Service {
    public List<Product> find(Filter filter) {
        int count = repo.count(filter);
        if(count > 0) {
            return repo.find(filter);
        }
        return new ArrayList<>();
    }
}
```

### 2) Ошибки/риски
- Два запроса на каждый вызов → нагрузка; при 50% `count=0` — половину запросов тратим зря.
- Нет индексов; нет лимитов.

### 3) Что сделать лучше
- Заменить на `exists` (дешевле, чем `count`) или сразу `find` с лимитом/страницей.
- Кэшировать «пустые» результаты на короткий TTL, если запросы повторяются.

### 4) Корректная реализация
```java
public List<Product> find(Filter filter) {
  if (!repo.existsByFilter(filter)) return Collections.emptyList();
  return repo.findByFilter(filter); // или первая страница
}
// или одним запросом:
public List<Product> find(Filter filter){
  return repo.findByFilter(filter, PageRequest.of(0,50)).getContent();
}
```

---

## 18) Синхронизация: разные мониторы и неверный `main`

### 1) Задание
Сделать защиту критической секции и исправить `main`.

### Исходный код
```java
class Main{
void main(){
    execute(()->new Service.doSome());
    execute(()->new Service.doSome());
}
}
class Service{
    void doSome(){
        Object obj = new Object();
        synchronized(obj){ /* code */ }
    }
}
```

### 2) Ошибки/риски
- Каждый раз создаётся новый объект-монитор → потоки не синхронизируются.
- `new Service.doSome()` — синтаксическая ошибка; `main` должен быть `static`.

### 3) Что сделать лучше
- Использовать общий `final Object lock` или `synchronized (this)`.
- Пример с пулом потоков.

### 4) Корректная реализация
```java
class Main {
  public static void main(String[] args){
    Service s = new Service();
    ExecutorService ex = Executors.newFixedThreadPool(2);
    ex.submit(s::doSome); ex.submit(s::doSome);
    ex.shutdown();
  }
}
class Service {
  private final Object lock = new Object();
  void doSome(){
    synchronized (lock) { /* критическая секция */ }
  }
}
```

---

## 19) In-memory `UserService` под нагрузкой

### 1) Задание
Сделать быстрый и безопасный в памяти «кеш» пользователей.

### Исходный код
```java
@Service
public class UserService {
    private Map<Integer, User> usersCache = new HashMap<>();
    public void addUser(User user) { usersCache.put(user.getId(), user); }
    public User getUser(int id) { return usersCache.get(id); }
    public List<User> getAllUsers() { return new ArrayList<>(usersCache.values()); }
}
```

### 2) Ошибки/риски
- `HashMap` не потокобезопасен.
- Возврат мутабельных объектов может привести к внешним изменениям состояния.
- Нет ограничений по памяти/TTL/метрик.

### 3) Что сделать лучше
- `ConcurrentHashMap`, `Optional`, иммутабельные объекты (record) или копии.
- Для прод-нагрузки — Caffeine/Redis с TTL/лимитами.

### 4) Корректная реализация
```java
@Service
public class UserService {
  private final ConcurrentMap<Integer, User> users = new ConcurrentHashMap<>();
  public void addUser(User u){ Objects.requireNonNull(u); users.put(u.getId(), u); }
  public Optional<User> getUser(int id){ return Optional.ofNullable(users.get(id)); }
  public List<User> getAllUsers(){ return List.copyOf(users.values()); }
  public void removeUser(int id){ users.remove(id); }
}
```

---

## 20) `DocumentService`: принят/завершён (строки, N+1, кэш)

### 1) Задание
Два метода: `isAcceptedDocument` и `isFinishedDocument`. Исправить сравнение строк, N+1 и повысить расширяемость.

### Исходный код
```java
@Service
public class DocumentService {
    @Autowired
    StatusRepository statusRepository;
    /*
     * Проверяет принят ли документ...
     */
    public boolean isAcceptedDocument(Document document) {
        List<String> statusCodes = Arrays.asList("500", "293", "216", "processed");
        boolean result = false;
        for (String status : statusCodes) {
            if (document.getCurrentStatus().getCode() == status) {
                result = true;
                break;
            }
        }
        return result;
    }
    /*
     * Проверяет завершена ли обработка...
     */
    public boolean isFinishedDocument(Document document) {
        List<String> documentStatusCodes = document.getCodes();
        for (String docStatusCode : documentStatusCodes) {
            Status status = statusRepository.findByCode(docStatusCode);
            if (status.isFinal()) {
                return true;
            }
        }
        return false;
    }
}
```

### 2) Ошибки/риски (расширенно)
- Сравнение строк через `==` → сравнение ссылок, не значений.
- «Магические коды» и смешение форматов (числа/строки), лучше enum/справочник/конфиг.
- Возможные NPE (`document`, `currentStatus`, `code`, `getCodes()`).
- N+1 к БД в `isFinishedDocument`.
- Полевая инъекция.

### 3) Что сделать лучше
- Конструкторная инъекция.
- Набор «принят» в конфиге/БД; `Set.contains`.
- Пакетная загрузка статусов или кэш финальных кодов; обработка `null`.

### 4) Корректная реализация
```java
@Service @RequiredArgsConstructor
public class DocumentService {
  private final StatusRepository repo;
  private static final Set<String> ACCEPTED = Set.of("500","293","216","processed");

  public boolean isAcceptedDocument(Document d){
    if (d==null || d.getCurrentStatus()==null) return false;
    String code = d.getCurrentStatus().getCode();
    return code!=null && ACCEPTED.contains(code);
  }

  @Transactional(readOnly=true)
  public boolean isFinishedDocument(Document d){
    if (d==null || d.getCodes()==null || d.getCodes().isEmpty()) return false;
    List<Status> statuses = repo.findAllByCodeIn(d.getCodes());
    return statuses.stream().anyMatch(Status::isFinal);
  }
}
```

---

## 21) `CachedPhotosService`: кэш ресайза

### 1) Задание
Ревью и исправление кэшируемого ресайза.

### Исходный код
```java
@Component
public final class CachedPhotosService {
    private static final String RESIZED_PHOTO_CACHE_NAME = "RESIZED_PHOTO_CACHE_NAME";
    @Autowired private PhotoRepository photoRepository;
    @Autowired private PhotoValidationService photoValidationService;
    @Autowired private PhotoOperations photoOperations;

    @Cacheable(cacheNames = RESIZED_PHOTO_CACHE_NAME)
    public PhotoDTO resizedPhoto(String photoId, int width, int height) {
        photoValidationService.validateSize(width, height);
        Photo photo = photoRepository.findById(photoId);
        PhotoDto photoDto = ConvertionUtils.convert(photo);
        var resizedPhoto = photoOperations.resize(photoDto, width, height);
        return resizedPhoto;
    }
}
```

### 2) Ошибки/риски (расширенно)
- `findById` вероятно `Optional`, а не `Photo`.
- Несогласованность `PhotoDTO`/`PhotoDto` и `ConvertionUtils` (опечатка).
- Полевая инъекция.
- Не ясна политика кэша (TTL/размер).

### 3) Что сделать лучше
- Конструкторная инъекция; явный ключ кэша; обработка «не найдено»; `@Transactional(readOnly = true)` если нужны ленивые поля; настроить кэш-менеджер.

### 4) Корректная реализация
```java
@Service @RequiredArgsConstructor
public final class CachedPhotosService {
  public static final String RESIZED_PHOTO_CACHE_NAME = "RESIZED_PHOTO_CACHE_NAME";
  private final PhotoRepository repo;
  private final PhotoValidationService validator;
  private final PhotoOperations ops;

  @Cacheable(cacheNames=RESIZED_PHOTO_CACHE_NAME, key="#photoId+':'+#width+'x'+#height")
  @Transactional(readOnly=true)
  public PhotoDto resizedPhoto(String photoId, int width, int height){
    validator.validateSize(width, height);
    Photo photo = repo.findById(photoId).orElseThrow(() -> new PhotoNotFoundException(photoId));
    PhotoDto dto = ConversionUtils.convert(photo);
    return ops.resize(dto, width, height);
  }
}
```

---

## 22) `PizzaService`: транзакции, сущность, логирование

### 1) Задание
Починить сервис и объяснить, почему на стендах «работает», а в проде — нет.

### Исходный код
```java
@Component
@Slf4j
public class PizzaSevice {
    @Autowired private UserRepository userRepository;
    @Autowired private PizzaRequestRepository pizzaRequestRepository;
    @Autowired private KitchenService kitchenService;

    public UUID requestPizza(UUID userId, PizzaType pizzaType, double price) {
        log.info("request pizza: user id = " + userId);
        User user = userRepository.getOne(userId);
        if ( user == null ) { return null; }
        PizzaRequest pizzaRequest = new PizzaRequest(UUID.randomUUID());
        fillValues(pizzaRequest, price, pizzaType);
        saveAndVerify(pizzaRequest);
        log.info("request created: pizza id = " + pizzaRequest.getPizzaId());
        return pizzaRequest.getPizzaId();
    }
    public Boolean isPizzaReady(UUID pizzaId) {
        log.info("check pizza: id = " + pizzaId);
        Optional<PizzaRequest> pizzaRequestOptional = pizzaRequestRepository.findById(pizzaId);
        if (pizzaRequestOptional.isPresent()) {
            PizzaRequest pizzaRequest = pizzaRequestOptional.get();
            try {
                PizzaStatus pizzaStatus = kitchenService.getStatus(pizzaRequest);
                log.info("status = " + pizzaStatus);
                return pizzaStatus == PizzaStatus.READY;
            } catch (Exception e) { return false; }
        } else { throw new RuntimeException("PizzaRequest not found"); }
    }
    private void fillValues(PizzaRequest pizzaRequest, double price, PizzaType pizzaType) {
        pizzaRequest.setPrice(price);
        pizzaRequest.setUserId(userId);
        pizzaRequest.setPizzaType(pizzaType);
    }
    @Transactional
    private void saveAndVerify(PizzaRequest pizzaRequest) {
        pizzaRequestRepository.save(pizzaRequest);
        if (pizzaRequest.getPizzaType() == PizzaType.PEPPERONI) {
            throw new RuntimeException("Pepperoni is not available");
        }
    }
    @Entity @Data
    public class PizzaRequest {
        private final UUID pizzaId;
        private UUID userId;
        private PizzaType pizzaType;
        private double price;
        @ManyToOne(fetch = FetchType.LAZY) private PizzaMaker pizzaMaker;
    }
}
```

### 2) Ошибки/риски (расширенно)
- Опечатка `PizzaSevice`; логи через конкатенацию строк.
- `getOne` устарел и ведёт себя как ленивый прокси; проверка `user==null` бессмысленна; на проде → `EntityNotFoundException` при доступе.
- Возврат `null` как результат — лучше исключение/Optional.
- `fillValues` использует `userId`, которого в параметрах нет.
- `@Transactional` на `private` методе — AOP не сработает (self-invocation) → сохранит, затем исключение, и **не откатит** в проде.
- Вложенная `@Entity` как inner non-static класс и `final UUID` без `@Id` — JPA так нельзя.
- Деньги: `double` вместо `BigDecimal`.
- Грубое подавление исключений в `isPizzaReady` (теряем причину).

### 3) Что сделать лучше
- Публичный транзакционный метод `requestPizza`, проверять доступность перед `save`, конструкторная инъекция, `BigDecimal`, вынести `PizzaRequest` в top-level и корректно аннотировать, логгирование через плейсхолдеры, корректная обработка ошибок.

### 4) Корректная реализация (сокращённо)
```java
@Entity @Table(name="pizza_request")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor
public class PizzaRequest {
  @Id private UUID id;
  @Column(nullable=false) private UUID userId;
  @Enumerated(EnumType.STRING) @Column(nullable=false) private PizzaType pizzaType;
  @Column(nullable=false, precision=19, scale=2) private BigDecimal price;
  @ManyToOne(fetch=FetchType.LAZY) private PizzaMaker pizzaMaker;
}

@Component @RequiredArgsConstructor @Slf4j
public class PizzaService {
  private final UserRepository users;
  private final PizzaRequestRepository requests;
  private final KitchenService kitchen;

  @Transactional
  public UUID requestPizza(UUID userId, PizzaType type, BigDecimal price){
    log.info("request pizza: userId={}", userId);
    users.findById(userId).orElseThrow(() -> new IllegalArgumentException("User not found"));
    if (type == PizzaType.PEPPERONI) throw new IllegalStateException("Pepperoni is not available");
    PizzaRequest r = new PizzaRequest(UUID.randomUUID(), userId, type, price, null);
    requests.save(r);
    log.info("request created: pizzaId={}", r.getId());
    return r.getId();
  }

  @Transactional(readOnly=true)
  public boolean isPizzaReady(UUID pizzaId){
    PizzaRequest r = requests.findById(pizzaId).orElseThrow(() -> new IllegalArgumentException("Not found"));
    try {
      return kitchen.getStatus(r) == PizzaStatus.READY;
    } catch (KitchenServiceException e){
      log.warn("Kitchen failed for {}: {}", pizzaId, e.getMessage(), e);
      return false;
    }
  }
}
```

---

