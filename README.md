# Hibernate, JPA & Spring REST – Ściąga Konfiguracyjna

## Część 1: Hibernate & JPA (Warstwa Persystencji)

W klasycznym podejściu (używając samej Javy) dane są przechowywane w różncy kolekcjach, np. w mapach. Jest to bardzo proste i z reguły dosyć efektywne rozwiązanie (np. kiedy wykorzystujemy HashMapy) podczas wykonywania operacji na danym zbiorze. Podejście to ma jeden podstawowy problem: dane te przechowywane są w pamięci RAM więc istnieją tak długo jak uruchomiona jest aplikacja. Po zakończeniu działania apki wszystko jest kasowane z pamięci. Kiepska sprawa zwłaszcza jeśli chcemy żeby dane były dostępne powiedzmy w każdej chwili. W tym celu wykorzystuje się narzędzia, które umożliwiają kontakt naszej aplikacji z bazą danych, w której dane są długożyjące. Do ustanawiania takiego połączenia wykorzystuje się **JDBC**.

**JDBC** to interfejs który stanowi most pomiędzy naszą aplikacją a bazą danych (np. MySQL, PostgreSQL). Jego podstawowym zadaniem jest ustanawianie połączenia między bazą danych, wykonywanie zapytań, przetwarzanie wyników. Warto wspomnieć, że zapytania są pisane ręcznie i może to być niewygodne na dłuższą metę i może to być niebezpieczne w sytuacji, gdy baza danych zmieni strukturę: wtedy trzeba również zmienić te zapytania w naszym kodzie Javy. Ponadto jeśli popełnimy gdzieś literówkę to całe zapytanie się nie wykona, dostaniemy wyjątek i będziemy musieli modyfikować kwerendę co jest mało wygodne.
Aby zniwelować ten problem wprowadzono **JPA (Jakarta Persistence API)** która jest warstwą abstrakcji lub też interfejsem którego zadaniem jest "dostarczenie" informacji, w jaki sposób możemy nawiązywać połącznie z bazą danych, jak zamieniać tabele baz danych na klasy i na odwrót, jednak brakuje implementacji tych funkcjonalności. Ma to sens, ponieważ dzięki takiemu podejściu łatwo jest utworzyć własne sposoby na interakcję z DB. Przykładem implementacji JPA jest **Hibernate**.

Najprościej mówiąc, Hibernate jest konkretną, gotową do użycia implementacją JPA która umożlwia mapowanie klas Javy (POJO) na tabele w bazie danych (ORM). Jest to bardzo wygodne, ponieważ możemy zarządzać bazą danych i jej tabelami jak zwykłymi klasami Javy. Hibernate jest potężnym narzędziem o wielu ciekawych funkcjonalnościach, jednak najpierw warto wiedzieć na jakiej zasadzie on działa.

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
