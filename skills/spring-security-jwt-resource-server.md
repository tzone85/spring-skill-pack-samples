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

## Greenfield path

New service, choose JWT resource server from day 1:

1. Add `spring-boot-starter-oauth2-resource-server` to `build.gradle` / `pom.xml`.
2. Pick the IDP early (Keycloak, Auth0, Cognito, internal). Put the issuer URI in `application.yml` per profile; never hard-code.
3. Define scopes BEFORE you define endpoints. Each endpoint maps to one scope. Document scope catalogue in an ADR.
4. Add the `SecurityConfig` bean with explicit allow-list for `/actuator/health` and `/actuator/info` only. Everything else `authenticated()`.
5. CI gate: ArchUnit test that no controller method is annotated without either `@PreAuthorize` or being on the public allow-list.

The rule: an endpoint without an explicit authorization decision is a build failure, not a security review item.

## Brownfield path

Existing service running on Spring Security 5.x with `WebSecurityConfigurerAdapter`:

1. Pin the current behaviour: snapshot every endpoint's authorization rules + integration test coverage. Don't trust the source — verify via MockMvc.
2. Migrate the bean structure first (`SecurityFilterChain` replaces `configure(HttpSecurity)`). Keep behaviour identical. Run the snapshot tests. They pass.
3. Migrate `antMatchers` to `requestMatchers`. Same drill.
4. Replace any custom `JwtAuthenticationProvider` with the resource-server DSL. This is the riskiest step — diff the granted authorities before/after.
5. Delete the deprecated adapter only after the snapshot suite is green for two weeks AND a chaos run flipped through every role permutation.
6. Write the ADR describing what changed and why.

What NOT to do on brownfield: rewrite security in a single sprint; turn off endpoint auth "while you refactor"; trust that integration tests cover all the routes (they don't — verify with the runtime endpoint listing).
