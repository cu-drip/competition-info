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
import jakarta.annotation.PostConstruct;
import javax.crypto.SecretKey;
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
    public SecretKey jwtKey() {
        return Keys.hmacShaKeyFor(secretBytes);
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
import javax.crypto.SecretKey;
import org.springframework.stereotype.Component;

import java.util.UUID;

@Component
public class JwtTokenProvider {

    private final SecretKey jwtKey;

    public JwtTokenProvider(SecretKey jwtKey) {
        this.jwtKey = jwtKey;
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

    public String getEmailFromToken(String token) {
        Claims c = Jwts.parserBuilder()
            .setSigningKey(jwtKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
        return c.get("email", String.class);
    }
}
```

---

## 5. Реализация `JwtAuthFilter`

Нужно заменить `UserHeaderAuthFilter` на фильтр, который валидирует JWT:

```java
package com.example.userservice.security;

import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;
import io.jsonwebtoken.Claims;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.UUID;
import com.example.userservice.model.User;
import com.example.userservice.repository.UserRepository;

@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtProvider;
    private final UserRepository userRepo;

    public JwtAuthFilter(JwtTokenProvider jwtProvider, UserRepository userRepo) {
        this.jwtProvider = jwtProvider;
        this.userRepo = userRepo;
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
            if (header == null || !header.startsWith("Bearer ")) {
                res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                res.getWriter().write("{"message":"Missing or invalid Authorization header"}");
                return;
            }

            String token = header.substring(7);
            if (!jwtProvider.validateToken(token)) {
                res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                res.getWriter().write("{"message":"Invalid or expired token"}");
                return;
            }

            UUID userId = jwtProvider.getUserIdFromToken(token);
            User user = userRepo.findById(userId)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

            UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(user, null, user.getRoles().stream()
                    .map(SimpleGrantedAuthority::new).toList());
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

## 7. Эндпоинты регистрации и логина

```java
package com.example.userservice.controller;

import com.example.userservice.model.User;
import com.example.userservice.service.UserService;
import com.example.userservice.security.JwtTokenProvider;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final UserService authService;
    private final JwtTokenProvider jwtProvider;

    public AuthController(UserService authService, JwtTokenProvider jwtProvider) {
        this.authService = authService;
        this.jwtProvider = jwtProvider;
    }

    public record RegisterRequest(String email, String password) {}
    public record LoginRequest(String email, String password) {}
    public record AuthResponse(String token) {}

    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@RequestBody RegisterRequest req) {
        User newUser = authService.register(req.email(), req.password());
        String token = jwtProvider.generateToken(newUser.getId(), newUser.getEmail());
        return ResponseEntity.ok(new AuthResponse(token));
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest req) {
        User user = authService.authenticate(req.email(), req.password());
        String token = jwtProvider.generateToken(user.getId(), user.getEmail());
        return ResponseEntity.ok(new AuthResponse(token));
    }
}
```

---

## Итог

1. Клиент регистрируется или логинится через `/api/auth/**` и получает JWT.  
2. На защищённых эндпоинтах отправляет `Authorization: Bearer <token>`.  
3. `JwtAuthFilter` валидирует токен, извлекает `userId`, загружает `User` и помещает в `SecurityContext`.  
4. Контроллеры получают аутентифицированного пользователя через `@AuthenticationPrincipal`.  
