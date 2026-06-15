# Spring Security
Jak wiadomo, każda aplikacja musi posiadać usługi zapewniające bezpieczeństwo oraz ograniczać dostęp do danych zasobów dla konkretnych użytkowników (np. nie możemy dodać posta lub znajomego na Facebooku jeśli nie jesteśmy zalogowani). Aby realizować podobne funkcjonalności w Springu, wykorzystujemy __Spring Security__.

Najpierw przyjrzyjmy się drodze jaką pokonuje _request_ zanim dotrze na serwer:

                                          Kontener Tomcata (W SpringBoot wbudowany) -> Filtry -> DispatcherServlet -> Kontrolery
                                          
Skoro już wiemy jak wygląda droga żądania na serwer omówmy czym są poszczególne elementy:
- _Tomcat_ - jest to serwer HTTP który jest punktem wejściowym i wyjściowym żądań - przyjmuje je i przekazuje je do DispatcherServleta chyba, że wcześniej żądanie musi przejść przez filtry, które sprawdzają czy to      zapytanie wgl może przejść dalej, doklejają dodatkowe informacje itd. Na koniec wysyła odpowiedź do klienta.
- _DispatcherServlet_ - najprościej mówiąc jest to klasa, która potrafi obsługiwać żądania HTTP: wyciąga z nich dane (za pomocą HandlerMappera) i przekazuje je do odpowiedniego kontrolera (za pomocą HandlerAdaptera). __Ciekawostka__: kiedyś na jeden kontroler przypadał jeden _Servlet_. NIe było to zbyt skalowalne więc postanowiono utworzyć jeden wielki DispatcherServlet który sam decyduje, gdzie ma trafić jakie żądanie.


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
