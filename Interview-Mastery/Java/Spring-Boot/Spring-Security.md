# Spring Security

---

## 1. Executive Summary

### What Is It?
Spring Security is the de-facto security framework for Spring applications. It provides authentication, authorization, and protection against common attacks (CSRF, session fixation, clickjacking).

### Core Features
- **Authentication** — who are you?
- **Authorization** — what can you do?
- **Protection** — CSRF, CORS, headers, XSS
- **Integration** — OAuth2, LDAP, JWT, SAML, Remember-Me

### Security Filter Chain
Spring Security is implemented as a chain of servlet filters:

```
Request → SecurityContextPersistenceFilter → LogoutFilter → 
UsernamePasswordAuthenticationFilter → BasicAuthenticationFilter → 
ExceptionTranslationFilter → FilterSecurityInterceptor → Controller
```

---

## 2. Core Theory

### Authentication Architecture

```
AuthenticationProvider ─── UserDetailsService
       │                         │
       │                    ┌────┴────┐
       │                    │  User   │
       │                    │ Details │
       │                    └─────────┘
       ▼
SecurityContextHolder (ThreadLocal)
       │
       ▼
SecurityContext
    └─ Authentication (Principal + Credentials + GrantedAuthorities)
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `SecurityContextHolder` | Stores current user (ThreadLocal) |
| `SecurityContext` | Holds Authentication object |
| `Authentication` | Principal, credentials, authorities |
| `GrantedAuthority` | Permission/role (e.g., ROLE_ADMIN) |
| `UserDetailsService` | Loads user from database |
| `PasswordEncoder` | Encodes/validates passwords |
| `AuthenticationManager` | Dispatches to providers |
| `AuthenticationProvider` | Attempts authentication |
| `AccessDecisionManager` | Authorizes access |
| `FilterChainProxy` | Manages security filter chain |

---

## 3. Production Code

### 3.1 Security Configuration (JWT + Role-Based)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 3.2 JWT Authentication Filter

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        
        try {
            String jwt = authHeader.substring(7);
            String userEmail = jwtService.extractUsername(jwt);
            
            if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(userEmail);
                
                if (jwtService.isTokenValid(jwt, user)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            user, null, user.getAuthorities());
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
            
            chain.doFilter(request, response);
        } catch (JwtException e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("Invalid or expired token");
        }
    }
}
```

### 3.3 Method-Level Security

```java
@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {
    
    @GetMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> getAllUsers() { /* ... */ }
    
    @PostMapping("/promote")
    @PreAuthorize("hasAuthority('admin:write')")
    public void promoteUser(@RequestBody UserId userId) { /* ... */ }
    
    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasRole('SUPER_ADMIN') or @securityService.canDelete(#id)")
    public void deleteUser(@PathVariable Long id) { /* ... */ }
    
    @GetMapping("/audit")
    @PostAuthorize("returnObject.owner == authentication.name")
    public AuditRecord getAudit(@PathVariable Long id) { /* ... */ }
}
```

### 3.4 UserDetailsService Implementation

```java
@Service
public class UserService implements UserDetailsService {
    private final UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepository.findByEmail(email)
            .map(UserPrincipal::new)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
    }
    
    public record UserPrincipal(User user) implements UserDetails {
        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return user.getRoles().stream()
                .flatMap(role -> role.getPermissions().stream())
                .map(perm -> new SimpleGrantedAuthority(perm.getName()))
                .toList();
        }
        
        @Override
        public String getPassword() { return user.getPassword(); }
        @Override
        public String getUsername() { return user.getEmail(); }
        @Override
        public boolean isEnabled() { return user.isActive(); }
        @Override public boolean isAccountNonExpired() { return true; }
        @Override public boolean isAccountNonLocked() { return !user.isLocked(); }
        @Override public boolean isCredentialsNonExpired() { return true; }
    }
}
```

---

## 4. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Storing plain text passwords | Data breach exposes all passwords | BCryptPasswordEncoder |
| 2 | Overly permissive CORS | XSS, data theft | Restrict origins/methods |
| 3 | Disabling CSRF for all endpoints | CSRF vulnerabilities | Only disable for REST APIs using tokens |
| 4 | Not validating JWT signature | Token forgery | Use proper signing |
| 5 | Storing JWT in localStorage | XSS can steal token | HTTP-only cookies |
| 6 | Hard-coded roles in code | Inflexible | Configurable permissions |
| 7 | ThreadLocal not cleaned | User leaks between requests | Spring Security handles this automatically |
| 8 | Public access to sensitive actuator endpoints | Info disclosure | Secure actuator endpoints |

---

## 5. Cheat Sheet

```
═══ SPRING SECURITY ══════════════════════════════════════════

┌─ DEPENDENCIES ─────────────────────────────────────────────┐
│ spring-boot-starter-security      — Core security           │
│ spring-security-oauth2-resource-server — OAuth2 resource   │
│ spring-security-oauth2-client     — OAuth2 client          │
│ spring-security-test              — Test utilities          │
└─────────────────────────────────────────────────────────────┘

┌─ COMMON CONFIG ────────────────────────────────────────────┐
│ .sessionManagement(s -> s.sessionCreationPolicy(STATELESS)) │
│ .csrf(AbstractHttpConfigurer::disable)    // For REST APIs │
│ .cors(Customizer.withDefaults())          // Enable CORS   │
│ .authorizeHttpRequests(auth -> auth.      // Define rules  │
│     .requestMatchers("/public").permitAll()                 │
│     .requestMatchers("/admin").hasRole("ADMIN")            │
│     .anyRequest().authenticated())                         │
└─────────────────────────────────────────────────────────────┘

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @PreAuthorize("hasRole('ADMIN')")       // Before method    │
│ @PostAuthorize("returnObject.owner == auth.name") // After │
│ @Secured("ROLE_ADMIN")                  // Simple role check│
│ @RolesAllowed("ADMIN")                  // JSR-250          │
│ @EnableMethodSecurity                   // Enable (Spring 6)│
└─────────────────────────────────────────────────────────────┘

┌─ PASSWORD ENCODERS ────────────────────────────────────────┐
│ BCryptPasswordEncoder   — Strong, adaptive (recommended)   │
│ Argon2PasswordEncoder   — Modern, memory-hard               │
│ SCryptPasswordEncoder   — CPU/memory-hard                   │
│ Pbkdf2PasswordEncoder   — NIST recommended                 │
└─────────────────────────────────────────────────────────────┘
```
