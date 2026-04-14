# Spring Security & OAuth2 Reference

## Security Config (Spring Boot 3 / Spring Security 6)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity          // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)          // stateless API
            .sessionManagement(s ->
                s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter())))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
                .accessDeniedHandler(new BearerTokenAccessDeniedHandler()))
            .build();
    }

    @Bean
    JwtAuthenticationConverter jwtConverter() {
        var grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}
```

## OAuth2 Resource Server (JWT)

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OAUTH2_ISSUER_URI:https://auth.example.com}
          # OR explicit JWK set:
          jwk-set-uri: ${OAUTH2_JWK_URI:https://auth.example.com/.well-known/jwks.json}
```

### Custom Claims Extraction
```java
@Component
public class CustomJwtConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    @Override
    public AbstractAuthenticationToken convert(Jwt jwt) {
        Collection<GrantedAuthority> authorities = extractAuthorities(jwt);
        var principal = new UserPrincipal(
            jwt.getSubject(),
            jwt.getClaimAsString("email"),
            jwt.getClaimAsStringList("groups")
        );
        return new JwtAuthenticationToken(jwt, authorities, jwt.getSubject());
    }

    private Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
        // Keycloak-style: realm_access.roles
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess == null) return List.of();
        @SuppressWarnings("unchecked")
        List<String> roles = (List<String>) realmAccess.get("roles");
        return roles.stream()
            .map(r -> new SimpleGrantedAuthority("ROLE_" + r.toUpperCase()))
            .toList();
    }
}
```

## OAuth2 Client (calling downstream services)

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          inventory-service:
            client-id: ${INVENTORY_CLIENT_ID}
            client-secret: ${INVENTORY_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: inventory:read
        provider:
          inventory-service:
            token-uri: ${OAUTH2_TOKEN_URI}
```

```java
@Configuration
public class WebClientConfig {

    @Bean
    WebClient inventoryWebClient(OAuth2AuthorizedClientManager manager) {
        var filter = new ServletOAuth2AuthorizedClientExchangeFilterFunction(manager);
        filter.setDefaultClientRegistrationId("inventory-service");
        return WebClient.builder()
            .baseUrl("${inventory.base-url}")
            .apply(filter::configure)
            .build();
    }
}
```

## Method Security

```java
// Use at service layer; keep controllers thin
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
public UserProfile getProfile(String userId) { ... }

@PreAuthorize("hasAuthority('order:create')")
public Order createOrder(CreateOrderCmd cmd) { ... }

@PostAuthorize("returnObject.ownerId == authentication.name")
public Document getDocument(UUID id) { ... }

// Domain object security
@PreAuthorize("@orderSecurityService.canAccess(#id, authentication)")
public Order getOrder(UUID id) { ... }
```

## JWT Auth Server (Spring Authorization Server)

```java
// Only if you are BUILDING the auth server
@Configuration
@Import(OAuth2AuthorizationServerConfiguration.class)
public class AuthServerConfig {

    @Bean
    RegisteredClientRepository registeredClientRepository() {
        RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("spa-client")
            .clientSecret("{noop}secret")
            .clientAuthenticationMethod(CLIENT_SECRET_BASIC)
            .authorizationGrantType(AUTHORIZATION_CODE)
            .authorizationGrantType(REFRESH_TOKEN)
            .redirectUri("https://app.example.com/callback")
            .scope(OidcScopes.OPENID)
            .scope("orders:read")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofDays(7))
                .build())
            .build();
        return new InMemoryRegisteredClientRepository(client);
    }

    @Bean
    JWKSource<SecurityContext> jwkSource() {
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        RSAKey rsaKey = new RSAKey.Builder(publicKey)
            .privateKey(privateKey).keyID(UUID.randomUUID().toString()).build();
        return new ImmutableJWKSet<>(new JWKSet(rsaKey));
    }
}
```

## CORS

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET","POST","PUT","PATCH","DELETE","OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

## Security Context in Service Layer

```java
public interface SecurityContext {
    String currentUserId();
    boolean hasRole(String role);
}

@Component
public class SpringSecurityContext implements SecurityContext {

    @Override
    public String currentUserId() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName)
            .orElseThrow(() -> new IllegalStateException("No authenticated user"));
    }
}
```

## Checklist

- [ ] `open-in-view: false`
- [ ] CSRF disabled (stateless JWT API)
- [ ] Session policy = STATELESS
- [ ] Actuator health endpoint public; others secured
- [ ] Method security enabled (`@EnableMethodSecurity`)
- [ ] Token TTL ≤ 15 min for access tokens
- [ ] Refresh token rotation enabled
- [ ] Secrets in env vars, never in source
