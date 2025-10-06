# Maven — кратко и по делу (для собеса Java-developer)

### Что такое Maven  
Maven — это инструмент сборки (build tool) и система управления зависимостями для проектов на Java (и JVM). По сути он описывает **как** проект собирается, какие у него зависимости, какие плагины запускаются и в каком порядке — всё через один файл `pom.xml`.

---

# Зачем нужен Maven
- **Управление зависимостями** — автоматически скачивает нужные библиотеки из репозиториев (Maven Central и т.д.).  
- **Стандартизованный жизненный цикл сборки** — единая последовательность фаз (compile → test → package → install → deploy).  
- **Плагины/расширяемость** — компиляция, тесты, сборка артефактов, генерация документации и т.д. через плагины.  
- **Повторяемость и переносимость** — одна и та же команда на любой машине/CI даёт предсказуемый результат.  
- **Мульти-модульные проекты (reactor)** — удобно собирать несколько модулей в одном запуске.  
- **Конфигурация в POM** — централизованное управление версиями, профилями сборки, свойствами.

---

# Главное в `pom.xml` (коротко)
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0.0</version>

  <dependencies>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>5.3.0</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- пример: компилятор -->
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.1</version>
        <configuration>
          <source>17</source>
          <target>17</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

---

# Жизненные циклы и ключевые фазы (с примерами)

Maven имеет три основных жизненных циклов: **clean**, **default (build)**, **site**.  
Каждый жизненный цикл содержит набор фаз; выполнение фазы запускает все предыдущие фазы в этом цикле.

## Жизненный цикл `clean`
Фазы: `pre-clean` → `clean` → `post-clean`  
Команда:
```bash
mvn clean
```
Пример: удаляет `target/` перед новой сборкой.

## Жизненный цикл `default` (основной — сборка)

**Общее поведение и порядок выполнения:**  
Жизненный цикл `default` — это основной цикл, который отвечает за сборку проекта: от валидации конфигурации до деплоя артефакта. Когда вы запускаете `mvn <фаза>`, Maven выполнит последовательно все фазы от начала жизненного цикла до указанной фазы включительно. Это важно: `mvn test` выполнит `validate`, `compile`, `test-compile` и `test` и всё между ними.

Ниже — подробное описание каждой ключевой фазы с пояснениями, типичными плагинами/целями (goals), и примерами использования.

### validate
- **Назначение:** проверка корректности проекта и его конфигурации (наличие `pom.xml`, корректность зависимостей, обязательных свойств и т.д.).  
- **Типичные действия:** валидация структуры POM, проверка обязательных свойств/профилей.  
- **Плагины/цели:** обычно кастомные проверки, например `maven-enforcer-plugin` для правил версий.  
- **Пример:** `mvn validate` — полезно в CI для быстрой проверки минимальной корректности.

### initialize
- **Назначение:** инициализация переменных и свойств, установка значений из профилей, подготовка окружения.  
- **Типичные действия:** установка property, подготовка директорий, чтение внешних конфигов.  
- **Плагины/цели:** часто используются кастомные плагины, или плагин `build-helper-maven-plugin` для добавления source directories.  
- **Пример:** генерация версий/метаданных перед сборкой.

### generate-sources
- **Назначение:** генерировать исходный код, который затем будет собран.  
- **Типичные действия:** генерация кода из WSDL, ANTLR, protobuf, jaxb, Lombok-делегированная генерация и т.д.  
- **Плагины/цели:** `jaxb2-maven-plugin`, `protobuf-maven-plugin`, `antlr4-maven-plugin`.  
- **Пример:** генерировать классы из `.proto` перед компиляцией: они попадут в `target/generated-sources`.

### process-sources
- **Назначение:** обработка исходников (например применение инструментов, препроцессинг).  
- **Типичные действия:** копирование/фильтрация исходников, шаблонизация.  
- **Плагины/цели:** `maven-resources-plugin` может применяться для фильтрации ресурсов.  
- **Пример:** подстановка значений property в исходники/ресурсы.

### generate-resources
- **Назначение:** генерировать ресурсы (не исходники) проекта.  
- **Типичные действия:** создание конфигов, swagger-генерация, статических файлов.  
- **Плагины/цели:** плагины генерации документации/ресурсов.  

### process-resources
- **Назначение:** копирование и обработка ресурсов проекта в каталог сборки (`src/main/resources` → `target/classes`).  
- **Типичные действия:** копирование, фильтрация (замена ${...}), изменение кодировок.  
- **Плагин/цель:** `maven-resources-plugin:resources` выполняется в этой фазе.  
- **Пример команды:** при `mvn package` ресурсы будут скопированы и подставлены в `target/classes`.

### compile
- **Назначение:** компиляция исходников Java в байт-код.  
- **Типичные действия:** запуск компилятора, указание target/source, опции.  
- **Плагин/цель:** `maven-compiler-plugin:compile`.  
- **Пример команды:** `mvn compile` — создаст `target/classes`.  
- **Упражнение:** если генерируемые классы находятся в `target/generated-sources`, убедитесь, что они добавлены в источник через `build-helper-maven-plugin` прежде чем `compile` запустится.

### process-test-resources
- **Назначение:** копирование и обработка ресурсов тестов (`src/test/resources` → `target/test-classes`).  
- **Плагин/цель:** `maven-resources-plugin:testResources`.

### test-compile
- **Назначение:** компиляция тестовых классов.  
- **Плагин/цель:** `maven-compiler-plugin:testCompile`.  
- **Пример:** после `test-compile` в `target/test-classes` доступны скомпилированные тесты.

### test
- **Назначение:** запуск юнит-тестов.  
- **Типичные действия:** исполнение тестов, формирование отчётов.  
- **Плагин/цель:** `maven-surefire-plugin:test` (юнит-тесты).  
- **Примеры команд:**  
  - `mvn test` — запустить тесты.  
  - `mvn -DskipTests test` — скомпилировать и пропустить исполнение тестов.  
  - `mvn -Dtest=MyTest#testMethod test` — запустить конкретный тест.

### prepare-package
- **Назначение:** подготовка к упаковке; можно использовать для предварительных шагов.  
- **Примеры:** упаковка дополнительных ресурсов внутрь артефакта, генерация manifest.

### package
- **Назначение:** упаковка скомпилированного кода в артефакт (jar/war/ear).  
- **Плагин/цель:** `maven-jar-plugin:jar`, `maven-war-plugin:war`.  
- **Команда:** `mvn package` — получим `target/<artifactId>-<version>.jar` или `.war`.  
- **Советы:** конфигурируйте `maven-jar-plugin`/`maven-shade-plugin`, если нужен fat-jar/uber-jar.

### pre-integration-test
- **Назначение:** подготовка окружения для интеграционных тестов (поднять контейнеры, deploy артефактов и т.д.).  
- **Типичные действия:** запуск docker-compose, разворачивание приложения во временное тестовое окружение.  
- **Плагины/цели:** `docker-maven-plugin`, `fabric8-maven-plugin`, `exec-maven-plugin`.

### integration-test
- **Назначение:** выполнение интеграционных тестов, которые требуют подготовленного окружения.  
- **Плагины/цели:** `maven-failsafe-plugin:integration-test`.  
- **Примечание:** интеграционные тесты чаще выполняют в `failsafe` плагине, потому что он разделяет фазы `integration-test` и `verify`.

### post-integration-test
- **Назначение:** очистка тестового окружения (выключить контейнеры, удалить временные данные).  
- **Примеры:** остановить docker-compose, удалить временные БД.

### verify
- **Назначение:** выполнить дополнительные проверки на собранном интеграционном артефакте (например проверки качества, отчеты).  
- **Плагины/цели:** `maven-failsafe-plugin:verify`, `jacoco-maven-plugin` для coverage checks.  
- **Пример:** проверить, что покрытие тестами выше порога.

### install
- **Назначение:** установка артефакта в локальный репозиторий (`~/.m2/repository`) для использования другими проектами на локальной машине.  
- **Плагин/цель:** `maven-install-plugin:install`.  
- **Команда:** `mvn install` — удобно для локальной разработки и мульти-модульных сборок.

### deploy
- **Назначение:** копирование финального артефакта в удалённый репозиторий (Nexus, Artifactory) для использования другими разработчиками/CI.  
- **Плагин/цель:** `maven-deploy-plugin:deploy`.  
- **Команда:** `mvn deploy` — обычно запускается в CI после успешного `install`/`verify`.

---

**Примеры команд и что они делают (повтор и расширение):**
- `mvn clean package` — удалит `target/`, выполнит весь цикл до `package` включительно, получим `target/my-app-1.0.0.jar`.  
- `mvn -DskipTests package` — собрать пакет, пропустить запуск тестов (тесты не будут выполняться, но будут скомпилированы если нужно).  
- `mvn clean install` — удалить старое, собрать, прогнать тесты и положить артефакт в локальный репозиторий (удобно для мульти-модульных зависимостей).  
- `mvn verify -P integration` — запустить до `verify` с профилем `integration` (например для интеграционных проверок и отчётов).

---

## Жизненный цикл `site`
Фазы: `pre-site` → `site` → `post-site` → `site-deploy`  
Используется для генерации сайта/документации проекта (`mvn site` / `mvn site:deploy`).

---

# Как фазы связаны с плагинами (коротко)
Фаза — это точка в жизненном цикле. Плагин выполняет задачу (goal) на этой фазе. Например:
- `maven-compiler-plugin:compile` выполняется в фазе `compile`.  
- `maven-surefire-plugin:test` — в фазе `test`.  
- `maven-jar-plugin:jar` — в `package`.  
- `maven-install-plugin:install` — в `install`.

---

# Частые вопросы/кейсы при собесе

**Q: Как запустить только компиляцию и пропустить тесты?**  
A: `mvn -DskipTests compile` или `mvn -DskipTests package`.

**Q: Что делает `mvn install` по сравнению с `mvn package`?**  
A: `package` формирует артефакт в `target/`. `install` делает то же и дополнительно помещает артефакт в локальный `.m2` репозиторий.

**Q: Почему фиксировать версии плагинов/зависимостей важно?**  
A: Для воспроизводимости сборки и чтобы CI не ломался при автоматическом апгрейде артефактов.

**Q: Что такое scope зависимости?**  
A: `compile`, `provided`, `runtime`, `test`, `system` — определяют, где и когда зависимость видна/включается.

---

# Короткая шпаргалка (команды)
- `mvn clean` — удалить артефакты сборки.  
- `mvn compile` — скомпилировать код.  
- `mvn test` — запустить юнит-тесты.  
- `mvn package` — упаковать jar/war.  
- `mvn install` — установить в локальный репозиторий.  
- `mvn deploy` — отправить в удалённый репозиторий.  
- `mvn site` — сгенерировать сайт проекта.

---

# Разница между `dependencyManagement` и `dependencies` в Maven

**Кратко:**  
- `dependencies` — конкретные зависимости проекта. Они **попадают в classpath** и включаются в артефакт/рантайм (в зависимости от `scope`).  
- `dependencyManagement` — **централизованная декларация версий и настроек** для зависимостей (и транзитивных зависимостей), но сама по себе **не добавляет** зависимости в classpath. Она даёт значения по умолчанию (version, scope, exclusions и т.д.), которые применяются только если зависимость явно объявлена в `dependencies` (в том же POM или в дочерних модулях).

---

## Поведение и назначение

### `dependencies`
- Описывает, какие артефакты нужны текущему модулю, например:
```xml
<dependencies>
  <dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.12.0</version>
    <scope>compile</scope>
  </dependency>
</dependencies>
```
- Эти зависимости:
  - скачиваются из репозитория (если их нет локально),
  - добавляются в classpath при компиляции/тестировании/рантайме (в зависимости от `scope`),
  - включаются в дерево зависимостей (и могут быть транзитивными).
- Требуется указывать `version` (если он не унаследован/не задан через `dependencyManagement`).

### `dependencyManagement`
- Служит для **централизованного управления версиями и настройками** зависимостей.
- Пример в parent POM:
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>5.3.30</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.14.0</version>
      <exclusions>
        <exclusion>
          <groupId>some.group</groupId>
          <artifactId>unwanted</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
  </dependencies>
</dependencyManagement>
```
- Такие записи **не добавляют** артефакт автоматически в сборку. Они только определяют значения по умолчанию. Чтобы зависимость появилась в classpath, её нужно объявить в `dependencies`:
```xml
<dependencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <!-- версия не указана — будет взята из dependencyManagement -->
  </dependency>
</dependencies>
```
- Часто используется в:
  - **parent POM** для унификации версий во всех модулях multi-module проекта;
  - **BOM (Bill of Materials)** — POM, который импортируют через `<scope>import</scope>`, например `spring-boot-dependencies`.

---

## Управление транзитивными зависимостями
- Если некоторый модуль A тянет в транзите артефакт B (через зависимость C), вы можете контролировать версию B, разместив B в `dependencyManagement` вашего POM с нужной версией. Тогда при разрешении транзитивной зависимости Maven использует версию, заданную в `dependencyManagement`.  
- Это полезно для **фиксирования версий транзитивных зависимостей** без необходимости явно объявлять их в `dependencies`.

Пример:
```xml
<!-- parent -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.36</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<!-- child, не объявляет slf4j прямо -->
<dependencies>
  <dependency>
    <groupId>org.some</groupId>
    <artifactId>library-that-uses-slf4j</artifactId>
    <version>1.0</version>
  </dependency>
</dependencies>
```
Если `library-that-uses-slf4j` тянет `slf4j-api` — будет использована версия `1.7.36` из `dependencyManagement`.

---

## BOM (Bill of Materials) и `<scope>import</scope>`
- BOM — специальный POM, который предоставляет секцию `dependencyManagement` с набором согласованных версий. Часто используемый пример — `spring-boot-dependencies`.
- Импорт BOM в `dependencyManagement`:
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```
- После импорта вы можете добавлять зависимости без указания версии — версия будет взята из BOM.

---

## Что можно и чего нельзя ожидать

**Можно ожидать:**
- Единый источник правды для версий и `exclusions`, `scope`, `optional`, `type`, `classifier` (если указано) — эти значения станут значениями по умолчанию для соответствующих зависимостей, когда они объявлены в `dependencies`.
- Контролировать версии транзитивных зависимостей (через перечисление их в `dependencyManagement`).

**Нельзя ожидать:**
- Запись в `dependencyManagement` не тянет артефакт в classpath — она не заменяет `dependencies`.
- Если дочерний POM явно укажет свою версию/scope/исключения — эти значения переопределят defaults из `dependencyManagement`.
- Если несколько источников задают версии (например, транзитивные зависимости и dependencyManagement), поведение разрешения версий подчиняется правилам Maven (ближайшая зависимость обычно выигрывает, но dependencyManagement может принудительно задать версию для транзитивной).

---

## Примеры (реальные сценарии)

### Сценарий 1. Parent POM управляет версиями для модулей
**parent/pom.xml**
```xml
<project>
  ...
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.14.0</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```
**module-a/pom.xml**
```xml
<dependencies>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <!-- версия не указана — взята из parent -->
  </dependency>
</dependencies>
```

### Сценарий 2. Зафиксировать транзитивную зависимость
Если библиотека `X` транзитивно тянет `commons-logging:1.1`, а вы хотите использовать `commons-logging:1.2`:
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```
Maven применит эту версию при разрешении транзитивной зависимости, даже если вы не объявляете `commons-logging` в `dependencies`.

### Сценарий 3. Использование BOM (Spring Boot)
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- версия не нужна — берётся из BOM -->
  </dependency>
</dependencies>
```

---

## Частые ошибки и подводные камни
- Ожидание, что `dependencyManagement` автоматически добавит зависимости — нет. Нужно объявлять зависимости в `dependencies`.
- Путаница между `dependencyManagement` и `dependencies` в aggregator (сборочный POM). Если POM является лишь aggregator (modules) и не является parent, `dependencyManagement` всё равно может работать как источник версий для модулей при наследовании POM-структуры.
- Неявные версии транзитивных зависимостей: если вы хотите контролировать транзитивные версии, явно указывайте их в `dependencyManagement`.
- Полагаясь только на BOM, убедитесь, что вы импортировали BOM правильно (`type` + `scope` = `pom` + `import`) и что BOM действительно содержит нужные версии.

---

## Практические советы
- Используйте `dependencyManagement` в parent POM для унификации версий в монорепозитории. Это уменьшает вероятность конфликтов версий.
- Для внешних стэков (Spring Boot, Quarkus) применяйте BOM — это значительно упрощает совместимость библиотек.
- Для временного «патча» транзитивной зависимости можно добавить её в `dependencyManagement` с нужной версией, не объявляя её явно в `dependencies`.
- Всегда проверяйте итоговое дерево зависимостей: `mvn dependency:tree -Dverbose` или `mvn dependency:resolve` — чтобы понять, какие версии реальны.

---

## Короткая сводка
- `dependencies` — реально подключаемые зависимости проекта (попадают в classpath).  
- `dependencyManagement` — централизованные defaults/политики для зависимостей (не добавляет автоматически зависимости).  
- BOM = `dependencyManagement` из внешнего POM, импортируемого через `scope=import` и `type=pom`.

---


