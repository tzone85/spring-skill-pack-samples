---
name: spring-security-jwt-resource-server
description: Use when adding JWT-based authentication to a Spring Boot REST API as a resource server (token issued elsewhere — Keycloak, Auth0, Cognito, internal IDP). Wires the modern Spring Security 6.x DSL correctly without legacy hacks.
---

# spring-security-jwt-resource-server

## When this fires

- New service needs to authenticate users via Bearer JWT.
- Token is issued by an external IDP (Keycloak, Auth0, Cognito, Okta, or in-house).
- You see `WebSecurityConfigurerAdapter` in existing code (deprecated, must migrate).
- You need role/scope-based authorization on endpoints.

## Dependencies

```kotlin
implementation("org.springframework.boot:spring-boot-starter-security")
implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
```

## Configuration — the only correct way for Spring Security 6.x

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http,
                                    Converter<Jwt, AbstractAuthenticationToken> jwtConverter) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)        // pure API; enable for cookie auth
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth -> oauth
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter)))
            .build();
    }

    @Bean
    Converter<Jwt, AbstractAuthenticationToken> jwtAuthConverter() {
        var scopes = new JwtGrantedAuthoritiesConverter();
        scopes.setAuthoritiesClaimName("scope");
        scopes.setAuthorityPrefix("SCOPE_");

        var roles = new Converter<Jwt, Collection<GrantedAuthority>>() {
            @Override public Collection<GrantedAuthority> convert(Jwt jwt) {
                Map<String, Object> realmAccess = jwt.getClaim("realm_access");
                if (realmAccess == null) return List.of();
                Collection<String> rs = (Collection<String>) realmAccess.getOrDefault("roles", List.of());
                return rs.stream().<GrantedAuthority>map(r -> new SimpleGrantedAuthority("ROLE_" + r)).toList();
            }
        };

        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            var all = new HashSet<>(scopes.convert(jwt));
            all.addAll(roles.convert(jwt));
            return all;
        });
        return converter;
    }
}
```

Properties:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${IDP_ISSUER}  # e.g. https://keycloak.example.com/realms/x
          # Spring fetches /.well-known/openid-configuration and JWKS automatically.
```

## Method-level security

```java
@PreAuthorize("hasAuthority('SCOPE_orders:write')")
public Order createOrder(...) { ... }

@PostAuthorize("returnObject.tenantId == authentication.principal.claims['tenant']")
public Order getOrder(String id) { ... }
```

## Testing

```java
@SpringBootTest @AutoConfigureMockMvc
class OrdersControllerTest {
    @Autowired MockMvc mvc;

    @Test
    void createOrder_forbidden_withoutScope() throws Exception {
        mvc.perform(post("/api/orders").with(jwt().jwt(j -> j.claim("scope", "orders:read"))))
           .andExpect(status().isForbidden());
    }

    @Test
    void createOrder_ok_withScope() throws Exception {
        mvc.perform(post("/api/orders").with(jwt().jwt(j -> j.claim("scope", "orders:write"))))
           .andExpect(status().isCreated());
    }
}
```

## Anti-patterns to refuse

- `WebSecurityConfigurerAdapter` — removed in Spring Security 6. Reject; migrate to `SecurityFilterChain` bean.
- `permitAll()` on `/api/**` "temporarily for dev" — gets shipped to prod. Reject; use profiles or feature toggles, never a permit-all.
- Validating JWT signature manually — duplicates what `oauth2ResourceServer` does, drift risk. Reject.
- Embedding the JWKS in resources — breaks key rotation. Reject; use `issuer-uri`.
- Storing the access token in `localStorage` (frontend concern but mention) — XSS-exploitable. Recommend httpOnly cookies via BFF pattern.

## See also

- `oauth2-client-flow` (when service IS the client)
- `method-level-security` (deeper @PreAuthorize/@PostAuthorize patterns)
- `owasp-top10-spring-checklist`
