## Лекция 1. Java Core, Spring Boot, JPA, PostgreSQL и Flyway

### Введение

В этой лекции мы пройдем путь от основ Java Core до разработки современного REST‑приложения на Spring Boot с использованием JPA, Hibernate, PostgreSQL и Flyway. Текст ориентирован на разработчика, готовящегося к собеседованию, поэтому акцент сделан не только на том, «как писать код», но и на том, «почему так, а не иначе», какие есть старые и новые подходы, а также на типичных ошибках, из-за которых заваливают интервью.

Я сознательно буду избегать списков и писать цельным лекционным текстом, но при этом буду структурировать материал заголовками и вставлять исполняемые примеры кода, каждый из которых будет подробно разобран.

### 1. Основы Java Core, объекты, классы и неизменяемость

Java — объектно‑ориентированный язык, где основная единица абстракции — класс. Класс описывает состояние и поведение, а объект является конкретным экземпляром класса в памяти. На собеседованиях любят спрашивать не столько синтаксис, сколько понимание принципов: инкапсуляция, наследование, полиморфизм и композиция.

Рассмотрим простой пример обычного Java‑класса, который часто сравнивают с Java Record, чтобы продемонстрировать старый и новый подходы.

```java
public class Cat {

    private String name;
    private int numberOfLives;
    private String color;

    public Cat(String name, int numberOfLives, String color) {
        this.name = name;
        this.numberOfLives = numberOfLives;
        this.color = color;
    }

    public String getName() {
        return name;
    }

    public int getNumberOfLives() {
        return numberOfLives;
    }

    public String getColor() {
        return color;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Cat cat = (Cat) o;
        return numberOfLives == cat.numberOfLives
                && java.util.Objects.equals(name, cat.name)
                && java.util.Objects.equals(color, cat.color);
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(name, numberOfLives, color);
    }

    @Override
    public String toString() {
        return "Cat{" +
               "name='" + name + '\'' +
               ", numberOfLives=" + numberOfLives +
               ", color='" + color + '\'' +
               '}';
    }
}
```

В этом коде мы видим классический Java‑подход: поля, конструктор, геттеры, переопределения методов equals, hashCode и toString. Такой код абсолютно легален и до появления Records был единственным стандартным способом описания «простых DTO» или value‑объектов. Главный минус — огромное количество шаблонного кода, который легко испортить ошибкой, и который тяжело поддерживать. На собеседовании важно понимать, что equals и hashCode должны быть согласованы: объекты, которые равны по equals, обязаны иметь одинаковый hashCode, иначе вы получите странное поведение в коллекциях вроде HashSet и HashMap.

### 2. Java Records: современный способ описания неизменяемых данных

Records — это новая форма классов, появившаяся в Java 14 (preview) и стабилизировавшаяся в более поздних версиях. Они предназначены для компактного и безопасного описания неизменяемых структур данных. Вместо десятков строк, приведенных выше, тот же кот описывается так же, как в ваших заметках.

```java
public record Cat(String name, int numberOfLives, String color) {
}
```

Этот код компилируется в полноценный класс. Компилятор автоматически генерирует приватные финальные поля, конструктор, геттеры, equals, hashCode и toString. При этом класс неизменяемый: после создания экземпляра изменить его состояние нельзя, потому что поля final и сеттеры не генерируются. Реализация equals сравнивает записи по типу и по значениям компонент, а реализация toString выводит их в удобном виде, например Cat[name=Fluffy, numberOfLives=9, color=White].

Records обладают несколькими важными ограничениями, которые важно знать на собеседовании. Во‑первых, запись не может расширять другие классы, но может реализовывать интерфейсы. Во‑вторых, запись не может быть абстрактной и не может быть унаследована, так как неявно является final. В‑третьих, внутри записи можно объявлять только статические дополнительные поля; это логично, так как объект должен быть целиком определен компонентами заголовка record.

Частый вопрос на интервью касается конструктора с параметрами по умолчанию. В записи это можно сделать с помощью дополнительного конструктора, делегирующего вызов основному, как в примере из ваших заметок.

```java
public record Cat(String name, int numberOfLives, String color) {

    public Cat(String name, String color) {
        this(name, 9, color);
    }
}
```

Здесь создается перегруженный конструктор, который использует значение 9 в качестве значения по умолчанию для количества жизней. Такой код особенно удобен для DTO и конфигурационных объектов.

Еще одна особенность records — так называемый компактный конструктор, который вы также приводили в заметках. Компактный конструктор позволяет выполнять валидацию аргументов или преобразование значений, не дублируя список параметров.

```java
public record Cat(String name, int numberOfLives, String color) {

    public Cat {
        if (numberOfLives <= 0) {
            throw new IllegalArgumentException("A cat must have at least one life");
        }
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name must not be blank");
        }
    }
}
```

Внутри компактного конструктора доступны параметры name, numberOfLives и color, и любые проверки, произведенные внутри, применяются к создаваемому объекту. Это современный, идиоматичный способ валидировать неизменяемые объекты.

Важно понимать сравнение старого и нового подхода. Раньше любая DTO‑структура требовала много кода и часто дублировалась в проекте. Сегодня, если вам нужен неизменяемый объект без сложной логики и наследования, record — лучший выбор. Интервьюеры любят спрашивать: можно ли сделать record неизменяемым и при этом добавить собственную реализацию equals и hashCode. Ответ: да, вы можете переопределить equals и hashCode вручную, но делать это следует только если вам действительно нужно изменить семантику сравнения; в большинстве случаев дефолтной реализации достаточно, и в ваших заметках вы приводили пример hashCode на основе Objects.hash, который эквивалентен тому, что сгенерирует компилятор.

### 3. Optional и его корректное использование

Optional — это обертка, которая может содержать либо значение, либо ничего. Его цель — сделать отсутствие значения явным и уменьшить количество NullPointerException. Однако на собеседовании часто проверяют не только знание методов Optional, но и понимание, где его нужно применять, а где нет.

Хорошая практика — использовать Optional как тип возвращаемого значения метода, когда результат может отсутствовать. Классический пример — поиск сущности в репозитории.

```java
import java.util.Optional;

public interface UserRepository {

    Optional<User> findByEmail(String email);
}
```

Этот интерфейс позволяет явно выразить, что пользователь может не быть найден. Теперь в сервисе вы можете безопасно обработать результат.

```java
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public void printUserName(String email) {
        Optional<User> userOptional = userRepository.findByEmail(email);
        userOptional.ifPresent(user -> System.out.println(user.getName()));
    }
}
```

В этом примере используется метод ifPresent, который выполнит переданное лямбда‑выражение только в том случае, если значение существует. Старый подход заключался в том, чтобы вернуть null и заставить вызывающий код писать if (user != null) с риском забыть эту проверку. Новый подход делает отсутствие значения типобезопасным и очевидным на уровне сигнатуры метода.

В заметках у вас есть важная мысль: Optional не следует использовать как поле сущности или параметр метода. Это типичная ошибка на собеседованиях. Optional задуман как «одноразовая обертка» вокруг результата вычисления, а не как часть модели данных. Если вы делаете поле Optional<User> внутри класса, то усложняете сериализацию, JPA‑маппинг и чтение кода. Вместо этого достаточно обычного nullable‑поля или явного разделения сущностей.

Еще одна распространенная ошибка — слепой вызов get без проверки наличия значения. Это эквивалентно использованию null и приводит к NoSuchElementException. Вместо этого нужно комбинировать методы orElse, orElseGet, orElseThrow и map. Рассмотрим пример цепочки операций над Optional, похожий на тот, что вы приводили в конце заметок.

```java
import java.util.Optional;

public class OptionalExample {

    public static void main(String[] args) {
        Optional<String> name = Optional.of(" Alex  ");

        Optional<Integer> length = name
                .map(String::trim)
                .map(String::length);

        length.ifPresent(len -> System.out.println("Length = " + len));
    }
}
```

Здесь мы сначала обрезаем пробелы, затем считаем длину, при этом ни разу не достаем значение вручную из Optional. На собеседовании любят спрашивать: в чем разница между map и flatMap для Optional. Ответ: map применяет функцию, которая возвращает обычное значение, и заворачивает результат в новый Optional, а flatMap работает с функцией, которая уже возвращает Optional, и «выпрямляет» вложенность, предотвращая Optional<Optional<T>>.

Главный вывод: используйте Optional для возвращаемых значений, не храните его в полях и не передавайте как аргумент метода, избегайте вызова get без проверки и комбинируйте операции map, flatMap, orElseGet и orElseThrow, чтобы код был декларативным и безопасным.

### 4. Java Streams: декларативная обработка коллекций

Стримы предоставляют функциональный способ обработки коллекций: фильтрация, трансформация, агрегация и т.д. В ваших заметках есть пример фильтрации списка людей по id, и это хороший старт, чтобы сравнить старый и новый подход.

Рассмотрим классический императивный подход.

```java
import java.util.ArrayList;
import java.util.List;

public class PersonServiceOldStyle {

    private final List<Person> personList = new ArrayList<>();

    public Person findById(long id) {
        for (Person person : personList) {
            if (person.getId() == id) {
                return person;
            }
        }
        return null;
    }
}
```

Здесь мы явно создаем цикл, проверяем условие и возвращаем найденного человека или null. Такой код легко написать, но он шумный и склонен к ошибкам, связанным с null. Современный потоковый подход выглядит проще и безопаснее.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class PersonService {

    private static final List<Person> personList;

    static {
        personList = new ArrayList<>();
    }

    public Optional<Person> findById(long id) {
        return personList.stream()
                .filter(person -> person.getId() == id)
                .findFirst();
    }
}
```

В этом примере используется тот же идиоматический фрагмент, что и в ваших заметках: personList.stream().filter(...).findFirst(). При этом мы возвращаем Optional<Person>, а не null, что сразу делает контракт метода более безопасным. В заметках вы указывали два варианта: orElse(null) и orElseThrow. На собеседовании важно показать, что вы понимаете, когда какой вариант уместен.

Если отсутствие человека — нормальная ситуация, и вызывающий код может с этим справиться, то имеет смысл вернуть Optional и дать вызывающему решать, как поступать. Если же отсутствие сущности считается ошибкой уровня бизнес‑логики, имеет смысл сразу бросить исключение, как в примере ниже.

```java
public Person findByIdOrThrow(long id) {
    return personList.stream()
            .filter(person -> person.getId() == id)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Person with id " + id + " not found"));
}
```

Старый подход выглядел бы как проверка if (person == null) и выброс исключения, разбросанная по коду. Новый подход централизует решение в одном методе, делает его выразительным и безопасным.

Типичные ошибки при работе со Streams, о которых спрашивают на интервью, связаны с попыткой модифицировать коллекцию внутри стрима, забыть терминальную операцию или использовать стримы там, где простой цикл будет проще и производительнее. Помните, что stream не выполняется, пока не будет вызвана терминальная операция (forEach, collect, reduce, findFirst и т.д.).

### 5. Основы Spring Boot и аннотации стартеров

Spring Boot упростил создание приложений на базе Spring, предоставив автоконфигурацию, стартер‑зависимости и удобные аннотации. В ваших заметках есть ключевая аннотация @SpringBootApplication, и важно понимать, что она собой представляет.

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Аннотация @SpringBootApplication эквивалентна комбинации трех аннотаций: @EnableAutoConfiguration, @Configuration и @ComponentScan. Это значит, что класс с этой аннотацией является конфигурационным, включает автоконфигурацию Spring Boot и запускает сканирование компонентов в текущем пакете и вложенных пакетах. В заметках вы отдельно выписывали ComponentScan(basePackages = "com.amigoscode"), что используется, когда нужно явно задать пакет для сканирования. На собеседованиях часто спрашивают, как работает сканирование компонентов, и почему иногда не находится бин; правильный ответ — либо пакет не попадает в зону сканирования, либо отсутствует соответствующая аннотация (@Component, @Service, @Repository и т.п.).

Старый подход (до Spring Boot) предполагал явное объявление большого количества бинов в XML или через Java‑конфигурацию, ручную настройку DataSource, DispatcherServlet и так далее. Новый подход со стартерами (например, spring-boot-starter-web) позволяет подключить зависимости, а Spring Boot за вас создаст большинство стандартных бинов. В ваших заметках есть пример POM с зависимостью spring-boot-starter-web и управлением зависимостями через spring-boot-dependencies в dependencyManagement; это именно тот современный способ, который ожидают увидеть на реальном проекте.

### 6. Dependency Injection в Spring: конструктор против @Autowired

В заметках вы приводите два варианта внедрения зависимостей: через поле с аннотацией @Autowired и через конструктор. Современная рекомендация Spring и то, что любят слышать на собеседованиях, — использовать конструкторную инъекцию.

Рассмотрим старый подход с полевым внедрением.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;

@Controller
public class AppController {

    @Autowired
    private PersonDAO dao;
}
```

Здесь зависимость внедряется в приватное поле. Такой код работает, но у него есть минусы: сложнее писать модульные тесты, так как нужно использовать reflection, инициализация полей происходит после создания объекта, и многие инструменты считают такой код менее прозрачным.

Современный подход — использовать конструкторную инъекцию, как вы уже делаете в заметках.

```java
import org.springframework.stereotype.Controller;

@Controller
public class AppController {

    private final PersonDAO dao;

    public AppController(PersonDAO personDAO) {
        this.dao = personDAO;
    }
}
```

Spring сгенерирует бин AppController и автоматически подставит нужный бин PersonDAO в конструктор. В новых версиях Spring аннотация @Autowired на конструкторе не обязательна, если у класса всего один конструктор, и в ваших заметках вы прямо указывали, что @Autowired писалось перед конструкторами в старых версиях фреймворка. На собеседовании важно подчеркнуть, что конструкторная инъекция делает зависимости явными, упрощает тестирование и позволяет создавать неизменяемые объекты (поля final).

Еще один полезный прием из ваших заметок — возможность получить и вывести все имена бинов из контекста, используя ConfigurableApplicationContext. Это обычно нужно для отладки.

```java
import org.springframework.boot.SpringApplication;
import org.springframework.context.ConfigurableApplicationContext;

public class Main {

    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(Main.class, args);
        String[] names = ctx.getBeanDefinitionNames();
        for (String name : names) {
            System.out.println(name);
        }
    }
}
```

Этот пример показывает, что Spring‑контейнер управляет жизненным циклом бинов и хранит их определения.

### 7. Spring Beans и области видимости (Scopes)

В ваших заметках есть описание основных областей видимости бинов: singleton, prototype, request, session и global session. На собеседовании часто задают вопрос: чем отличаются singleton и prototype, и какие скопы вообще бывают в веб‑приложении.

Spring по умолчанию создает бины в области видимости singleton, то есть один экземпляр на контейнер. Если вы пишете обычный сервис, который не хранит состояния между вызовами, то это правильный выбор. Область prototype создает новый экземпляр бина при каждом запросе к контейнеру. Это используется редко, например, для объектов, которые должны быть короткоживущими и не разделяться между потоками.

В веб‑контексте становятся доступными дополнительные области: request, session и globalSession. Область request создаёт новый бин для каждого HTTP‑запроса и применяется, когда вы хотите хранить данные, живущие ровно столько же, сколько обрабатывается запрос. Область session привязывает бин к HTTP‑сессии, а globalSession используется в специфичных портлет‑окружениях.

Простейшая конфигурация бина с нестандартной областью видимости выглядит так.

```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope("prototype")
public class PrototypeBean {
}
```

Здесь при каждом запросе PrototypeBean из контекста будет создаваться новый объект. Типичная ошибка на собеседовании — не понимать, что прототипный бин, внедренный в singleton‑бин, создается только один раз, при создании singleton, и не пересоздается при каждом запросе. Для подобных кейсов нужно использовать либо ObjectFactory, либо scoped‑proxy, либо пересмотреть дизайн.

### 8. REST API и контроллеры в Spring Boot

В заметках у вас есть важное замечание: @RestController эквивалентен @Controller + @ResponseBody. Это означает, что все методы в таком контроллере по умолчанию возвращают данные прямо в тело HTTP‑ответа, а не представление (view). Для создания REST‑API на Spring Boot используется именно @RestController.

Рассмотрим базовый пример REST‑контроллера, который управляет сущностями Person и использует сервисный слой для бизнес‑логики.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("api/v1/people")
public class PeopleController {

    private final PeopleService peopleService;

    public PeopleController(PeopleService peopleService) {
        this.peopleService = peopleService;
    }

    @GetMapping("{id}")
    public ResponseEntity<Person> getPerson(@PathVariable Long id) {
        Person person = peopleService.getPersonById(id);
        return ResponseEntity.ok(person);
    }

    @PostMapping
    public ResponseEntity<Void> addPerson(@RequestBody PeopleRegistrationRequest request) {
        peopleService.addPerson(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```

Здесь мы используем @RequestMapping на уровне класса, чтобы задать базовый путь api/v1/people, и аннотации @GetMapping и @PostMapping на уровне методов. В заметках у вас есть пример @RequestMapping с указанием path и method, что было популярно в старых версиях Spring. Современный подход предпочитает специализированные аннотации @GetMapping, @PostMapping, @PutMapping и т.д., которые короче и читаются лучше.

Аннотация @RequestParam, которую вы упоминали в заметках, используется для получения параметров запроса, например /api/v1/people?name=Alex. Важно понимать разницу между @PathVariable и @RequestParam и уметь объяснить её на собеседовании.

### 9. Обработка ошибок в Spring: собственные исключения, @ResponseStatus и application.yml

В заметках у вас приведен пример собственного исключения NotFound с аннотацией @ResponseStatus(HttpStatus.NOT_FOUND). Это классический и по‑современному лаконичный способ пробросить HTTP‑код в ответ.

```java
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(code = HttpStatus.NOT_FOUND)
public class NotFoundException extends RuntimeException {

    public NotFoundException(String message) {
        super(message);
    }
}
```

Теперь, если где‑то в сервисе или репозитории вы сделаете orElseThrow(() -> new NotFoundException("Wrong id")), Spring автоматически вернет клиенту статус 404 и сообщение ошибки. Это лучше, чем возвращать null или 500 без пояснений. В заметках у вас есть пример использования orElseThrow с amigoscode.exceptions.NotFound, что соответствует этой идее.

Дополнительно вы отмечали изменение application.yml для включения сообщения об ошибке в ответ. Современная конфигурация может выглядеть так.

```yaml
server:
  port: 8080
  error:
    include-message: always
```

Это говорит Spring Boot, что в JSON‑ответ о ошибке нужно включать текст сообщения исключения. На собеседовании стоит упомянуть, что в продакшене нужно аккуратно относиться к тому, какие детали ошибки вы показываете клиенту, чтобы не раскрывать внутреннюю структуру приложения или конфиденциальную информацию.

Более продвинутый подход к обработке ошибок в Spring Boot — использование @ControllerAdvice и @ExceptionHandler для централизованной обработки исключений. Но уже умение правильно использовать @ResponseStatus и настраивать server.error.* в application.yml показывает, что вы понимаете базовый механизм.

### 10. JPA, Hibernate и репозитории

JPA (Java Persistence API) — это спецификация, а Hibernate — одно из самых популярных её реализаций. В ваших заметках много примеров аннотаций JPA: @Entity, @Table, @SequenceGenerator, @GeneratedValue, а также интерфейса репозитория, расширяющего JpaRepository. Интервьюеры часто проверяют, понимаете ли вы, как JPA отображает объекты на таблицы и как работает жизненный цикл сущности.

Рассмотрим пример сущности Customer с использованием последовательности для генерации идентификаторов, как в ваших конспектах.

```java
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.SequenceGenerator;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

@Entity
@Table(
        name = "customer",
        uniqueConstraints = {
                @UniqueConstraint(
                        name = "customer_email_unique",
                        columnNames = "email"
                )
        }
)
public class Customer {

    @Id
    @SequenceGenerator(
            name = "customer_id_sequence",
            sequenceName = "customer_id_sequence"
    )
    @GeneratedValue(
            strategy = GenerationType.SEQUENCE,
            generator = "customer_id_sequence"
    )
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    private Integer age;

    protected Customer() {
    }

    public Customer(String name, String email, Integer age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    public Integer getAge() {
        return age;
    }
}
```

Здесь используются те же аннотации, что вы выписывали в заметках: @Entity обозначает, что класс является сущностью JPA, @Table с uniqueConstraints создает ограничение уникальности на уровне базы, а @SequenceGenerator и @GeneratedValue настраивают использование последовательности customer_id_sequence. В ваших заметках есть и SQL‑пример создания последовательности вручную через create sequence и nextval, а также понимание того, что тип BIGSERIAL в PostgreSQL автоматически создает sequence. Понимание того, как аннотации JPA соотносятся с DDL‑операциями в PostgreSQL, очень ценится на собеседованиях.

Репозиторий для этой сущности в современном Spring Data‑подходе выглядит минималистично.

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface CustomerRepository extends JpaRepository<Customer, Long> {

    boolean existsByEmail(String email);
}
```

Этот интерфейс соответствует идее из ваших заметок: «Repository отвечает за взаимодействие с базой данных» и пример метода boolean existsPersonByEmail(String email). Здесь мы наследуемся от JpaRepository, получаем готовые CRUD‑методы и добавляем метод existsByEmail, который Spring Data реализует автоматически на основании имени. Старый подход предполагал ручное написание DAO‑классов и SQL‑запросов через JdbcTemplate, как в ваших примерах с var sql и jdbcTemplate.update. Сегодня, когда нет сложных требований к SQL, репозитории на основе JpaRepository — стандарт и best practice.

### 11. Настройка PostgreSQL и интеграция с Spring Boot и JPA

PostgreSQL — популярная реляционная база данных, которая часто используется вместе с Java и Spring Boot. В ваших заметках есть как команды для работы с psql (\l, \c, \d, работа с sequence), так и пример docker run для запуска Postgres в контейнере. На собеседовании полезно показать, что вы понимаете, как связать Spring Boot с PostgreSQL.

Современная конфигурация data source и JPA в application.yml для локальной разработки может выглядеть так, почти один в один с вашей записью.

```yaml
server:
  error:
    include-message: always

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/people?createDatabaseIfNotExist=true
    username: postgres
    password: password
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
    show-sql: true
```

Здесь мы настраиваем подключение к базе people, указываем логин и пароль, задаем диалект PostgreSQL и включаем вывод SQL в консоль. В ваших заметках в качестве ddl-auto часто фигурирует create-drop, что подходит для учебных проектов, но на продакшене используется validate или none, а миграции базы берут на себя инструменты вроде Flyway. На собеседовании важно это проговорить: ddl-auto=create-drop удобно для разработки, но опасно в проде.

Для запуска PostgreSQL в Docker можно использовать команду, которую вы сохраняли в файле.

```bash
docker run --name postgres --rm -p 5432:5432 \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_USER=postgres \
  -d postgres:16
```

Этот запуск поднимает контейнер с Postgres на порту 5432, что соответствует настройке в application.yml. Подключиться к базе можно через psql, используя команды \l для просмотра баз, \d для просмотра таблиц и различные операции с последовательностями и уникальными ограничениями, которые вы подробно фиксировали в конспекте (ALTER TABLE ... ADD CONSTRAINT ... UNIQUE (email)).

### 12. Flyway для миграций базы данных

Важная современная практика — не полагаться на автоматическое создание схемы Hibernate, а управлять изменениями базы данных через миграции. В ваших заметках есть детальное описание Flyway: зависимость в pom.xml, структура папки db/migration и нумерация файлов миграций V1__init.sql, V2__add_new_column.sql и так далее.

Для начала нужно подключить зависимости в Maven.

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

После этого Flyway по умолчанию будет искать SQL‑файлы миграций в папке src/main/resources/db/migration. Например, первая миграция V1__create_customer_table.sql может выглядеть так.

```sql
CREATE TABLE customer (
    id           BIGSERIAL PRIMARY KEY,
    name         VARCHAR(255) NOT NULL,
    email        VARCHAR(255) NOT NULL,
    age          INTEGER,
    created_at   TIMESTAMP DEFAULT NOW()
);

ALTER TABLE customer
    ADD CONSTRAINT customer_email_unique UNIQUE (email);
```

Этот SQL‑скрипт создает таблицу customer с типом BIGSERIAL для id, что автоматически создает sequence, как вы писали в заметках, и добавляет ограничение уникальности для email. Важно запомнить правило, которое вы сами записали: никогда не изменяйте файлы миграций, которые уже были применены к базе. Любое изменение нужно вносить через новую миграцию, например V2__add_status_to_customer.sql. Интервьюеры часто спрашивают про стратегию управления схемой БД, и упоминание Flyway с таким подходом — большой плюс.

В интеграционных тестах вы также можете явным образом запускать миграции, как у вас было в конспекте с Testcontainers, используя Flyway.configure().dataSource(...).load().migrate(). Это показывает, что вы понимаете, как обеспечить синхронизацию схемы базы данных даже в тестовой среде внутри Docker‑контейнеров.

### 13. Совместное использование Spring Boot, JPA, PostgreSQL и Flyway: пример малого приложения

Чтобы связать вместе все разобранные части, рассмотрим минимальный пример приложения, которое поднимает REST‑API для работы с сущностью Customer, использует PostgreSQL в Docker, JPA‑репозиторий и миграции Flyway.

Во‑первых, у нас есть сущность Customer и репозиторий CustomerRepository, которые мы уже рассмотрели. Во‑вторых, нам понадобится сервисный слой.

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CustomerService {

    private final CustomerRepository customerRepository;

    public CustomerService(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    @Transactional
    public void registerCustomer(CustomerRegistrationRequest request) {
        boolean emailTaken = customerRepository.existsByEmail(request.email());
        if (emailTaken) {
            throw new IllegalArgumentException("Email already taken");
        }
        Customer customer = new Customer(
                request.name(),
                request.email(),
                request.age()
        );
        customerRepository.save(customer);
    }

    @Transactional(readOnly = true)
    public Customer getCustomer(Long id) {
        return customerRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("Customer with id " + id + " not found"));
    }
}
```

Здесь мы используем современный Java‑подход с record для DTO запроса, аналогичный PeopleRegistrationRequest из ваших заметок, и аннотацию @Transactional для управления транзакциями. Старый подход предполагал ручное управление транзакциями через EntityManager и begin/commit; сегодня в Spring Boot достаточно правильно аннотировать методы сервисного слоя.

Класс CustomerRegistrationRequest можно оформить как record, что хорошо согласуется с идеей неизменяемых DTO.

```java
public record CustomerRegistrationRequest(
        String name,
        String email,
        Integer age
) {
}
```

Наконец, REST‑контроллер связывает HTTP‑слой с сервисом.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("api/v1/customers")
public class CustomerController {

    private final CustomerService customerService;

    public CustomerController(CustomerService customerService) {
        this.customerService = customerService;
    }

    @PostMapping
    public ResponseEntity<Void> register(@RequestBody CustomerRegistrationRequest request) {
        customerService.registerCustomer(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @GetMapping("{id}")
    public ResponseEntity<Customer> getCustomer(@PathVariable Long id) {
        Customer customer = customerService.getCustomer(id);
        return ResponseEntity.ok(customer);
    }
}
```

Этот пример показывает связку всех основных элементов: Spring Boot управляет жизненным циклом приложения и бинов, JPA и Hibernate обеспечивают ORM‑слой, PostgreSQL хранит данные, а Flyway версионирует схему. Важно, что мы используем современные практики: конструкторную инъекцию, record для DTO, централизованную обработку ошибок и миграции вместо ddl-auto=create-drop в продакшене.

### Заключение

В этой лекции мы прошлись по ключевым темам, отраженным в вашем файле с заметками: от Java Core, неизменяемых объектов и records до Optional и Streams, от базовых аннотаций Spring Boot, Dependency Injection и scopes до REST‑контроллеров, обработки ошибок, JPA с Hibernate, работы с PostgreSQL и организации миграций через Flyway. Для собеседований особенно важно не просто знать синтаксис, но и уметь объяснять мотивацию современных практик и отличать старые подходы от новых. Records позволяют упростить неизменяемые модели, Optional и Streams дают декларативность и безопасность, Spring Boot со стартер‑зависимостями и автоконфигурацией избавляет от рутинной настройки, а связка JPA, PostgreSQL и Flyway обеспечивает надежный и контролируемый доступ к данным. Если вы уверенно чувствуете себя с этим стеком, понимаете, как он работает изнутри, и можете приводить такие примеры кода с объяснениями, то вы уже на хорошем уровне для технического собеседования по Java и Spring.


