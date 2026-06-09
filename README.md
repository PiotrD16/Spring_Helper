# Hibernate, JPA & Spring REST – Ściąga Konfiguracyjna

## Część 1: Hibernate & JPA (Warstwa Persystencji)

# Przechowywanie danych w aplikacjach Java – od kolekcji do baz danych 🗃️📊

W klasycznym podejściu (używając samej Javy) dane są przechowywane w różnych kolekcjach, np. w mapach. Jest to bardzo proste i z reguły dosyć efektywne rozwiązanie (np. kiedy wykorzystujemy HashMapy) podczas wykonywania operacji na danym zbiorze. Podejście to ma jeden podstawowy problem: dane te przechowywane są w pamięci RAM więc istnieją tak długo jak uruchomiona jest aplikacja. Po zakończeniu działania apki wszystko jest kasowane z pamięci. Kiepska sprawa zwłaszcza jeśli chcemy żeby dane były dostępne powiedzmy w każdej chwili. W tym celu wykorzystuje się narzędzia, które umożliwiają kontakt naszej aplikacji z bazą danych, w której dane są długożyjące. Do ustanawiania takiego połączenia wykorzystuje się **JDBC**. 🔌

## JDBC

**JDBC** to interfejs który stanowi most pomiędzy naszą aplikacją a bazą danych (np. MySQL, PostgreSQL). Jego podstawowym zadaniem jest ustanawianie połączenia między bazą danych, wykonywanie zapytań, przetwarzanie wyników. Warto wspomnieć, że zapytania są pisane ręcznie i może to być niewygodne na dłuższą metę i może to być niebezpieczne w sytuacji, gdy baza danych zmieni strukturę: wtedy trzeba również zmienić te zapytania w naszym kodzie Javy. Ponadto jeśli popełnimy gdzieś literówkę to całe zapytanie się nie wykona, dostaniemy wyjątek i będziemy musieli modyfikować kwerendę co jest mało wygodne. ⚠️

## JPA

Aby zniwelować ten problem wprowadzono **JPA (Jakarta Persistence API)** która jest warstwą abstrakcji lub też interfejsem którego zadaniem jest "dostarczenie" informacji, w jaki sposób możemy nawiązywać połączenie z bazą danych, jak zamieniać tabele baz danych na klasy i na odwrót, jednak brakuje implementacji tych funkcjonalności. Ma to sens, ponieważ dzięki takiemu podejściu łatwo jest utworzyć własne sposoby na interakcję z DB. Przykładem implementacji JPA jest **Hibernate**. 🧩

## Hibernate

Najprościej mówiąc, Hibernate jest konkretną, gotową do użycia implementacją JPA która umożliwia mapowanie klas Javy (POJO) na tabele w bazie danych (ORM). Jest to bardzo wygodne, ponieważ możemy zarządzać bazą danych i jej tabelami jak zwykłymi klasami Javy. Relację między JPA a Hibernatem możemy opisać następująco: JPA to kontrakt: mówi, co ma być wykonane (coś w stylu dokumentacji) a Hibernate jest po prostu realizacją tej dokumentacji z własną implementacją. Hibernate jest potężnym narzędziem o wielu ciekawych funkcjonalnościach, jednak najpierw warto wiedzieć na jakiej zasadzie on działa. 🚀

Bazy danych nie wiedzą, czym jest klasa w Javie: one jedynie rozumieją dane otrzymane w formie tabeli i vice versa: Java nie wie jak obsługiwać dane w postaci tabeli. Aby umożliwić komunikację tych dwóch warstw wykorzystuje się właśnie Hibernate'a i jego ORM'owe właściwości, czyli mapowanie klas z Javy na tabele w bazie danych. Ekosystem tego frameworku składa się z następujących elementów: 🔄

### Session 🫂

Pojęcie sesji możemy kojarzyć z codziennego życia: wchodzimy na stronę internetową, logujemy się na nasze konto (ustanawiamy sesję), przeglądamy coś i następnie się wylogowujemy (koniec sesji). W tym czasie wykonaliśmy jakieś działania, np. opublikowaliśmy jakiś post lub film, więc sesja została ustanowiona abyśmy mogli wykonać jakieś określone działania. W Hibernate jest bardzo podobnie: sesja to krótkie połączenie z bazą danych, które jest ustanawiane, aby wysłać do niej zapytania i wykonać jakieś operacje w bazie danych. Po ich zakończeniu sesja jest zakończona.

### PersistenceContext 📦

Tutaj znajdują się nasze encje (klasy z Javy), które są zarządzane przez **EntityManager** (o nim szerzej się wypowiem poniżej) tzn. może je tworzyć, modyfikować oraz usuwać. Element ten działa w ramach jednej sesji, tj. po zakończeniu jej, PersistenceContext jest usuwany. Warto wiedzieć, że encje w nim są unikalne i w przypadku, gdy chcemy drugi raz pobrać jakiś zestaw danych (tabelę) i znajduje się ona w PersistenceContext, nie zostanie wykonane zapytanie do bazy tylko obiekt zostanie pobrany z tego kontenera. Mechanizm ten znacznie usprawnia działanie aplikacji poprzez zredukowanie ilości zapytań do bazy danych. ⚡

### EntityManagerFactory 🏭

Jest to po prostu implementacja wzorca Fabryka którego zadaniem jest tworzenie danych obiektów w miejscu, w którym tego potrzebujemy. Wszystkie EntityManagery utworzone z jednego EntityManagerFactory będą miały to samo połączenie z bazą danych i te same ustawienia. Jeśli chcemy mieć np. 2 połączenia z różnymi bazami danych, musimy utworzyć nowy EntityManagerFactory.

### EntityManager 🎮

Jest to mechanizm, który ma kluczowe zadanie w kontekście działania Hibernate'a - ustanawia połączenie z bazą danych na określony czas (na wykonanie transakcji, czyli zestawu kilku zapytań do baz danych - każde z nich musi zakończyć się powodzeniem, inaczej całość jest cofana). To zadanie jest analogiczne do zadania **Session**. Innym ważnym zadaniem EntityManagera jest zarządzanie PersistenceContext, a co za tym idzie - naszymi encjami. Tutaj warto określić, w jakich stanach może być nasza encja:

#### Stany encji (Entity States) 🔄

![Diagram stanów encji Hibernate](https://www.baeldung.com/wp-content/uploads/2020/04/Hibernate-Entity-Lifecycle.png)

*Źródło grafiki: Baeldung.com*


- **Transient** 🆕 - obiekt został świeżo utworzony poprzez new(), np. `User user = new User();`
- **Managed** 🟢 - obiekt jest zarządzany przez EntityManager i znajduje się w PersistenceContext. Aby to osiągnąć, stosujemy metodę: `entityManager.persist(user)`. Taki obiekt nie wymaga stosowania `merge()` do aktualizacji zmian, ponieważ dzięki mechanizmowi **dirty-check** zmiany są aktualizowane automatycznie.
- **Detached** 🚪 - obiekt nie jest śledzony przez PersistenceContext, więc nie wchodzi w skład naszej transakcji. Zmiany również nie są widoczne więc aby dodać do kontekstu należy zastosować metodę `entityManager.merge(user)`. Po jej zastosowaniu powstaje kopia tego obiektu w PersistenceContext a zmiany są dociągane z bazy danych. Z kolei odłączenie użytkownika może również nastąpić manualnie: `entityManager.detach(user)`
- **Removed** ❌ - obiekt klasyfikuje się do usunięcia z bazy danych. Aby go zakwalifikować do tego wpisujemy: `entityManager.remove(user)`

W celu odszukania elementu w bazie danych (lub w PersistenceContext, o ile już się tam znajduje) po jego kluczu głównym warto wykorzystywać metodę `find()`:
`User user = entityManager.find(User.class, 123L)`. Tutaj jako pierwszy parametr podajemy naszą klasę, którą chcemy znaleźć a jako drugi argument podajemy klucz główny. 🔍

### Transaction 💰

Jest to informacja dla bazy danych, że tutaj rozpoczyna się transakcja, czyli zestaw zapytań do bazy danych. Jej najważniejszą cechą jest to, że jeśli przynajmniej jedna kwerenda z jakiegoś powodu nie zostanie wykonana (błąd bazy, błąd w samym zapisie kwerendy) reszta również zostanie wycofana i baza danych się nie zmieni. W nowoczesnym podejściu wystarczy oznaczyć metodę wykonującą zapytania jako `@Transactional` a Spring automatycznie rozpocznie transakcję i wykona commit lub rollback w zależności, czy wszystkie operacje się powiodły. 🔄✅❌

## Caching w Hibernate ⚡💾

Póki jesteśmy przy omawianiu PersistenceContext, warto opowiedzieć o **cache'owaniu w Hibernate**.

Caching polega na zapisaniu danych w pamięci podręcznej, aby zapewnić sobie szybszy dostęp do nich (nie trzeba udawać się do odpowiedniej komórki w pamięci aby wydobyć z niej zmienną żeby ją zmodyfikować za każdym razem, można ją zapisać w swojej pamięci podręcznej aby ten dostęp był szybszy. Wątki tak robią jednak wiąże się to z pewnym niebezpieczeństwem czyli próbą nadpisania nieaktualnej zmiennej (race-condition)). ⚠️🏎️

Hibernate również wykorzystuje ten mechanizm (związany z zapisem danych z bazy danych) i posiada 3 poziomy cachingu:

### 1st level caching 🥇

To jest nasz PersistenceContext - zanim zostanie wykonane zapytanie do bazy danych, EntityManager sprawdza najpierw PersistenceContext: jeśli nie ma tam tego, czego szukamy, wykonywane jest zapytanie do bazy danych. Jest on zawsze uruchomiony. ✅

### 2nd level caching 🥈

Jest opcjonalny i domyślnie wyłączony. To jest po prostu rozszerzenie pierwszego stopnia: tym razem dane są dzielone między każdy obiekt EntityManagerFactory, a co za tym idzie: między każdą istniejącą sesją. Działa to tak: jeśli nie ma danego elementu w PersistenceContext, sprawdzamy 2nd level cache czyli obiekty dzielone między sesjami. Jeśli w innych sesjach nie ma tego obiektu, udajemy się do bazy danych. 🔄

### 3rd level caching 🥉

Technicznie nie ma czegoś takiego jak 3rd level caching: jest to bardziej rozszerzenie 2nd level cache, który dobrze sobie radzi jak wyszukujemy obiekty po ich kluczu głównym (po ID). A co w sytuacji gdy chcemy wyszukać obiekt po innym parametrze, np. nazwisku? Wtedy pomocny jest właśnie ten poziom cachingu. Tworzy on mapę w której kluczem są kwerendy i dane parametry np. `nazwisko = "kowalski"` a wartością skojarzone ID. Trzeba go jednak używać ostrożnie, ponieważ w sytuacji, gdy baza danych się zmienia, ten cache jest czyszczony, więc i tak będzie trzeba wykonać szereg zapytań do bazy danych. 🗺️⚠️

### 1. Mapowanie encji

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String fullName;
}

### 2. Strategie dziedziczenia – JOINED

@Entity
@Table(name = "users")
@Inheritance(strategy = InheritanceType.JOINED)
public class User { ... }

@Entity
@Table(name = "trainers")
@PrimaryKeyJoinColumn(name = "user_id")
public class Trainer extends User { ... }

@Entity
@Table(name = "trainees")
@PrimaryKeyJoinColumn(name = "user_id")
public class Trainee extends User { ... }

### 3. Relacje

#### Many-to-One / One-to-Many

@Entity
public class Training {
    @ManyToOne
    @JoinColumn(name = "trainer_id")
    private Trainer trainer;
}

@Entity
public class Trainer {
    @OneToMany(mappedBy = "trainer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Training> trainings;
}

#### Many-to-Many z @JoinTable

@Entity
public class Trainee {
    @ManyToMany
    @JoinTable(
        name = "trainee_trainer",
        joinColumns = @JoinColumn(name = "trainee_id"),
        inverseJoinColumns = @JoinColumn(name = "trainer_id")
    )
    private Set<Trainer> trainers;
}

### 4. Konfiguracja JPA w czystym Springu (The Holy Trinity)

@Configuration
@EnableTransactionManagement
public class PersistenceConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("pass");
        return new HikariDataSource(config);
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource());
        em.setPackagesToScan("com.example.entity");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        em.setJpaProperties(hibernateProperties());
        return em;
    }

    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }

    private Properties hibernateProperties() {
        Properties props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "update");
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect");
        props.setProperty("hibernate.show_sql", "true");
        return props;
    }
}

### 5. DAO z EntityManager

@Repository
public class UserDao {

    @PersistenceContext
    private EntityManager em;

    public User findById(Long id) {
        return em.find(User.class, id);
    }

    public User save(User user) {
        if (user.getId() == null) {
            em.persist(user);
            return user;
        } else {
            return em.merge(user);
        }
    }

    public void delete(User user) {
        em.remove(em.contains(user) ? user : em.merge(user));
    }
}

### 6. JPQL – dynamiczne filtry, podzapytania, JOIN FETCH

public List<Training> findByFilters(String name, Long trainerId) {
    StringBuilder jpql = new StringBuilder("SELECT t FROM Training t WHERE 1=1");
    Map<String, Object> params = new HashMap<>();

    if (name != null) {
        jpql.append(" AND t.name LIKE :name");
        params.put("name", "%" + name + "%");
    }
    if (trainerId != null) {
        jpql.append(" AND t.trainer.id = :trainerId");
        params.put("trainerId", trainerId);
    }

    TypedQuery<Training> query = em.createQuery(jpql.toString(), Training.class);
    params.forEach(query::setParameter);
    return query.getResultList();
}

// Podzapytanie NOT EXISTS
String jpql = "SELECT t FROM Trainer t WHERE NOT EXISTS (SELECT tr FROM Trainee tr WHERE tr.trainer = t)";

// JOIN FETCH – przeciwdziałanie N+1
String jpql = "SELECT t FROM Trainer t JOIN FETCH t.trainings WHERE t.id = :id";

## Część 2: REST API & Spring Web

### 1. Podstawy REST

- Bezstanowość – każde żądanie zawiera pełny kontekst.
- Zasoby – identyfikowane przez URI.
- Poziomy Richardsona:
  - Level 0 – jeden URI, jedna metoda (np. POST /service)
  - Level 1 – zasoby (np. POST /users, POST /users/1/orders)
  - Level 2 – metody HTTP + kody statusu
  - Level 3 – HATEOAS (linki w odpowiedzi)

### 2. Metody HTTP – bezpieczeństwo i idempotentność

| Metoda | Bezpieczna | Idempotentna |
|--------|------------|---------------|
| GET    | ✅         | ✅            |
| POST   | ❌         | ❌            |
| PUT    | ❌         | ✅            |
| PATCH  | ❌         | ❌            |
| DELETE | ❌         | ✅            |

### 3. Kody statusu HTTP

- 200 OK – GET, PUT, DELETE
- 201 Created – POST + nagłówek Location
- 204 No Content – udane DELETE / PUT bez treści
- 400 Bad Request – błąd walidacji
- 404 Not Found – brak zasobu
- 500 Internal Server Error

### 4. Konfiguracja Spring MVC (czysty Spring)

@Configuration
@EnableWebMvc
@ComponentScan("com.example.controller")
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MappingJackson2HttpMessageConverter());
    }
}

### 5. Hierarchia kontekstów

- Root WebApplicationContext – warstwa serwisowa, DAO, bezpieczeństwo (zwykle applicationContext.xml lub @Configuration z ContextLoaderListener)
- Servlet WebApplicationContext – kontrolery, @ControllerAdvice, HandlerMapping (ładowany przez DispatcherServlet)

### 6. Kontroler REST

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> findById(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<User> create(@RequestBody @Valid User user) {
        User saved = userService.save(user);
        return ResponseEntity.created(URI.create("/api/users/" + saved.getId())).body(saved);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<User>> findAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(userService.findAll(PageRequest.of(page, size)));
    }
}

### 7. Globalna obsługa błędów (@ControllerAdvice)

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }
}

### 8. Wersjonowanie API

#### W URI (najczęstsze)

```bash
/api/v1/users
/api/v2/users
```

#### W nagłówkach (Accept header)

```bash
GET /api/users
Accept: application/vnd.example.v1+json
```

### 9. CORS

```java
@RestController
@CrossOrigin(origins = "http://localhost:4200", methods = {RequestMethod.GET, RequestMethod.POST})
public class UserController { ... }
```

Lub globalnie:

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**").allowedOrigins("http://localhost:4200");
}
```

## Checklista – co wrzucić do projektu?

### Pliki konfiguracyjne

- [ ] PersistenceConfig.java – DataSource, EntityManagerFactory, TransactionManager
- [ ] WebConfig.java – @EnableWebMvc, message converters, CORS
- [ ] AppInitializer.java (lub web.xml) – konfiguracja DispatcherServlet i ContextLoaderListener

### Adnotacje – gdzie?

- @Entity – klasy encji
- @Repository – DAO
- @Service – warstwa logiki biznesowej
- @RestController – kontrolery
- @Transactional – na metodach serwisowych (nigdy na kontrolerach)
- @ControllerAdvice – globalna obsługa wyjątków
- @EnableTransactionManagement – na konfiguracji JPA

### Pamiętaj o unikaniu:

- N+1 problem → JOIN FETCH lub @EntityGraph
- LazyInitializationException → trzymaj transakcję na poziomie serwisu
- Ręcznego otwierania transakcji – używaj @Transactional
- Mieszania @PathVariable z ciałem w POST/PUT

## Przydatne adnotacje w pigułce

| Adnotacja | Zastosowanie |
|-----------|--------------|
| @Entity, @Table | Mapowanie encji |
| @Id, @GeneratedValue | Klucz główny |
| @OneToMany(mappedBy=...) | Relacja dwukierunkowa |
| @JoinTable | Many-to-Many |
| @PersistenceContext | Wstrzyknięcie EntityManager |
| @Transactional | Zarządzanie transakcjami |
| @RestController | Kontroler REST |
| @RequestMapping | Mapowanie ścieżek |
| @ExceptionHandler | Obsługa wyjątków w kontrolerze |
| @ControllerAdvice | Globalny handler |
| @CrossOrigin | CORS |
