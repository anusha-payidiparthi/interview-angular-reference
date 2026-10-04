# 08 — Spring Security

## 1. Core concepts

| Term | Meaning |
|---|---|
| **Authentication** | *Who are you?* Verifying identity (password, token, certificate) |
| **Authorization** | *What may you do?* Checking roles/authorities/permissions |
| **Principal** | The authenticated user |
| **`Authentication`** | Object holding principal, credentials and authorities |
| **`GrantedAuthority`** | A permission string (`ROLE_ADMIN`, `SCOPE_orders.read`, `orders:write`) |
| **`SecurityContext`** | Holds the current `Authentication`; stored in a `ThreadLocal` via `SecurityContextHolder` |

Adding `spring-boot-starter-security` alone secures **every** endpoint with HTTP Basic + form login
and a generated password printed at startup (user `user`).

## 2. Architecture: the filter chain

```
Request
  → Servlet container filters
  → DelegatingFilterProxy                (servlet filter that delegates to a Spring bean)
  → FilterChainProxy ("springSecurityFilterChain")
      → picks the first SecurityFilterChain whose matcher matches the request
      → runs its filters in order, e.g.:
          SecurityContextHolderFilter      load SecurityContext
          CorsFilter
          CsrfFilter
          LogoutFilter
          BearerTokenAuthenticationFilter / UsernamePasswordAuthenticationFilter / BasicAuthenticationFilter
          ExceptionTranslationFilter       AuthenticationException → 401, AccessDeniedException → 403
          AuthorizationFilter              checks authorizeHttpRequests rules
  → DispatcherServlet → controller (method security via AOP)
```

### Authentication flow (username/password)
```
Filter builds UsernamePasswordAuthenticationToken (unauthenticated)
  → AuthenticationManager (ProviderManager)
      → AuthenticationProvider (DaoAuthenticationProvider)
          → UserDetailsService.loadUserByUsername()
          → PasswordEncoder.matches(raw, hash)
      ← authenticated Authentication with authorities
  → stored in SecurityContextHolder (and session, if stateful)
```

## 3. Configuration (Spring Security 6/7 style)

`WebSecurityConfigurerAdapter` was **removed** in Security 6. You now declare beans, using the
lambda DSL (the `.and()` chaining is gone in Security 7).

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                      // enables @PreAuthorize etc.
class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .csrf(csrf -> csrf.disable())                         // stateless token API
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/orders").hasAuthority("SCOPE_orders.write")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
            .exceptionHandling(e -> e
                .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())   // 401
                .accessDeniedHandler(new BearerTokenAccessDeniedHandler()));           // 403
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();   // {bcrypt}... by default
    }
}
```
Rules are evaluated **in order; first match wins**, so put specific rules before general ones
and end with `anyRequest()`. Multiple `SecurityFilterChain` beans (e.g. `/api/**` stateless JWT,
everything else form login) are ordered with `@Order`.

### `hasRole` vs `hasAuthority`
`hasRole("ADMIN")` checks for authority `ROLE_ADMIN` (prefix added automatically).
`hasAuthority("ROLE_ADMIN")` checks the exact string. Roles are coarse groups; authorities can be
fine-grained permissions or OAuth scopes (`SCOPE_...`).

## 4. Users from a database

```java
@Service
class JpaUserDetailsService implements UserDetailsService {
    private final UserRepository users;
    JpaUserDetailsService(UserRepository users) { this.users = users; }

    @Override
    public UserDetails loadUserByUsername(String username) {
        AppUser u = users.findByEmail(username)
                .orElseThrow(() -> new UsernameNotFoundException(username));
        return User.withUsername(u.getEmail())
                .password(u.getPasswordHash())                 // already BCrypt-hashed
                .roles(u.getRoles().toArray(String[]::new))
                .accountLocked(u.isLocked())
                .build();
    }
}
```
Boot wires a `UserDetailsService` + `PasswordEncoder` bean into `DaoAuthenticationProvider`
automatically. For tests/demos: `InMemoryUserDetailsManager`.

**Password storage:** never plain text or fast hashes (MD5/SHA-256). Use adaptive, salted hashes:
BCrypt (default), Argon2, SCrypt, PBKDF2. `DelegatingPasswordEncoder` stores the algorithm prefix
(`{bcrypt}$2a$10$...`) so you can migrate algorithms later.

## 5. Sessions vs tokens

| | Session (stateful) | Token / JWT (stateless) |
|---|---|---|
| State | Server keeps session; client has `JSESSIONID` cookie | Server keeps nothing; client sends `Authorization: Bearer <token>` |
| Scaling | Sticky sessions or shared store (Spring Session + Redis) | Any instance can validate |
| Revocation | Easy (delete session) | Hard: short expiry + refresh tokens, or a deny-list |
| CSRF | Needs protection (cookies sent automatically) | Not needed if token isn't in a cookie |
| Typical use | Server-rendered web apps, BFF | APIs, mobile, service-to-service |

## 6. JWT

A JWT is `base64url(header).base64url(payload).signature`. The payload holds **claims** (`sub`,
`exp`, `iat`, `iss`, `aud`, `scope`, roles). It's **signed, not encrypted** (anyone can read it), so
never put secrets in it.

Validation steps: verify the signature (HMAC shared secret, or RSA/EC public key from the issuer's
JWKS endpoint), check `exp`/`nbf`, `iss` and `aud`.

### Preferred: OAuth2 resource server (let an identity provider issue tokens)
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://login.example.com/realms/shop   # Keycloak, Okta, Auth0, Entra ID, Cognito
```
Spring fetches the JWKS keys, validates every token and maps `scope`/`scp` claims to `SCOPE_*`
authorities. Map custom role claims:
```java
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    var authorities = new JwtGrantedAuthoritiesConverter();
    authorities.setAuthoritiesClaimName("roles");
    authorities.setAuthorityPrefix("ROLE_");
    var converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authorities);
    return converter;
}
```

### Issuing your own JWTs (simple monolith)
```java
@RestController
class AuthController {
    private final AuthenticationManager authManager;
    private final JwtEncoder encoder;                    // NimbusJwtEncoder with an RSA key

    @PostMapping("/auth/login")
    TokenResponse login(@RequestBody LoginRequest req) {
        Authentication auth = authManager.authenticate(
                UsernamePasswordAuthenticationToken.unauthenticated(req.username(), req.password()));
        Instant now = Instant.now();
        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("shop-api")
                .subject(auth.getName())
                .issuedAt(now)
                .expiresAt(now.plus(Duration.ofMinutes(15)))
                .claim("roles", auth.getAuthorities().stream()
                        .map(a -> a.getAuthority().replace("ROLE_", "")).toList())
                .build();
        return new TokenResponse(encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue());
    }
}
```
Then validate them with `oauth2ResourceServer().jwt()` and a `JwtDecoder` built from the public
key. Interviewers also accept a custom `OncePerRequestFilter` that parses the header with a JWT
library and sets `SecurityContextHolder`, but the built-in resource server is the modern answer.

**Access + refresh tokens:** short-lived access token (5–15 min) + longer refresh token (stored
securely, rotated on use, revocable) to get new access tokens without re-login.

## 7. OAuth2 and OpenID Connect

- **OAuth2**: delegated **authorization**. Roles: resource owner (user), client (app),
  authorization server (issues tokens), resource server (your API).
- **OIDC**: identity layer on OAuth2; adds an **ID token** (JWT about the user) and `/userinfo`.
- Grant types: **Authorization Code + PKCE** (web/mobile/SPA users), **Client Credentials**
  (service-to-service), Refresh Token. Implicit and password grants are deprecated.

| Spring role | Starter | Config |
|---|---|---|
| Login with Google/Keycloak (web app) | `spring-boot-starter-oauth2-client` | `http.oauth2Login()` |
| API validating tokens | `spring-boot-starter-oauth2-resource-server` | `http.oauth2ResourceServer().jwt()` |
| Calling another API with a token | `oauth2-client` | `OAuth2AuthorizedClientManager`, `RestClient` interceptor |
| Your own authorization server | Spring Authorization Server | |

## 8. Method-level security

```java
@Service
class OrderService {
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAll() { ... }

    @PreAuthorize("hasAuthority('SCOPE_orders.read') and #customerId == authentication.name")
    public List<Order> forCustomer(String customerId) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.name")   // check after loading
    public Document get(Long id) { ... }

    @PreAuthorize("@orderSecurity.canEdit(#id, authentication)")    // delegate to a bean
    public void edit(Long id, OrderEdit edit) { ... }
}
```
Requires `@EnableMethodSecurity` (replaces `@EnableGlobalMethodSecurity`). Also available:
`@Secured("ROLE_ADMIN")`, `@RolesAllowed` (JSR-250), `@PreFilter`/`@PostFilter` on collections.
Method security is AOP-based, so the **self-invocation** trap applies.

Getting the current user:
```java
@GetMapping("/me")
UserDto me(@AuthenticationPrincipal Jwt jwt) { return new UserDto(jwt.getSubject()); }

Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

## 9. CSRF

**Cross-Site Request Forgery:** a malicious site makes the victim's browser send a request to your
site, and the browser automatically attaches your session cookie. Spring's `CsrfFilter` requires a
secret token (in a form field or `X-XSRF-TOKEN` header) on state-changing requests (POST/PUT/
PATCH/DELETE).

- **Keep it enabled** for browser apps using cookies/sessions (`CookieCsrfTokenRepository` for SPAs).
- **Safe to disable** for stateless APIs authenticated with `Authorization` headers (browsers don't
  attach those automatically).

## 10. CORS vs CSRF
CORS is a browser mechanism that *relaxes* the same-origin policy to let a different origin read
responses. CSRF protection *prevents* forged requests. Different problems; you often configure both.

## 11. Other security topics interviewers like

- **Security headers** (on by default): `X-Content-Type-Options`, `X-Frame-Options`,
  `Strict-Transport-Security`, `Cache-Control`. Add a `Content-Security-Policy` explicitly.
- **Rate limiting / brute force:** Bucket4j, gateway rate limiter, account lockout after N failures.
- **SQL injection:** use parameterized queries (JPA parameters, `JdbcClient.param`), never string
  concatenation.
- **Secrets:** externalize (see [03](./03-configuration-and-profiles.md)); rotate keys.
- **Actuator:** expose only `health`/`info` publicly; secure the rest.
- **Async/virtual threads:** `SecurityContext` is thread-local; use
  `DelegatingSecurityContextExecutor` or `SecurityContextHolder` strategy `MODE_INHERITABLETHREADLOCAL`
  carefully when spawning threads.

## 12. Testing security

```java
@WebMvcTest(OrderController.class)
@Import(SecurityConfig.class)
class OrderControllerSecurityTest {
    @Autowired MockMvc mvc;

    @Test
    void anonymousGets401() throws Exception {
        mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanDelete() throws Exception {
        mvc.perform(delete("/api/admin/orders/1").with(csrf())).andExpect(status().isNoContent());
    }

    @Test
    void jwtWithScope() throws Exception {
        mvc.perform(post("/api/orders")
                .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders.write")))
                .contentType(APPLICATION_JSON).content("{...}"))
           .andExpect(status().isCreated());
    }
}
```

## 13. Gotchas

1. Rules in the wrong order (`anyRequest()` first, or a broad matcher before a specific one).
2. `hasRole("ROLE_ADMIN")` → looks for `ROLE_ROLE_ADMIN`.
3. Disabling CSRF on a cookie-based web app.
4. Storing JWTs in `localStorage` exposes them to XSS; prefer `HttpOnly` cookies via a BFF for browsers.
5. Long-lived JWTs without a revocation strategy.
6. 401 vs 403 confusion: 401 = not authenticated, 403 = authenticated but not allowed.
7. Security exceptions don't reach `@ControllerAdvice`; configure entry point / access-denied handler.
8. CORS preflight blocked because `http.cors()` isn't enabled.

## 14. Interview questions

1. **How does Spring Security work internally?**
   A `DelegatingFilterProxy` hands requests to `FilterChainProxy`, which runs the matching
   `SecurityFilterChain`'s filters. Authentication filters build an `Authentication`, the
   `AuthenticationManager` delegates to `AuthenticationProvider`s, the result is stored in the
   `SecurityContextHolder`, and `AuthorizationFilter` checks access rules.

2. **Authentication vs authorization?**
   Proving identity vs checking permissions.

3. **How do you configure security in Spring Security 6+?**
   Declare a `SecurityFilterChain` bean using `HttpSecurity`'s lambda DSL;
   `WebSecurityConfigurerAdapter` was removed.

4. **What is `UserDetailsService`?**
   Interface with `loadUserByUsername` that loads user data (username, password hash, authorities)
   for `DaoAuthenticationProvider`.

5. **How do you store passwords?**
   Hashed with an adaptive salted algorithm (BCrypt/Argon2) via `PasswordEncoder`;
   `DelegatingPasswordEncoder` for algorithm upgrades.

6. **How do you implement JWT authentication?**
   Login endpoint authenticates and issues a signed short-lived JWT (or an IdP issues it); the API
   validates it with `oauth2ResourceServer().jwt()` (signature, expiry, issuer, audience), stateless
   sessions, CSRF disabled, roles mapped from claims.

7. **JWT pros and cons?**
   Pros: stateless, scalable, self-contained claims, works across services. Cons: hard to revoke,
   size, readable payload, key management. Mitigate with short expiry + refresh tokens.

8. **What is OAuth2? Name the grant types.**
   An authorization framework for delegated access via tokens. Authorization Code (+PKCE), Client
   Credentials, Refresh Token (Device Code for TVs/CLIs).

9. **OAuth2 vs OIDC?**
   OAuth2 is authorization (access tokens); OIDC adds authentication (ID token, user info).

10. **What is CSRF and when can you disable protection?**
    A forged cross-site request riding on the victim's cookies. Disable only for stateless APIs that
    don't use cookie authentication.

11. **`hasRole` vs `hasAuthority`?**
    `hasRole("X")` checks `ROLE_X`; `hasAuthority` checks the exact string.

12. **How do you secure individual methods?**
    `@EnableMethodSecurity` + `@PreAuthorize`/`@PostAuthorize` with SpEL, `@Secured`, `@RolesAllowed`.

13. **How do you get the logged-in user?**
    `@AuthenticationPrincipal` in a controller, `Principal`/`Authentication` parameters, or
    `SecurityContextHolder.getContext().getAuthentication()`.

14. **401 vs 403?**
    401 Unauthorized: missing/invalid credentials. 403 Forbidden: authenticated but lacking permission.

15. **How do you secure service-to-service calls?**
    OAuth2 Client Credentials tokens validated by the receiving resource server, or mTLS (often via a
    service mesh).

16. **How would you allow some endpoints publicly?**
    `requestMatchers(...).permitAll()` before `anyRequest().authenticated()`; a separate
    `SecurityFilterChain` for public paths.

17. **How do you test secured endpoints?**
    `spring-security-test`: `@WithMockUser`, `with(jwt())`, `with(csrf())`, `with(user(...))` on MockMvc.
