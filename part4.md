Spring Security в современном Java‑приложении — это не просто «фреймворк для логина и пароля». Это целая инфраструктура, которая берет на себя аутентификацию, авторизацию, работу с фильтрами HTTP, хранение информации о пользователе в контексте безопасности, интеграцию с JWT, OAuth2 и другими механизмами. Для собеседований очень важно не только уметь «скопировать конфиг из туториала», но и понимать, что реально происходит под капотом: какие объекты участвуют в процессе, как они связаны и где чаще всего допускают ошибки.

В основе Spring Security лежит цепочка фильтров `FilterChain`. Каждый запрос перед тем, как попасть в контроллеры, проходит через набор фильтров, один из которых — `UsernamePasswordAuthenticationFilter`, другой — ваш кастомный JWT‑фильтр. Фильтры отвечают за то, чтобы из запроса извлечь учетные данные (логин/пароль, токен, заголовки, куки), попытаться аутентифицировать пользователя, положить результат в `SecurityContext` и уже потом передать управление бизнес‑слою. Центральным объектом при этом является `Authentication`: он содержит данные о пользователе, его ролях и статусе аутентификации. Реальные данные о пользователе инкапсулируются в объекте `UserDetails`, а загрузкой этих данных по имени пользователя занимается `UserDetailsService`. Логика проверки логина и пароля (или других учетных данных) делегируется `AuthenticationProvider`, самый распространенный из которых — `DaoAuthenticationProvider`, работающий через `UserDetailsService` и `PasswordEncoder`.

При переходе к JWT ситуация меняется. Вместо серверной сессии, которая хранится, например, в памяти или Redis, мы начинаем жить в stateless‑модели: сервер не хранит состояние, каждая HTTP‑запрос несет в себе токен, а сервер на каждый запрос заново валидирует токен и восстанавливает пользователя. Это упрощает масштабирование, позволяет легко добавлять новые инстансы приложения, работает идеально для микросервисов и взаимодействия через REST. Но вместе с этим появляются новые риски: нельзя просто так «удалить» уже выданный токен, нужно аккуратно работать с секретным ключом, продумывать срок жизни токенов, хранение refresh‑токена, защиту от кражи токенов и повторного их использования.

Типичная архитектура безопасности с JWT в Spring Boot включает несколько ключевых элементов. Во‑первых, сущность пользователя, в которой хранится роль и прочие поля. Во‑вторых, `UserRepository`, через который мы ищем пользователей по e‑mail или username. В‑третьих, `UserDetailsService`, реализующий загрузку пользователя для Spring Security. В‑четвертых, сервис работы с JWT (`JwtService`), где реализована генерация токенов, извлечение claims, проверка подписи и срока действия. В‑пятых, кастомный фильтр `JwtAuthenticationFilter`, который на каждый запрос проверяет заголовок `Authorization`, достает оттуда токен, валидирует его и устанавливает `Authentication` в `SecurityContextHolder`. И, наконец, конфигурация безопасности `SecurityConfig` с `SecurityFilterChain`, где мы отключаем CSRF в stateless API, настраиваем разрешенные/защищенные эндпоинты, добавляем наш фильтр в цепочку, регистрируем `DaoAuthenticationProvider` и `BCryptPasswordEncoder`.

Рассмотрим по шагам, как из этих строительных блоков собрать реальную конфигурацию безопасности, пригодную для продакшена и понятную на собеседовании. Для примера будем использовать модель, близкую к той, что у тебя в конспекте: пользователь с ролью, репозиторий, JWT‑сервис, фильтр, конфигурация, сервис аутентификации, работа с logout и refresh‑токенами, а также таблица токенов для их отзыва.

Начнем с пользователя и ролей. Важно, чтобы пользователь либо реализовывал интерфейс `UserDetails`, либо мы оборачивали его в стандартный класс `org.springframework.security.core.userdetails.User`. В конспекте у тебя есть идея хранить роль в отдельном enum и возвращать `SimpleGrantedAuthority` на основе названия роли. Это классическая и ожидаемая реализация на собеседовании.

```java
package com.example.security.user;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority(role.name()));
    }

    @Override
    public String getUsername() {
        return email;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}

enum Role {
    USER,
    ADMIN
}
```

Здесь пользователь реализует `UserDetails`, а роль хранится как enum и транслируется в `SimpleGrantedAuthority`. На собеседовании часто проверяют, понимаешь ли ты связь между ролями и authorities: в Spring Security доступ обычно проверяется по authorities (`hasAuthority`, `hasRole`), а роль — это удобная форма, которая в итоге превращается в `SimpleGrantedAuthority`. Важно также понимать, почему мы используем `email` как `username`: это полностью допустимо, главное — согласованно использовать его и в сущности, и в `UserDetailsService`, и в JWT‑claims.

Далее нужен репозиторий для пользователей, который позволит искать по e‑mail. В конспекте это выражено как `Optional<User> findByEmail(String email)`. Это минимальный, но достаточный контракт для `UserDetailsService` и для аутентификационного сервиса.

```java
package com.example.security.user;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);
}
```

Реализация `UserDetailsService` строится вокруг этого репозитория. Spring Security ожидает, что `UserDetailsService` по имени пользователя (в нашем случае e‑mail) вернет объект `UserDetails` или бросит `UsernameNotFoundException`. На собеседовании любят спрашивать, как ты интегрируешь свою базу пользователей со Spring Security — именно через этот интерфейс.

```java
package com.example.security.user;

import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepository.findByEmail(username)
                .orElseThrow(() -> new UsernameNotFoundException("User with email %s not found".formatted(username)));
    }
}
```

Теперь переходим к JWT. В твоем конспекте уже есть основные идеи: `SECRET_KEY`, извлечение всех claims через `Jwts.parser().verifyWith(key).build().parseSignedClaims(token).getPayload()`, получение одного claim через функцию (`Claims::getSubject`), генерация токена с `issuedAt`, `expiration` и подписью с симметричным ключом. На собеседовании важно показать, что ты понимаешь структуру JWT: заголовок (alg, typ), payload с claims (subject, roles, custom claims) и подпись, которая вычисляется по схеме, зависящей от алгоритма (часто HS256). Также часто спрашивают, почему нельзя хранить в payload секретные данные: потому что payload кодируется в Base64Url, но не шифруется, его может прочитать любой, у кого есть токен.

Создадим сервис `JwtService`, который будет отвечать за все операции с токенами. Секретный ключ лучше хранить в `application.yml` и инжектить через `@Value` или `@ConfigurationProperties`. При этом ключ должен быть достаточно длинным и случайным, а не короткой строкой типа `secret`.

```java
package com.example.security.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import java.security.Key;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
@RequiredArgsConstructor
public class JwtService {

    private final long accessExpirationMillis;
    private final long refreshExpirationMillis;

    private final String secretKey;

    public JwtService(
            @Value("${application.security.jwt.secret-key}") String secretKey,
            @Value("${application.security.jwt.access-token.expiration}") long accessExpirationMillis,
            @Value("${application.security.jwt.refresh-token.expiration}") long refreshExpirationMillis
    ) {
        this.secretKey = secretKey;
        this.accessExpirationMillis = accessExpirationMillis;
        this.refreshExpirationMillis = refreshExpirationMillis;
    }

    public String extractEmail(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts
                .parser()
                .verifyWith(getSignInKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    private Key getSignInKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateAccessToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails, accessExpirationMillis);
    }

    public String generateRefreshToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails, refreshExpirationMillis);
    }

    public String generateToken(
            Map<String, Object> extraClaims,
            UserDetails userDetails,
            long expirationMillis
    ) {
        Date now = new Date(System.currentTimeMillis());
        Date expiration = new Date(System.currentTimeMillis() + expirationMillis);

        return Jwts
                .builder()
                .claims(extraClaims)
                .subject(userDetails.getUsername())
                .issuedAt(now)
                .expiration(expiration)
                .signWith(getSignInKey())
                .compact();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String email = extractEmail(token);
        return email.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        Date expiration = extractClaim(token, Claims::getExpiration);
        return expiration.before(new Date());
    }
}
```

Этот сервис реализует все ключевые операции: извлечение subject (у нас это e‑mail), генерацию access‑токена и refresh‑токена с разными сроками жизни, валидацию токена и проверку его истечения. На собеседовании полезно проговорить, почему access‑токен делают короткоживущим (например, 15 минут или час), а refresh‑токен — более долгим (неделя, месяц): кража access‑токена менее опасна, так как он быстро протухает; к refresh‑токену относятся как к «чему‑то более секретному» и хранят в HttpOnly‑куки или защищенном хранилище на клиенте.

Следующий элемент — JWT‑фильтр. В конспекте у тебя есть правильный скелет: наследуемся от `OncePerRequestFilter`, в `doFilterInternal` читаем заголовок `Authorization`, проверяем префикс `Bearer`, извлекаем токен, достаем из него e‑mail, загружаем пользователя через `UserDetailsService`, валидируем токен и при успехе создаем `UsernamePasswordAuthenticationToken` и кладем его в `SecurityContextHolder`. Очень важно на собеседовании показать, что ты понимаешь, зачем мы проверяем, что в `SecurityContext` еще нет аутентификации, и почему обязательно в конце вызывается `filterChain.doFilter`.

```java
package com.example.security.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        final String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);
        final String jwt;
        final String userEmail;

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        jwt = authHeader.substring(7);
        userEmail = jwtService.extractEmail(jwt);

        if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);

            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                                userDetails,
                                null,
                                userDetails.getAuthorities()
                        );
                authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request)
                );
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

Фильтр делает ровно то, что нужно в stateless‑архитектуре: если токен не передан или некорректен, он просто пропускает запрос дальше без аутентификации. Если токен валиден, то в `SecurityContext` появляется аутентифицированный пользователь, и дальше все механизмы авторизации (`@PreAuthorize`, `hasRole`, `hasAuthority`, `@Secured`) начинают работать автоматически. На собеседовании часто спрашивают, в каком месте цепочки фильтров лучше вставлять JWT‑фильтр: обычно его добавляют до `UsernamePasswordAuthenticationFilter`, чтобы токен‑аутентификация происходила раньше стандартной формы логина.

Теперь нужно связать все это в конфигурации безопасности. В новых версиях Spring Security мы используем бины `SecurityFilterChain`, а не наследуемся от `WebSecurityConfigurerAdapter`. Конфигурация из конспекта очень близка к правильной: отключаем CSRF для REST API, разрешаем доступ к `/api/v1/auth/**`, все остальное защищаем, настраиваем stateless‑политику сессий, регистрируем `authenticationProvider` и вставляем `jwtFilter` перед `UsernamePasswordAuthenticationFilter`. Также определяем `PasswordEncoder` и `AuthenticationManager`.

```java
package com.example.security.config;

import com.example.security.jwt.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/auth/**").permitAll()
                        .anyRequest().authenticated()
                )
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                )
                .authenticationProvider(authenticationProvider())
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Здесь используется `DaoAuthenticationProvider`, который будет проверять логин и пароль при аутентификации через эндпоинт `/api/v1/auth/login` (или любой другой, который ты реализуешь). К нему подключен `UserDetailsService` и `PasswordEncoder`. Очень частый вопрос на собеседовании: почему нельзя хранить пароль в базе в открытом виде и почему `BCrypt` — это де‑факто стандарт. Правильный ответ — потому что пароль должен храниться в виде безопасного хэша с солью, чтобы при утечке базы нельзя было легко восстановить исходные пароли, `BCrypt` специально рассчитан на то, чтобы быть «медленным» и устойчивым к атакам перебора.

Следующий важный элемент — сервис аутентификации, который будет обрабатывать регистрацию и логин, генерировать токены и, если нужно, сохранять их в базу для последующего отзыва. В конспекте у тебя есть примеры методов `register` и `authenticate`, а также идея хранить токены в таблице `Token`, помечая их как `isExpired` и `isRevoked`. Это хороший подход для реализации logout и централизованного отзыва токенов.

```java
package com.example.security.auth;

import com.example.security.jwt.JwtService;
import com.example.security.token.Token;
import com.example.security.token.TokenRepository;
import com.example.security.token.TokenType;
import com.example.security.user.Role;
import com.example.security.user.User;
import com.example.security.user.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final TokenRepository tokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;

    public AuthResponse register(RegisterRequest request) {
        User user = User.builder()
                .email(request.getEmail())
                .password(passwordEncoder.encode(request.getPassword()))
                .role(Role.USER)
                .build();

        userRepository.save(user);

        String accessToken = jwtService.generateAccessToken(user);
        String refreshToken = jwtService.generateRefreshToken(user);

        saveUserToken(user, accessToken);

        return AuthResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }

    public AuthResponse authenticate(AuthRequest request) {
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        request.getEmail(),
                        request.getPassword()
                )
        );

        User user = userRepository.findByEmail(request.getEmail())
                .orElseThrow();

        revokeAllUserTokens(user);

        String accessToken = jwtService.generateAccessToken(user);
        String refreshToken = jwtService.generateRefreshToken(user);

        saveUserToken(user, accessToken);

        return AuthResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }

    private void saveUserToken(User user, String accessToken) {
        Token token = Token.builder()
                .user(user)
                .token(accessToken)
                .tokenType(TokenType.BEARER)
                .expired(false)
                .revoked(false)
                .build();
        tokenRepository.save(token);
    }

    private void revokeAllUserTokens(User user) {
        var validUserTokens = tokenRepository.findAllValidTokensByUser(user.getId());
        if (validUserTokens.isEmpty()) {
            return;
        }
        validUserTokens.forEach(token -> {
            token.setExpired(true);
            token.setRevoked(true);
        });
        tokenRepository.saveAll(validUserTokens);
    }
}
```

Этот сервис реализует две ключевые операции. При регистрации создается новый пользователь с ролью `USER`, пароль шифруется через `PasswordEncoder`, генерируются access‑ и refresh‑токены, access‑токен сохраняется в таблицу токенов. При аутентификации сначала вызывается `AuthenticationManager`, который через `DaoAuthenticationProvider` проверяет логин и пароль. Затем загружается пользователь, все его действующие токены помечаются как отозванные и истекшие, генерируется новая пара access/refresh‑токенов и новый access‑токен сохраняется в базу. Вопрос, который часто поднимают на собеседовании: почему вообще нужен отдельный репозиторий токенов, если JWT самодостаточен и stateless. Ответ — чтобы реализовать logout и отзыв токена до истечения срока его действия; без этого ты не можешь «забыть» уже выданный токен.

Для работы с токенами в базе нужна соответствующая сущность и репозиторий. В конспекте ты уже описал идею: `Token` с полями `value`, `TokenType`, `isExpired`, `isRevoked`, `@ManyToOne User`, плюс метод в репозитории `findValidTokenByUserId`. Расширим это до полноценной реализации.

```java
package com.example.security.token;

import com.example.security.user.User;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "tokens")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Token {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String token;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TokenType tokenType;

    @Column(nullable = false)
    private boolean expired;

    @Column(nullable = false)
    private boolean revoked;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
}

enum TokenType {
    BEARER
}
```

```java
package com.example.security.token;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface TokenRepository extends JpaRepository<Token, Long> {

    @Query("""
            select t from Token t inner join t.user u
            where u.id = :userId and (t.expired = false and t.revoked = false)
            """)
    List<Token> findAllValidTokensByUser(@Param("userId") Long userId);

    Token findByToken(String token);
}
```

Теперь токен не просто существует на стороне клиента, но и отслеживается на стороне сервера. В JWT‑фильтре можно добавить логику, которая будет проверять, не помечен ли токен как `revoked` или `expired` в базе. Это уже не полностью stateless‑архитектура, но дает гибкость, необходимую для logout и отзыва токенов при, например, компрометации учетной записи.

```java
package com.example.security.jwt;

import com.example.security.token.TokenRepository;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilterWithTokenCheck extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    private final TokenRepository tokenRepository;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        final String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String jwt = authHeader.substring(7);
        String userEmail = jwtService.extractEmail(jwt);

        if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);
            boolean isTokenValid = tokenRepository.findByToken(jwt) != null
                    && !tokenRepository.findByToken(jwt).isExpired()
                    && !tokenRepository.findByToken(jwt).isRevoked();

            if (jwtService.isTokenValid(jwt, userDetails) && isTokenValid) {
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                                userDetails,
                                null,
                                userDetails.getAuthorities()
                        );
                authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request)
                );
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

Такое решение демонстрирует на собеседовании понимание реальных проблем с JWT: сам по себе токен не может быть отозван до истечения срока действия, но, сочетая его с базой валидных токенов, мы можем в любой момент пометить его как недействительный и прекратить принимать.

Вопрос logout в stateless‑модели тоже решается не через удаление серверной сессии, а через изменение состояния на стороне сервера (отзыв токена) и, по возможности, на стороне клиента (удаление токена/куки). В Spring Security можно использовать конфигурацию logout, чтобы связать URL logout с кастомной логикой.

```java
package com.example.security.config;

import com.example.security.token.TokenRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.logout.LogoutHandler;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

@Configuration
@RequiredArgsConstructor
public class LogoutConfig {

    private final TokenRepository tokenRepository;

    @Bean
    public SecurityFilterChain logoutFilterChain(HttpSecurity http) throws Exception {
        http.logout(logout -> logout
                .logoutUrl("/api/v1/auth/logout")
                .addLogoutHandler(logoutHandler())
                .logoutSuccessHandler((request, response, authentication) ->
                        SecurityContextHolder.clearContext())
        );
        return http.build();
    }

    @Bean
    public LogoutHandler logoutHandler() {
        return (request, response, authentication) -> {
            String authHeader = request.getHeader("Authorization");
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return;
            }
            String jwt = authHeader.substring(7);
            var storedToken = tokenRepository.findByToken(jwt);
            if (storedToken != null) {
                storedToken.setExpired(true);
                storedToken.setRevoked(true);
                tokenRepository.save(storedToken);
            }
        };
    }
}
```

Этот конфиг связывает URL `/api/v1/auth/logout` с нашим `LogoutHandler`, который ищет токен в заголовке запроса, находит его в базе и помечает как истекший и отозванный. После этого любой дальнейший запрос с тем же токеном будет отклонен JWT‑фильтром. На собеседовании здесь важно показать понимание того, что logout в stateless‑JWT‑модели — это не уничтожение серверной сессии, а фактическое добавление токена в список недействительных.

Отдельная важная тема — refresh‑токены и поток обновления access‑токена. В конспекте у тебя уже есть пример `AuthResponse` с полями `accessToken` и `refreshToken`, настройки в `application.yml` с отдельными значениями `access-token.expiration` и `refresh-token.expiration`, а также метод `refreshToken(HttpServletRequest request, HttpServletResponse response)` в `AuthService`, который читает refresh‑токен из заголовка `Authorization`, проверяет его, генерирует новый access‑токен и пишет новый ответ в `response.getOutputStream()` с помощью `ObjectMapper`. Такой поток работы часто обсуждают на собеседованиях.

```java
package com.example.security.auth;

import com.example.security.jwt.JwtService;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpHeaders;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.stereotype.Service;

import java.io.IOException;

@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    private final ObjectMapper objectMapper;

    public void refreshToken(HttpServletRequest request, HttpServletResponse response) throws IOException {
        final String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        String refreshToken = authHeader.substring(7);
        String userEmail = jwtService.extractEmail(refreshToken);

        if (userEmail != null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);
            if (jwtService.isTokenValid(refreshToken, userDetails)) {
                String newAccessToken = jwtService.generateAccessToken(userDetails);
                AuthResponse authResponse = AuthResponse.builder()
                        .accessToken(newAccessToken)
                        .refreshToken(refreshToken)
                        .build();
                objectMapper.writeValue(response.getOutputStream(), authResponse);
                return;
            }
        }

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    }
}
```

Схема работы такая: клиент вызывает, например, `/api/v1/auth/login`, получает пару токенов, использует access‑токен для обычных запросов. Когда access‑токен истекает, клиент обращается к `/api/v1/auth/refresh-token`, передавая refresh‑токен в заголовке `Authorization`. Сервер проверяет подпись и срок действия refresh‑токена, убеждается, что пользователь существует и не заблокирован, и генерирует новый access‑токен, возвращая его клиенту. На собеседовании важно отметить несколько моментов: refresh‑токен лучше хранить не в `localStorage`, а в HttpOnly‑куках (если это браузер); при ротации refresh‑токена можно каждый раз выдавать новый и помечать старый как отозванный; нельзя позволять бесконечное обновление цепочки, иначе окно компрометации будет бесконечным.

С точки зрения уязвимостей и лучших практик при использовании Spring Security и JWT есть несколько ключевых моментов, которые любят спрашивать. Первое — работа с секретным ключом. Нельзя хардкодить секрет в код, особенно в публичных репозиториях. Его нужно хранить в переменных окружения, секрет‑хранилищах (Vault, AWS Secrets Manager и т. д.) и подставлять через конфигурацию. Второе — срок жизни токена. Слишком долгоживущие access‑токены повышают риск, что украденный токен будет использован долго; слишком короткие — создают перегрузку на refresh‑эндпоинт и неудобство для пользователя. Компромисс — короткий access‑токен плюс refresh‑токен с более длинным сроком и хорошая стратегия ротации.

Третье — хранение токена на клиенте. Если это browser‑based SPA, то хранить токен в `localStorage` небезопасно из‑за XSS: скрипт, внедренный на страницу, может прочитать токен и отправить его атакующему. Гораздо безопаснее — HttpOnly‑куки, которые недоступны из JavaScript, плюс защита от CSRF (например, через `SameSite`, CSRF‑токены или разделение доменов). Четвертое — защита от повторного использования токена (replay‑атаки). Здесь помогает короткий срок жизни плюс blacklist/whitelist токенов на сервере и, при необходимости, хранение дополнительных claims, таких как `iat`, `jti`, IP‑адрес или fingerprint устройства.

Кроме того, общие уязвимости, такие как SQL‑инъекции, XSS и CSRF, никуда не деваются при использовании JWT. Для SQL‑инъекций нужно использовать ORM и параметризованные запросы, никогда не подставлять параметры напрямую в SQL‑строки. Для XSS — экранировать пользовательский ввод и вывод, использовать Content Security Policy и не доверять данным с клиента. Для CSRF — в классической форме‑логин‑сценарии нужны CSRF‑токены; в stateless‑API часто используют JWT в заголовке `Authorization` плюс CORS‑настройки и `SameSite`‑куки. На собеседованиях любят, когда ты явно говоришь, что JWT не решает эти проблемы сам по себе, он решает только вопрос безсессионной аутентификации.

Если говорить о типичных ошибках и ловушках на собеседованиях по Spring Security и JWT, то их несколько. Первая — непонимание разницы между аутентификацией и авторизацией: аутентификация отвечает на вопрос «кто ты», а авторизация — «что тебе разрешено». Вторая — путаница между ролями и authorities: `ROLE_ADMIN` как роль против `ADMIN` как authority и поведение методов `hasRole` и `hasAuthority`. Третья — неправильное использование `SecurityContextHolder` и незнание того, что контекст безопасности по умолчанию хранится в `ThreadLocal`, а при асинхронных вызовах и реактивном программировании нужно отдельное управление контекстом.

Четвертая ловушка — уверенность, что раз JWT подписан, его нельзя прочитать. Payload токена может прочитать любой, у кого есть токен, подпись лишь гарантирует неизменность, а не конфиденциальность. Пятая — попытка реализовать logout в stateless‑JWT‑архитектуре через удаление сессии на сервере, хотя никакой сессии нет, и важно уметь объяснить, как ты отзывать токены и очищать их на клиенте. Шестая — отсутствие понимания, как работает цепочка фильтров Spring Security и почему порядок фильтров важен. Реализуя свой `OncePerRequestFilter`, нужно не забывать вызывать `filterChain.doFilter` и правильно обрабатывать случаи, когда токена нет или он невалиден.

Наконец, хороший кандидат на собеседовании по безопасности с использованием Spring Security и JWT умеет объяснить весь поток запроса: клиент отправляет логин/пароль на `/api/v1/auth/login`, Spring Security через `AuthenticationManager` и `DaoAuthenticationProvider` валидирует учетные данные, `AuthService` генерирует access‑ и refresh‑токены и при необходимости сохраняет access‑токен в базу. Дальше все запросы несут access‑токен в заголовке `Authorization`, `JwtAuthenticationFilter` на каждом запросе извлекает токен, валидирует его подпись, срок действия и состояние в базе, восстанавливает пользователя и кладет его в `SecurityContext`. Если access‑токен истекает, клиент вызывает `/api/v1/auth/refresh-token` с refresh‑токеном, сервер проверяет его и выдает новый access‑токен. При logout клиент вызывает `/api/v1/auth/logout`, сервер помечает текущий токен как отозванный, и дальнейшее использование этого токена становится невозможным. Понимание этих шагов и умение связать их с конкретными классами и интерфейсами Spring Security — то, что обычно отличает сильного кандидата на security‑ориентированном собеседовании.


