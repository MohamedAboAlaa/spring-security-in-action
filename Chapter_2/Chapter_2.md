# Chapter 2 — Hello, Spring Security
### Revision Notes

---

## 1. The Big Picture: Why Spring Boot + Spring Security?

Spring Boot is an evolutionary step in Spring application development. It ships with **preconfigured defaults** so you only override what doesn't fit (**convention over configuration**). Before Boot, developers wrote dozens of repetitive config lines per app. This was tolerable in monoliths (configure once, rarely touch), but painful with service-oriented architectures and microservices, where every service needs its own configuration. Boot's autoconfiguration shortens setup, which is why it is popular for modern apps.

This chapter builds the first Spring Security app, uses the Boot defaults to introduce authentication, and shows how to override them. Chapters 3–6 go deeper into each responsibility introduced here.

**This chapter covers:**
- Creating your first project with Spring Security
- Designing simple functionality using the basic components for authentication and authorization
- The underlying concept and how to use it in a given project
- Applying the basic contracts and understanding how they are correlated
- Writing custom implementations for primary responsibilities
- Overriding Spring Boot's default configurations for Spring Security

**Steps followed in this chapter:**
1. Create a project with only Spring Security + web dependencies, and observe the default authentication/authorization behavior.
2. Add user management by overriding the defaults to define custom users and passwords.
3. Learn that all endpoints are authenticated by default, and that this can be customized.
4. Apply different configuration styles to the same setup to understand best practices.

**Key definition:**
> **HTTP Basic** is a way for a web app to authenticate a user with credentials (username and password) taken from the header of the HTTP request.

---

## 2. Starting Your First Project

The first project is a small web app exposing one REST endpoint. Without extra work, Spring Security secures it with HTTP Basic.

![](figure_2_1.png)
*(Initial app: `curl -u user:pass http://localhost:8080/hello` returns `200 OK Hello!` using HTTP Basic)*

> **NOTE** — By default the app has **two** authentication mechanisms: HTTP Basic and **Form Login**. Form Login is left for later chapters. If you open the URL in a browser you'll see a login form instead of the HTTP Basic pop-up box; don't be confused, the focus here is HTTP Basic.

> **NOTE** — There are several ways to create Spring Boot projects (some IDEs do it directly). The author recommends *Spring Boot: Up and Running* (Heckler), *Spring Boot in Practice* (Musib), and *Spring Start Here* (his own book).

> **NOTE** — Examples refer to the book's companion source code (https://www.manning.com/downloads/2105). Download it to unstick yourself or validate your solutions. Each example lists the `pom.xml` dependencies needed.

> **NOTE** — The examples are build-tool independent (Maven or Gradle); all are built with **Maven** for consistency.

Just adding the right dependencies makes Boot apply default configuration, **including a username and password**, at startup.

**Project:** `ssiach2-ex1` (an empty project). Only two dependencies are needed:

```xml
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
*Listing 2.1 — Spring Security dependencies for the first web app*

Goal: see the behavior of a default-configured app and understand which components make up that default and what they are for.

We need at least one endpoint to secure, so add `HelloController` in a `controllers` package inside the main namespace.

> **NOTE** — Spring Boot only scans for components in the package (and subpackages) of the class annotated `@SpringBootApplication`. For stereotype-annotated classes outside it, declare the location with `@ComponentScan`.

```java
@RestController
public class HelloController {
 @GetMapping("/hello")
 public String hello() {
 return "Hello!";
 }
}
```
*Listing 2.2 — The HelloController class and a REST endpoint*

| Annotation | Effect |
|---|---|
| `@RestController` | Registers the bean in the context; marks it as a web controller; the method's return value becomes the HTTP response body |
| `@GetMapping("/hello")` | Maps the `/hello` path to the method via a GET request |

**On startup the console prints a generated password:**
```
Using generated security password: 93a01cf0-794b-4b98-86ef-54860f36f7f3
```
- A **new password is generated on every run**.
- Use it with HTTP Basic to call any endpoint.

**Call without credentials:**
```bash
curl http://localhost:8080/hello
```
```json
{
 "status":401,
 "error":"Unauthorized",
 "message":"Unauthorized",
 "path":"/hello"
}
```

**Call with the default user (`user`) and generated password:**
```bash
curl -u user:93a01cf0-794b-4b98-86ef-54860f36f7f3 http://localhost:8080/hello
```
```
Hello!
```

> **NOTE** — cURL is used throughout the book for readability. You can use a GUI tool such as **Postman, Insomnia, or Bruno**; install one if your OS lacks it.

### ⚠️ Important: 401 vs 403

- **401 Unauthorized** is ambiguous by name; it is normally used for failed **authentication** (missing/incorrect credentials).
- **403 Forbidden** is normally used for failed **authorization**: the server identified the caller, but they lack the needed privileges.

### 2.1 Calling the endpoint with HTTP Basic (what `-u` does)

`curl -u` Base64-encodes `<username>:<password>` and sends it in the `Authorization` header, prefixed with `Basic`. It's important to know what the real request looks like, so build the header manually:

1. Encode `<username>:<password>` with Base64 (Linux / Git Bash; `-n` = no trailing newline; online tools like base64encode.org also work):

```bash
echo -n user:93a01cf0-794b-4b98-86ef-54860f36f7f3 | base64
```
```
dXNlcjo5M2EwMWNmMC03OTRiLTRiOTgtODZlZi01NDg2MGYzNmY3ZjM=
```

2. Use the result as the `Authorization` header value (same result as `-u`):

```bash
curl -H "Authorization: Basic dXNlcjo5M2EwMWNmMC03OTRiLTRiOTgtODZlZi01NDg2MGYzNmY3ZjM=" localhost:8080/hello
```
```
Hello!
```

**Takeaway:** the default project has no significant security configuration. It only proves the dependencies are in place and does little for authentication/authorization. It is a good starting point but ❌ **not** production-ready. Next: see what Boot configures for Spring Security, then how to override it.

---

## 3. The Big Picture of Spring Security Class Design

The main actors in authentication and authorization must be understood because you'll **override these preconfigured components** to fit your app. Only the high-level picture is given here; details come in later chapters.

In section 2 we had a default user and a random password per startup. That logic lives in components that Boot sets up depending on your dependencies (convention over configuration).

![](figure_2_2.png)
*(Core elements of Spring Security authentication and how they connect)*

**Flow (figure 2.2):**
1. The **authentication filter** captures the request and delegates authentication to the authentication manager; based on the response, it configures the security context.
2. The **authentication manager** takes on responsibility for authentication and uses the authentication provider.
3. The **authentication provider** implements the authentication logic.
4. It finds the user with a **user details service** (user management) and validates the password with a **password encoder** (password management).
5. The result is returned to the filter.
6. The **security context** keeps the authentication data after authentication, until the action ends. In a thread-per-request app, that usually means until the response is sent to the client.

| Component | Responsibility |
|---|---|
| Authentication filter | Captures request, delegates to manager, configures security context |
| Authentication manager | Delegates to the authentication provider |
| Authentication provider | Implements the authentication logic |
| `UserDetailsService` | User management |
| `PasswordEncoder` | Password management |
| Security context | Holds authentication data until the request ends |

The autoconfigured beans discussed next: `UserDetailsService` and `PasswordEncoder`.

### 3.1 UserDetailsService

- An object implementing `UserDetailsService` manages user details.
- The default implementation registers only **default credentials in the app's memory**: user `user` with a **UUID** password, generated randomly at Spring context load (app startup) and written to the console.
- ✅ Good as a proof of concept (shows the dependency is in place).
- ❌ Avoid in production: credentials are in memory only, not persisted.

### 3.2 PasswordEncoder

Does two things:
- **Encodes** a password (usually with an encryption or hashing algorithm)
- **Verifies** if a password matches an existing encoding

- Less obvious than `UserDetailsService`, but **mandatory** for the Basic authentication flow.
- The simplest implementation keeps passwords in plain text, without encoding (details in chapter 4).
- ⚠️ When you replace the default `UserDetailsService`, you **must also specify a `PasswordEncoder`**.

### 3.3 HTTP Basic authentication method

Spring Boot chooses **HTTP Basic access authentication**, the most straightforward method:
- The client sends username and password in the HTTP `Authorization` header.
- Header value = prefix `Basic` + Base64 encoding of `username:password` (colon-separated).

> **NOTE** — HTTP Basic does **not** offer confidentiality. Base64 is only an encoding for transfer convenience, **not encryption or hashing**. If intercepted in transit, anyone can read the credentials. Don't use HTTP Basic without at least **HTTPS**. See RFC 7617 (https://tools.ietf.org/html/rfc7617).

### 3.4 AuthenticationProvider

- The `AuthenticationProvider` defines the authentication logic and **delegates** user and password management.
- The default implementation uses the default `UserDetailsService` and `PasswordEncoder`.
- Implicitly, **all endpoints are secured**, so for this example we only add the endpoint.
- With only one user who can access every endpoint, there is little to do for authorization here.

### Sidebar: HTTP vs. HTTPS

- The examples use plain HTTP; in practice apps communicate **only over HTTPS**.
- Spring Security configuration is **the same** with HTTP or HTTPS, so HTTPS is not configured in the examples (to keep focus).
- HTTPS can be configured at the **application level**, via a **service mesh**, or at the **infrastructure level**. Boot makes application-level HTTPS easy.
- In any scenario you need a **certificate signed by a certification authority (CA)**. It lets the client know the response comes from the real server and wasn't intercepted. You can buy one, or for testing generate a **self-signed certificate** (e.g., with OpenSSL: https://www.openssl.org/).

**Step 1 — generate key and certificate:**
```bash
openssl req -newkey rsa:2048 -x509 -keyout key.pem -out cert.pem -days 365
```
- Prompts for a password and CA details; for a test certificate any data works, but **remember the password**.
- Outputs `key.pem` (private key) and `cert.pem` (public certificate).

> **NOTE** — The PDF text shows `-days 36524 Chapter 2...`; this is an extraction artifact where the page header ("24 Chapter 2") merged with the value. The Windows example later in the sidebar uses `-days 365`, so `365` is used above.

**Step 2 — build the PKCS12 certificate** (PKCS12 = Public Key Cryptography Standards #12, the most common format; **Java KeyStore (JKS)** is used less frequently). Recommended reading: *Real-World Cryptography* by David Wong (Manning, 2020).
```bash
openssl pkcs12 -export -in cert.pem -inkey key.pem -out certificate.p12 -name "certificate"
```

**Windows Bash shell:** prefix with `winpty`:
```bash
winpty openssl req -newkey rsa:2048 -x509 -keyout key.pem -out cert.pem -days 365
winpty openssl pkcs12 -export -in cert.pem -inkey key.pem -out certificate.p12 -name "certificate"
```

**Step 3 — configure HTTPS:** copy `certificate.p12` into the `resources` folder and add to `application.properties`:
```properties
server.ssl.key-store-type=PKCS12
server.ssl.key-store=classpath:certificate.p12
server.ssl.key-store-password=12345
```
- The password (here `12345`) is the one entered when running the **second** command (that is why it isn't visible in the command).

**Step 4 — test endpoint:**
```java
@RestController
public class HelloController {
 @GetMapping("/hello")
 public String hello() {
 return "Hello!";
 }
}
```

**Step 5 — call with HTTPS.** With a self-signed certificate, the tool must skip the authenticity check, otherwise it won't recognize the certificate and the call fails. In cURL use `-k`:
```bash
curl -k -u user:93a01cf0-794b-4b98-86ef-54860f36f7f3 https://localhost:8080/hello
```
```
Hello!
```

⚠️ HTTPS is **only one brick** in the security wall. Communication isn't bulletproof, and people who say "I'm not encrypting this anymore, I'll use HTTPS!" are mistaken. Take care of **all layers** of the system.

---

## 4. Overriding Default Configurations

Overriding the default components is how you plug in custom implementations and apply security to fit your app. It's also about writing configurations that stay **highly maintainable**.

- Often there are **multiple ways** to override the same configuration; this flexibility can confuse.
- Some developers use **beans in the Spring context**; others **override methods** for the same purpose. Rapid evolution of the Spring ecosystem produced these multiple approaches.
- ⚠️ **Mixing styles** in one app is undesirable: it hurts readability and maintainability.
- Knowing your options and when to use them is a valuable skill.

This section configures a `UserDetailsService` and a `PasswordEncoder`, the two components that usually take part in authentication and are customized in most apps (details in chapters 3 and 4). All implementations used here are provided by Spring Security.

### 4.1 Customizing User Details Management

Goal: define a custom `UserDetailsService` bean to override Boot's default. Chapter 3 covers creating your own implementation or using a predefined one; here we use the predefined **`InMemoryUserDetailsManager`** (a bit more than a plain `UserDetailsService`, but we treat it as one for now).

> **NOTE** — Interfaces in Java define **contracts** between objects and decouple them. The book mainly calls them "contracts".

> **NOTE** — `InMemoryUserDetailsManager` is ❌ not for production. It is an excellent tool for examples and proofs of concept, when all you need is users without implementing that part. Here it is used to learn how to override the default `UserDetailsService`.

**Project:** `ssia-ch2-ex2`. Put configuration classes in a separate `config` package.

```java
@Configuration // ← marks the class as a configuration class
public class ProjectConfig {

 @Bean // ← adds the returned value as a bean in the Spring context
 UserDetailsService userDetailsService() {
 return new InMemoryUserDetailsManager();
 }
}
```
*Listing 2.3 — Configuration class for the `UserDetailsService` bean*

- Run as-is: the autogenerated password **no longer appears** (your bean replaces the default).
- But the endpoint is **inaccessible**, for two reasons:
  - ❌ No users
  - ❌ No `PasswordEncoder`

**To fix (figure 2.2 showed authentication depends on a `PasswordEncoder`):**
1. Create at least one user with credentials (username + password)
2. Add the user to be managed by our `UserDetailsService`
3. Define a `PasswordEncoder` bean to verify passwords against those stored by `UserDetailsService`

Users are covered in chapter 3; for now use a predefined builder to create a `UserDetails` object.

> **NOTE** — `var` (reserved type name since **Java 10**) works only for **local declarations**. It's used to shorten syntax and hide the variable type so you can focus on what's relevant; hidden types are explained in later chapters. (The author admits it could be seen as bad clean-coding in some cases.)

A user needs a **username, a password, and at least one authority**. An **authority** is an action allowed for that user; any string works. Here it's `read`; the name doesn't matter yet because it isn't used.

```java
@Configuration
public class ProjectConfig {

 @Bean
 UserDetailsService userDetailsService() {
 var user = User.withUsername("john") // ← builds the user with username, password, authorities
 .password("12345")
 .authorities("read")
 .build();

 return new InMemoryUserDetailsManager(user); // ← adds the user to be managed by UserDetailsService
 }
}
```
*Listing 2.4 — Creating a user with the `User` builder class*

> **NOTE** — `User` is in `org.springframework.security.core.userdetails`; it's the builder used to create the user object. **General rule in the book:** if a class isn't shown being written in a listing, Spring Security provides it.

Still not enough; a `PasswordEncoder` is needed. With the default `UserDetailsService` a `PasswordEncoder` is autoconfigured; since we overrode the former, we must declare the latter. Calling now gives an exception in the console, and the client gets **HTTP 401 with an empty body**:

```bash
curl -u john:12345 http://localhost:8080/hello
```
```
java.lang.IllegalArgumentException:
There is no PasswordEncoder mapped for the id "null"
 at
org.springframework.security.crypto.
➥password.DelegatingPasswordEncoder$
➥UnmappedIdPasswordEncoder.matches(
➥DelegatingPasswordEncoder.java:289)
➥~[spring-security-crypto-6.0.0.jar:6.0.0]
 at org.springframework.security.crypto.
➥password.DelegatingPasswordEncoder.matches(
➥DelegatingPasswordEncoder.java:237)
➥~[spring-security-crypto-6.0.0.jar:6.0.0]
```

**Fix:** add a `PasswordEncoder` bean:
```java
@Bean
public PasswordEncoder passwordEncoder() {
 return NoOpPasswordEncoder.getInstance();
}
```

> **NOTE** — `NoOpPasswordEncoder` treats passwords as **plain text** (no encryption or hashing); matching just uses `String.equals(Object o)`. ❌ Not for production. Good for examples where you don't want to focus on hashing. Its developers marked it **`@Deprecated`**, so your IDE shows it with a strikethrough.

```java
@Configuration
public class ProjectConfig {

 @Bean
 UserDetailsService userDetailsService() {
 var user = User.withUsername("john")
 .password("12345")
 .authorities("read")
 .build();

 return new InMemoryUserDetailsManager(user);
 }

 @Bean // ← new @Bean method adds a PasswordEncoder to the context
 PasswordEncoder passwordEncoder() {
 return NoOpPasswordEncoder.getInstance();
 }
}
```
*Listing 2.5 — Full definition of the configuration class*

```bash
curl -u john:12345 http://localhost:8080/hello
```
```
Hello!
```

> **NOTE** — Tests are not covered in the examples to keep focus, but integration tests for Spring Security are provided with all book examples. Testing is discussed in **chapter 18**.

### 4.2 Applying Authorization at the Endpoint Level

With user management in place (section 4.1), now look at the authentication method and endpoint configuration. Authorization is covered in chapters 7–12; here only the big picture.

- By default all endpoints require a valid user managed by the app.
- By default the app uses HTTP Basic, but this can be overridden.
- HTTP Basic doesn't fit most application architectures, and not all endpoints need securing; those that do may need different authentication methods and authorization rules.
- To customize authentication and authorization, define a bean of type **`SecurityFilterChain`**.

**Project:** `ssia-ch2-ex3`.

```java
@Configuration
public class ProjectConfig {

 @Bean
 SecurityFilterChain configure(HttpSecurity http)
 throws Exception {
 return http.build();
 }

 // Omitted code
}
```
*Listing 2.6 — Defining a `SecurityFilterChain` bean*

Alter the configuration using methods of the `HttpSecurity` object:

```java
@Configuration
public class ProjectConfig {

 @Bean
 SecurityFilterChain configure(HttpSecurity http)
 throws Exception {
 http.httpBasic(Customizer.withDefaults()); // ← app uses HTTP Basic authentication
 http.authorizeHttpRequests(
 c -> c.anyRequest().authenticated() // ← all requests require authentication
 );
 return http.build();
 }

 // Omitted code
}
```
*Listing 2.7 — Using the `HttpSecurity` parameter to alter the configuration*

This behaves the same as the default. A slight change makes all endpoints accessible without credentials:

```java
@Configuration
public class ProjectConfig {

 @Bean
 public SecurityFilterChain configure(HttpSecurity http)
 throws Exception {
 http.httpBasic(Customizer.withDefaults());
 http.authorizeHttpRequests(
 c -> c.anyRequest().permitAll() // ← no request needs authentication
 );
 return http.build();
 }

 // Omitted code
}
```
*Listing 2.8 — Using `permitAll()` to change the authorization configuration*

```bash
curl http://localhost:8080/hello
```
```
Hello!
```

`permitAll()` together with `anyRequest()` makes all endpoints accessible without credentials.

**Two configuration methods used:**

| Method | Purpose |
|---|---|
| `httpBasic()` | Configures the **authentication** approach; tells the app to accept HTTP Basic |
| `authorizeHttpRequests()` | Configures **authorization rules at the endpoint level**; tells the app how to authorize requests on specific endpoints |

Both take a **`Customizer`** parameter.

**`Customizer`:**
- A contract you implement to define customization for a Spring Security element: authentication, authorization, or protection mechanisms such as **CSRF or CORS** (chapters 9 and 10).
- It's a **functional interface**, so lambdas can implement it.
- `withDefaults()` is just a `Customizer` implementation that **does nothing**.

```java
@FunctionalInterface
public interface Customizer<T> {
 void customize(T t);

 static <T> Customizer<T> withDefaults() {
 return (t) -> {
 };
 }
}
```

**Older chaining style (left behind):** configuration was applied without a `Customizer`, following the method call:
```java
http.authorizeHttpRequests()
 .anyRequest().authenticated()
```

**Why `Customizer` replaced it:**
- More flexibility to **move configuration where needed**.
- Lambdas are comfortable in simple examples, but real-world configs can grow a lot. Moving them into **separate classes** keeps them easier to **maintain and test**.

The purpose here is a feel for overriding defaults; authorization detail comes in chapters 7–10.

> **NOTE** — In earlier Spring Security versions, a security configuration class had to extend **`WebSecurityConfigurerAdapter`**. ❌ This practice is no longer used. For older codebases or upgrades, see the first edition of *Spring Security in Action*.

### 4.3 Configuring in Different Ways

Spring Security often offers **multiple ways to configure the same thing**. Know the options to recognize them in the book, blogs, and articles, and to know when to use them. Later chapters extend this.

Starting from the first project, we overrode `UserDetailsService` and `PasswordEncoder` by adding them as **beans in the Spring context**. The alternative: set both via the **`SecurityFilterChain` bean**.

**Project:** `ssia-ch2-ex3`.

```java
@Configuration
public class ProjectConfig {

 @Bean
 public SecurityFilterChain configure(HttpSecurity http)
 throws Exception {
 http.httpBasic(Customizer.withDefaults());
 http.authorizeHttpRequests(
 c -> c.anyRequest().authenticated()
 );

 var user = User.withUsername("john") // ← defines a user with all its details
 .password("12345")
 .authorities("read")
 .build();

 var userDetailsService =
 new InMemoryUserDetailsManager(user); // ← stores users in memory, adds the user

 http.userDetailsService(userDetailsService); // ← UserDetailsService now set via SecurityFilterChain

 return http.build();
 }

 // Omitted code
}
```
*Listing 2.9 — Setting `UserDetailsService` with the `SecurityFilterChain` bean*

- The `UserDetailsService` is declared like in listing 2.5, but **locally** inside the `SecurityFilterChain` bean method.
- `HttpSecurity.userDetailsService()` registers the instance.

```java
@Configuration
public class ProjectConfig {

 @Bean
 SecurityFilterChain configure(HttpSecurity http)
 throws Exception {
 http.httpBasic(Customizer.withDefaults());
 http.authorizeHttpRequests(
 c -> c.anyRequest().authenticated()
 );

 var user = User.withUsername("john") // ← creates a new user
 .password("12345")
 .authorities("read")
 .build();

 var userDetailsService =
 new InMemoryUserDetailsManager(user); // ← adds the user to be managed by our UserDetailsService

 http.userDetailsService(userDetailsService); // ← configures UserDetailsService

 return http.build();
 }

 @Bean
 PasswordEncoder passwordEncoder() {
 return NoOpPasswordEncoder.getInstance();
 }
}
```
*Listing 2.10 — Full definition of the configuration class*

**Which to choose?** Both are correct.

| Option | When it fits |
|---|---|
| ✅ Beans in the context | Lets you **inject** the values into other classes that might need them |
| ✅ Set inside `SecurityFilterChain` | Equally good if you don't need that injection |

⚠️ Pick one and stay consistent (see section 4).

### 4.4 Defining Custom Authentication Logic

Spring Security components are flexible. You've seen the purpose of `UserDetailsService` and `PasswordEncoder` and a few ways to configure them. Now customize the component that **delegates to these**, the `AuthenticationProvider`.

![](figure_2_3.png)
*(AuthenticationProvider implements authentication logic; delegates user lookup to UserDetailsService and password verification to PasswordEncoder)*

- `AuthenticationProvider` implements the authentication logic and delegates to `UserDetailsService` and `PasswordEncoder`.
- This goes one step deeper into the authentication architecture. Only a brief picture is given here; chapters 3–6 go into detail.

**Design advice:**
- Spring Security's architecture is **loosely coupled with fine-grained responsibilities**, which makes it flexible and easy to integrate.
- You could change the design (e.g., override the default `AuthenticationProvider` so you no longer need a `UserDetailsService` or `PasswordEncoder`), but ⚠️ be careful, as this can **complicate your solution**.

**Project:** `ssia-ch2-ex4`.

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

 @Override
 public Authentication authenticate(Authentication authentication)
 throws AuthenticationException {

 // authentication logic here
 }

 @Override
 public boolean supports(Class<?> authenticationType) {
 // type of the Authentication implementation here
 }
}
```
*Listing 2.11 — Implementing the `AuthenticationProvider` interface*

- `authenticate(Authentication authentication)` holds **all the authentication logic**.
- `supports()` is explained in **chapter 6**; for now, take its implementation for granted (not essential here).

```java
@Override
public Authentication authenticate(
 Authentication authentication)
 throws AuthenticationException {

 String username = authentication.getName(); // ← getName() is inherited by Authentication from the Principal interface
 String password = String.valueOf(
 authentication.getCredentials());

 if ("john".equals(username) && // ← generally calls UserDetailsService and PasswordEncoder to test username/password
 "12345".equals(password)) {
 return new UsernamePasswordAuthenticationToken(
 username,
 password,
 Arrays.asList());
 } else {
 throw new AuthenticationCredentialsNotFoundException("Error!");
 }
}
```
*Listing 2.12 — Implementing the authentication logic*

- The `if-else` condition **replaces the responsibilities** of `UserDetailsService` and `PasswordEncoder`.
- ⚠️ You aren't required to use the two beans, but if you work with users and passwords, **strongly keep their management logic separate**, as the Spring Security architecture designed it, even when overriding the authentication implementation.
- Replacing the authentication logic is useful when the default implementation doesn't entirely fit your requirements.

```java
@Component
public class CustomAuthenticationProvider
 implements AuthenticationProvider {

 @Override
 public Authentication authenticate(
 Authentication authentication)
 throws AuthenticationException {
 String username = authentication.getName();
 String password = String.valueOf(authentication.getCredentials());

 if ("john".equals(username) &&
 "12345".equals(password)) {
 return new UsernamePasswordAuthenticationToken(
 username, password, Arrays.asList());
 } else {
 throw new AuthenticationCredentialsNotFoundException("Error!");
 }
 }

 @Override
 public boolean supports(Class<?> authenticationType) {
 return UsernamePasswordAuthenticationToken
 .class
 .isAssignableFrom(authenticationType);
 }
}
```
*Listing 2.13 — The full implementation of the authentication provider*

**Register it** in the configuration class with `HttpSecurity.authenticationProvider()`:

```java
@Configuration
public class ProjectConfig {

 private final CustomAuthenticationProvider authenticationProvider;

 public ProjectConfig(
 CustomAuthenticationProvider authenticationProvider) {
 this.authenticationProvider = authenticationProvider;
 }

 @Bean
 SecurityFilterChain configure(HttpSecurity http) throws Exception {
 http.httpBasic(Customizer.withDefaults());

 http.authenticationProvider(authenticationProvider); // ← registers the custom provider
 http.authorizeHttpRequests(
 c -> c.anyRequest().authenticated()
 );
 return http.build();
 }
}
```
*Listing 2.14 — Registering the new implementation of `AuthenticationProvider`*

The only recognized user is `john` / `12345`:
```bash
curl -u john:12345 http://localhost:8080/hello
```
```
Hello!
```

Chapter 6 covers `AuthenticationProvider` in more detail, plus the `Authentication` interface and its implementations such as `UserPasswordAuthenticationToken`.

> **NOTE** — The book writes `UserPasswordAuthenticationToken` here, while the code in listings 2.12–2.13 uses `UsernamePasswordAuthenticationToken`. Kept as in the book; likely a typo in the prose.

### 4.5 Using Multiple Configuration Classes

Previous examples used one configuration class. Good practice is **one class per responsibility**, even for configuration classes. This matters as config grows: production apps have more declarations, and several classes make the project more readable.

Here, user management is separated from authorization configuration into `UserManagementConfig` (listing 2.15) and `WebAuthorizationConfig` (listing 2.16).

**Project:** `ssia-ch2-ex5`.

```java
@Configuration
public class UserManagementConfig {

 @Bean
 public UserDetailsService userDetailsService() {
 var userDetailsService = new InMemoryUserDetailsManager();

 var user = User.withUsername("john")
 .password("12345")
 .authorities("read")
 .build();

 userDetailsService.createUser(user);
 return userDetailsService;
 }

 @Bean
 public PasswordEncoder passwordEncoder() {
 return NoOpPasswordEncoder.getInstance();
 }
}
```
*Listing 2.15 — Configuration class for user and password management*

`UserManagementConfig` holds only the two user-management beans: `UserDetailsService` and `PasswordEncoder`.

```java
@Configuration
public class WebAuthorizationConfig {

 @Bean
 SecurityFilterChain configure(HttpSecurity http)
 throws Exception {
 http.httpBasic(Customizer.withDefaults());
 http.authorizeHttpRequests(
 c -> c.anyRequest().authenticated()
 );
 return http.build();
 }
}
```
*Listing 2.16 — Configuration class for authorization management*

`WebAuthorizationConfig` defines the `SecurityFilterChain` bean that configures authentication and authorization rules.

---

## 5. Summary of All Approaches

| Approach | Syntax / Tool | When to use |
|---|---|---|
| Default configuration | `spring-boot-starter-security` only | Proof that dependencies are in place; ❌ not production |
| Custom `UserDetailsService` as a bean | `@Bean UserDetailsService` + `InMemoryUserDetailsManager` | When you may need to inject it elsewhere; ✅ |
| Custom `PasswordEncoder` as a bean | `@Bean PasswordEncoder` + `NoOpPasswordEncoder.getInstance()` | Required when replacing `UserDetailsService`; ❌ NoOp only for learning/PoC |
| `UserDetailsService` via filter chain | `http.userDetailsService(...)` | When you don't need to inject it elsewhere; ✅ equally good |
| HTTP Basic authentication | `http.httpBasic(Customizer.withDefaults())` | Configure the authentication approach |
| Require authentication on all endpoints | `c.anyRequest().authenticated()` | Default behavior |
| Open all endpoints | `c.anyRequest().permitAll()` | No credentials needed for any endpoint |
| Custom authentication logic | `AuthenticationProvider` + `http.authenticationProvider(...)` | Default implementation doesn't fit your requirements |
| Multiple configuration classes | `UserManagementConfig` + `WebAuthorizationConfig` | Separate responsibilities; ✅ recommended as config grows |

⚠️ In a single application, **choose one configuration style and stick to it**.

---

## Quick-Reference Summary

| Concept | One-liner |
|---|---|
| Spring Boot defaults | Provided when you add Spring Security to the dependencies (convention over configuration) |
| HTTP Basic | Authentication via the `Authorization` header: `Basic` + Base64(`username:password`); needs HTTPS for confidentiality |
| Base64 | Only an encoding for transfer, **not** encryption or hashing |
| `UserDetailsService` | Contract for user management; default stores one `user` with a random UUID password in memory |
| `InMemoryUserDetailsManager` | Spring Security's simple `UserDetailsService` that keeps users in application memory (❌ not for production) |
| `User` | Builder to define users; needs at least a username, a password, and an authority |
| Authority | An action a user is allowed to do in the application context; any string |
| `PasswordEncoder` | Encodes passwords and verifies whether a password matches an existing encoding; mandatory for the Basic flow |
| `NoOpPasswordEncoder` | `PasswordEncoder` using cleartext passwords; deprecated; good for learning and (maybe) PoCs only |
| `AuthenticationProvider` | Contract that implements authentication logic and delegates user/password management |
| `SecurityFilterChain` | Bean used to customize authentication and authorization via `HttpSecurity` |
| `HttpSecurity.httpBasic()` | Configures the authentication approach to HTTP Basic |
| `HttpSecurity.authorizeHttpRequests()` | Configures endpoint-level authorization rules |
| `Customizer` | Functional interface to define customization of a Spring Security element; `withDefaults()` does nothing |
| `permitAll()` / `authenticated()` | Allow all requests without credentials / require authentication |
| Security context | Stores authentication data until the request ends |
| `@Configuration` / `@Bean` | Mark a configuration class / add the returned object to the Spring context |
| HTTP 401 vs 403 | 401 = failed authentication (missing/incorrect credentials); 403 = identified caller without privileges |
| Configuration styles | Multiple ways exist; choose one and stay consistent for cleaner, more understandable code |

---
