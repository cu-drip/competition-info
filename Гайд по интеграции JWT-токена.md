# Гайд по интеграции JWT в проекте

## 1. Деплой секретного ключа в Docker Compose

Нужно добавить монтирование файла `jwt_secret.key` в контейнер и передавать путь через переменную окружения:

```yaml
# docker-compose.yaml
services:
  user-service:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - ./jwt_secret.key:/run/secrets/jwt_secret.key
    environment:
      - JWT_SECRET_PATH=/run/secrets/jwt_secret.key
```

---

## 2. Конфигурация пути до ключа

Добавь в `src/main/resources/application.yml` (или `application.properties`):

```yml
# application.yml
jwt:
  secret-file: ${JWT_SECRET_PATH}   # путь к смонтированному ключу
```

---

## 3. Создание бина `JwtConfig` для загрузки `SecretKey`

```java
package com.example.userservice.security;

import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.crypto.SecretKey;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;


@Configuration
public class JwtConfig {

    @Value("${jwt.secret-file}")
    private String secretPath;

    private byte[] secretBytes;

    @PostConstruct
    public void init() throws Exception {
        secretBytes = Files.readAllBytes(Path.of(secretPath));
    }

    @Bean
    public SecretKey jwtKey(@Value("${jwt.secret-file}") String path) throws IOException {
        return Keys.hmacShaKeyFor(Files.readAllBytes(Path.of(path)));
    }

}
```

---

## 4. Настройка `JwtTokenProvider`

Методы для валидации и извлечения данных из токена:

```java
package com.example.userservice.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.List;
import java.util.UUID;


@Component
public class JwtTokenProvider {
    private final SecretKey jwtKey;
    private final long jwtExpirationMs = 86400000; // 1 день, если нужно; иначе хранится в Auth-сервисе

    public JwtTokenProvider(SecretKey jwtKey) {
        this.jwtKey = jwtKey;
    }

    public String generateToken(UUID userId,
                                String email,
                                List<String> userRolesList) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + jwtExpirationMs);
        return Jwts.builder()
            .setSubject(userId.toString())
            .claim("email", email)
            .claim("roles", userRolesList)   
            .setIssuedAt(now)
            .setExpiration(expiry)
            .signWith(jwtKey, SignatureAlgorithm.HS256)
            .compact();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(jwtKey)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException ex) {
            return false;
        }
    }

    public UUID getUserIdFromToken(String token) {
        Claims c = Jwts.parserBuilder()
            .setSigningKey(jwtKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
        return UUID.fromString(c.getSubject());
    }

    @SuppressWarnings("unchecked")
    public List<String> getRolesFromToken(String token) {
        Claims c = Jwts.parserBuilder()
            .setSigningKey(jwtKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
        return (List<String>) c.get("roles");
    }
}
```

---

## 5. Реализация `JwtAuthFilter`

Нужно заменить `UserHeaderAuthFilter` на фильтр, который валидирует JWT:

```java
package com.example.userservice.security;

import com.example.userservice.model.User;
import com.example.userservice.repository.UserRepository;
import io.jsonwebtoken.JwtException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;



@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtTokenProvider jwtProvider;
    private final UserRepository userRepo;

    public JwtAuthFilter(JwtTokenProvider jwtProvider, UserRepository userRepo) {
        this.jwtProvider = jwtProvider;
        this.userRepo    = userRepo;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain)
            throws ServletException, IOException {
        String path = req.getRequestURI();
        boolean isPublic = path.startsWith("/api/auth/");

        if (!isPublic) {
            String header = req.getHeader("Authorization");
            if (header == null || !header.toLowerCase().startsWith("bearer ")) {
                res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                res.getWriter().write("{\"message\":\"401: Unauthorized\"}");
                return;
            }

            String token = header.substring(header.indexOf(' ') + 1);
            if (!jwtProvider.validateToken(token)) {
                res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                res.getWriter().write("{\"message\":\"401: Unauthorized\"}");
                return;
            }

            UUID userId = jwtProvider.getUserIdFromToken(token);
            User user = userRepo.findById(userId)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

            List<SimpleGrantedAuthority> authorities = jwtProvider.getRolesFromToken(token).stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());

            UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(user, null, authorities);
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }

        chain.doFilter(req, res);
    }
}
```

---

## 6. Обновление `SecurityConfig`

```java
package com.example.userservice.security;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {
    private final JwtAuthFilter jwtFilter;

    public SecurityConfig(JwtAuthFilter jwtFilter) {
        this.jwtFilter = jwtFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
          .csrf().disable()
          .sessionManagement()
             .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
          .and()
          .authorizeHttpRequests(authz -> authz
             .requestMatchers("/api/auth/**").permitAll()
             .anyRequest().authenticated()
          );
        http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```
---


## Итоговая схема работы

1. Клиент отправляет на защищённые эндпоинты header `Authorization: Bearer <token>`, полученный из аутентификационного сервиса.
2. `JwtAuthFilter` проверяет токен, извлекает `userId` и `roles`, загружает `User` из БД и устанавливает `Authentication`.
3. Контроллеры получают аутентифицированного пользователя через `@AuthenticationPrincipal` и могут проверить роли через `@PreAuthorize` и другие механизмы безопасности.
