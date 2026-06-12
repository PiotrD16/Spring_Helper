## Spring Security - plik konfiguracyjny
Plik ten musi zawierac definicje filterChaina. Przykład:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

  @Bean
  public SecurityFilterChain securityFilterChain (HttpSecurity http) throws Exception {
    // Przykładowe uprawnienia dla endpointu login:

    http.authorizeHttpRequests((requests) -> requests.requestMatchers("/", "/api/login")).permitAll() // ten endpoint zezwala na dostep dla kazdego
            .anyRequest().authenticated())                                                            // pozostale musza byc authenticated
            .formLogin((form) -> form.loginPage("/login").permitAll())                                // pod tym endpointem znajduje sie formularz logowania
            .logout(logout -> logout.permitAll()));                                                   // logowanie dla kazdego
    return http.build();
  }
}

```
