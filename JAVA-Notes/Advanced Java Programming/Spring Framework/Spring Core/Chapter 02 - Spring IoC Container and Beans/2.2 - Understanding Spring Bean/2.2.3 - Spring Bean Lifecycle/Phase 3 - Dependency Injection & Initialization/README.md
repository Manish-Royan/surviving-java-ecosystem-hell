# 🌿 Phase 3: Dependency Injection, Initialization & Ready State

## 0. Where We Are

```text
Phase 1 → Spring reads the blueprint (BeanDefinition)
Phase 2 → Spring creates the raw object (constructor runs)
Phase 3 → Spring wires it, initializes it, makes it ready   ← we are here
Phase 4 → Spring destroys it (container shutdown)
```

Inside Spring, bean creation is done by `doCreateBean()`, which calls three methods in order:

```text
doCreateBean()
   ├─ createBeanInstance()   → Phase 2: raw object is created
   ├─ populateBean()         → Phase 3 (a): dependency injection
   └─ initializeBean()       → Phase 3 (b): initialization
```

- `populateBean()` = fill the object with its dependencies
- `initializeBean()` = run the setup callbacks, then give processors a last chance to wrap the bean

---

## 1. The Big Picture 🖼️

Think of it like a house:

- **Phase 2** built an empty house (object exists, dependencies are still `null`)
- **Phase 3** connects water and electricity (dependency injection), furnishes it (initialization), and hands over the keys (ready bean)

```text
Constructor finished (Phase 2)  — object exists, dependencies not injected yet
        │
        ▼
[1] populateBean()                         → @Autowired / @Value injection
        │
        ▼
[2] Aware callbacks                        → bean learns its name / container
        │
        ▼
[3] BeanPostProcessor.postProcessBeforeInitialization()
        │                                  → @PostConstruct runs here
        ▼
[4] afterPropertiesSet()                   → InitializingBean interface
        │
        ▼
[5] custom init-method                     → @Bean(initMethod = "...")
        │
        ▼
[6] BeanPostProcessor.postProcessAfterInitialization()
        │                                  → AOP proxy may be created here
        ▼
[7] Stored in the container (singleton)    ✔ BEAN IS READY
```

Steps **[2] to [6]** together are what `initializeBean()` does.

**What is true at each moment:**

| Moment | Dependencies injected? | Init callbacks done? | Proxy created? |
|---|---|---|---|
| After constructor (Phase 2) | ❌ (setter/field) | ❌ | ❌ |
| After `populateBean()` | ✅ | ❌ | ❌ |
| After `initializeBean()` | ✅ | ✅ | ✅ (only if the bean needs one) |

---

## 2. 🎏 Two Ways Dependencies Enter a Bean

Before looking at `populateBean()`, understand that dependencies can arrive at two different times:

| | Constructor injection | Setter / field injection |
|---|---|---|
| When | **During instantiation** (Phase 2) | **After instantiation** (Phase 3, `populateBean()`) |
| Inside the constructor | Dependencies are already available | Dependencies are still `null` |
| `final` fields | ✅ Possible | ❌ Not possible |
| Object state | Fully valid right after the constructor | Temporarily incomplete |
| Circular dependency | ❌ Fails | ✅ Can be resolved |

```java
// Constructor injection → Spring must find UserRepository BEFORE it can call this constructor
public UserService(UserRepository userRepository) { ... }

// Setter injection → object is created first, then Spring calls the setter
@Autowired
public void setUserRepository(UserRepository userRepository) { ... }
```

> So don't think "Spring always creates the object first and injects later." That is only true for setter/field injection. This is also why constructor injection is the modern recommendation.

This phase's injection step (`populateBean()`) handles **setter and field injection**. Constructor injection was already handled in Phase 2.

---

## 3. Step 1 — 💉 Dependency Injection (`populateBean()`)

### 3.1 What happens

```text
Before:  UserService ── userRepository = null

After:   UserService ── userRepository ──► UserRepository object
```

Spring does not let `UserService` create its own repository (`new UserRepository()`). Instead, the container supplies an already-managed one. That is **Inversion of Control (IoC)**.

### 3.2 `@Autowired` is not magic

`@Autowired` is only a marker (metadata). Java does nothing with it. A Spring component called **`AutowiredAnnotationBeanPostProcessor`** finds the marker and does the injection using reflection:

```java
// Concept only (simplified):
Field field = bean.getClass().getDeclaredField("userRepository");
field.setAccessible(true);                 // allow access to a private field
field.set(bean, userRepositoryBean);       // inject the dependency
```

- For a **setter**, it calls the method (`method.invoke(...)`) instead.
- If no such processor is registered in the container, `@Autowired` fields silently stay `null`.

This is the key idea of the whole lifecycle: **Spring's core only orchestrates; small helper classes called `BeanPostProcessor`s do the actual work** (injection, `@PostConstruct`, proxies...). Otherwise Spring's core would need hard-coded `if @Autowired ... if @PostConstruct ... if @Transactional ...` logic.

### 3.3 Missing dependencies are created on demand

For each dependency, Spring asks the container for it (`getBean()`). If it doesn't exist yet, Spring creates it first, running its **full lifecycle** (constructor → injection → initialization).

> That's why constructor logs of dependencies appear **before** your own bean's callbacks.

### 3.4 `@Value`

`@Value("${app.timeout}")` is handled by the same processor. The `${...}` placeholder is replaced with the real value first, then injected.

### 3.5 Circular dependencies

Bean A needs Bean B, and Bean B needs Bean A.

- **Setter / field injection** → works. Spring hands out an *early reference* to the half-built bean (it keeps it in an internal 3-level cache).
- **Constructor injection** → ❌ `BeanCurrentlyInCreationException`, because neither object can be created without the other.
- **Fix:** redesign so the cycle disappears, or put `@Lazy` on one constructor parameter.
- Spring Boot 2.6+ rejects circular references by default (not relevant for pure Spring, but good to know).

---

## 4. Step 2 — 🚦Initialization (`initializeBean()`)

Now the bean is fully wired. Spring runs the setup callbacks **in a fixed order**.

### 4.1 Aware callbacks — "the bean learns where it lives"

| Interface | What the bean receives |
|---|---|
| `BeanNameAware` | its own bean name |
| `BeanClassLoaderAware` | the class loader |
| `BeanFactoryAware` | the `BeanFactory` |
| `ApplicationContextAware`, `EnvironmentAware`, ... | the context / environment |

Two small details:

- The first three are called **directly** by `initializeBean()`.
- The context-style ones (`ApplicationContextAware`, `EnvironmentAware`, ...) are delivered by a post-processor (`ApplicationContextAwareProcessor`) a moment later, in the "before initialization" step.

> 💡 Avoid Aware interfaces in normal business code — they tie your class to Spring. Use them only for framework-level utilities.

### 4.2 `postProcessBeforeInitialization()` → where `@PostConstruct` runs

Every registered `BeanPostProcessor` gets a "before initialization" call. During this step, Spring's `CommonAnnotationBeanPostProcessor` invokes your `@PostConstruct` method.

```java
@PostConstruct
public void init() {
    // Safe: all dependencies are already injected here
}
```

> 🚨 `@PostConstruct` is **not** a Spring annotation. It belongs to `jakarta.annotation` (Spring 6+; `javax.annotation` in older versions) and was removed from the JDK in Java 11, so you must add the `jakarta.annotation-api` dependency. Spring just ships a processor that knows how to read it.

### 4.3 `InitializingBean.afterPropertiesSet()`

The method name literally means "after all properties are set". It's a good place for validation:

```java
@Override
public void afterPropertiesSet() {
    Assert.notNull(userRepository, "userRepository must be injected!");
}
```

### 4.4 Custom `init-method`

Declared on the bean definition, so the class itself needs no Spring annotation or interface:

```java
@Bean(initMethod = "customInit")
public UserService userService() {
    return new UserService();
}
```

### 4.5 The order (memorize this)

If a bean uses all three, they always run like this:

```text
@PostConstruct  →  afterPropertiesSet()  →  custom init-method
```

Why three ways? History: Spring 1.x had the interface + XML `init-method`; the standard `@PostConstruct` came later.

| Option | Coupling | Verdict |
|---|---|---|
| `@PostConstruct` | Standard annotation, minimal | ✅ Preferred for your own classes |
| `afterPropertiesSet()` | Ties the class to a Spring interface | ⚠️ Mostly legacy / framework code |
| `init-method` | None (declared outside the class) | ✅ Best for third-party classes you can't modify |

---

## 5. Step 3 — `postProcessAfterInitialization()` → 🌐 the AOP Proxy 

This is the **last hook** before the bean is ready. Each processor may return the same bean, or **a different object (a proxy) in its place**.

Spring's AOP infrastructure (`AbstractAutoProxyCreator`) checks:

> "Does this bean match any `@Transactional` / `@Async` / `@Cacheable` / aspect rule?"

- **No** → the bean is returned as it is.
- **Yes** → Spring returns a **proxy** wrapping your bean. (In plain Spring: a JDK proxy if the bean implements an interface, otherwise a CGLIB subclass.)

Why last? Init callbacks run **once on the real, finished object**, and then the proxy wraps it.

```text
Container stores →  Proxy  ──wraps──►  your real bean (the "target")
```

> 💡 If you print `userService.getClass()` for a `@Transactional` bean, you'll see something like `UserService$$SpringCGLIB$$0` (Spring 5: `UserService$$EnhancerBySpringCGLIB$$...`), not `UserService`. **The container holds the proxy**, so everyone who injects this bean receives the proxy.

---

## 6. Step 4 — Bean Is Ready 🫛 (Runtime State)

### 6.1 Where beans live

| | Singleton (default) | Prototype |
|---|---|---|
| Stored in container? | ✅ Yes, in `singletonObjects` (a `ConcurrentHashMap` inside `DefaultSingletonBeanRegistry`) | ❌ No |
| Each `getBean()` call | Returns the **same instance** | Creates a **new instance** (constructor → injection → initialization) |
| Destruction callbacks (`@PreDestroy`...) | ✅ Run at shutdown | ❌ **Never** — Spring forgets it after handing it over |

For a singleton, the lifecycle runs **once**; after that, `getBean()` just returns the stored object.

### 6.2 What happens when a method is called

If the bean is proxied, every call goes through the proxy first:

```text
Caller ──► Proxy ──► (extra logic: transaction / cache / async) ──► Real bean
```

Conceptually, a `@Transactional` proxy does this:

```java
public void placeOrder() {
    System.out.println("Start transaction...");
    try {
        target.placeOrder();                    // call the real bean
        System.out.println("Commit...");
    } catch (Exception e) {
        System.out.println("Rollback...");
        throw e;
    }
}
```

### 6.3 🚨 The Self-Invocation Trap (classic bug)

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    public void processOrder() {
        System.out.println("Processing order...");
        this.saveToDatabase();          // ← calls the REAL bean directly, skips the proxy
    }

    @Transactional
    public void saveToDatabase() {
        System.out.println("Saving to database...");
    }
}
```

Outside code calls the **proxy**, but inside the class `this` means the **real bean**. So `saveToDatabase()` runs **without a transaction**. This affects every proxy-based feature (`@Transactional`, `@Async`, `@Cacheable`).

**Fixes:**
1. Move the method into a **separate bean** (cleanest)
2. Inject the bean into itself: `@Autowired @Lazy private OrderService self;` and call `self.saveToDatabase()`
3. `AopContext.currentProxy()` (needs `@EnableAspectJAutoProxy(exposeProxy = true)`)

### 6.4 Two more runtime gotchas

- **Singletons are shared by all threads** → keep them stateless (or thread-safe). A field like `private int orderCounter = 0;` in a singleton causes race conditions.
- **Prototype beans holding resources** (DB connections, files) must be cleaned up manually, since Spring never calls their destroy callbacks.

---

## 7. Full Demo — See the Whole Order Yourself

A plain Spring project (no Spring Boot). Needs Java 17+.

**Dependencies (Maven):**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>6.2.0</version>
    </dependency>
    <dependency>
        <groupId>jakarta.annotation</groupId>
        <artifactId>jakarta.annotation-api</artifactId>
        <version>2.1.1</version>
    </dependency>
</dependencies>
```

(Use any Spring 6.x and match the `jakarta.annotation-api` version to it.)

All files go in the package `com.example.lifecycle`.

**UserRepository.java**

```java
package com.example.lifecycle;

import org.springframework.stereotype.Component;

@Component
public class UserRepository {

    public UserRepository() {
        System.out.println("UserRepository created");
    }
}
```

**UserService.java**

```java
package com.example.lifecycle;

import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;

// No @Component here: it is registered in AppConfig with @Bean (so we can use initMethod)
public class UserService implements BeanNameAware, InitializingBean {

    private UserRepository userRepository;

    public UserService() {
        System.out.println("1. Constructor            -> userRepository = " + userRepository);
    }

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
        System.out.println("2. Dependency injected    -> setUserRepository() called");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("3. Aware callback         -> my bean name is '" + name + "'");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("5. @PostConstruct         -> userRepository is ready: " + (userRepository != null));
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("6. afterPropertiesSet()");
    }

    public void customInit() {
        System.out.println("7. custom init-method");
    }
}
```

**TracerBeanPostProcessor.java**

```java
package com.example.lifecycle;

import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
public class TracerBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (beanName.equals("userService")) {
            System.out.println("4. BPP.before");
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (beanName.equals("userService")) {
            System.out.println("8. BPP.after");
        }
        return bean;   // returning a different object here is how proxies are created
    }
}
```

**AppConfig.java**

```java
package com.example.lifecycle;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.example.lifecycle")
public class AppConfig {

    @Bean(initMethod = "customInit")
    public UserService userService() {
        return new UserService();
    }
}
```

**Main.java**

```java
package com.example.lifecycle;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {

    public static void main(String[] args) {
        System.out.println("=== Starting container ===");

        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(AppConfig.class);

        System.out.println("=== Container ready ===");

        UserService s1 = context.getBean(UserService.class);
        UserService s2 = context.getBean(UserService.class);
        System.out.println("Same instance? " + (s1 == s2));

        // context.close() and destruction callbacks → Phase 4
    }
}
```

**Output:**

```text
=== Starting container ===
UserRepository created
1. Constructor            -> userRepository = null
2. Dependency injected    -> setUserRepository() called
3. Aware callback         -> my bean name is 'userService'
4. BPP.before
5. @PostConstruct         -> userRepository is ready: true
6. afterPropertiesSet()
7. custom init-method
8. BPP.after
=== Container ready ===
Same instance? true
```

**How to read it:**

| Output line | Lifecycle step |
|---|---|
| `UserRepository created` | The dependency is created first, because `UserService` needs it |
| `1.` | Phase 2 — constructor; dependency is still `null` |
| `2.` | `populateBean()` — setter injection |
| `3.` | Aware callback |
| `4.` | `postProcessBeforeInitialization()` |
| `5.` | `@PostConstruct` — dependency is available now |
| `6.` / `7.` | `afterPropertiesSet()` → `init-method` |
| `8.` | `postProcessAfterInitialization()` (a proxy would be created here if needed) |
| `Same instance? true` | Singleton: stored once, returned every time |

> 💡 **Why does `BPP.before` print before `@PostConstruct`?** Spring's own annotation processors (the ones handling `@Autowired` and `@PostConstruct`) sit at the end of the processor list, so a custom processor's "before" hook runs ahead of them. Treat this as a detail to observe, not something to depend on in real code.

---

## 8. Quick Reference — Who Does What

| Component | Role |
|---|---|
| `AbstractAutowireCapableBeanFactory` | Contains `createBeanInstance()`, `populateBean()`, `initializeBean()` |
| `AutowiredAnnotationBeanPostProcessor` | `@Autowired` and `@Value` injection |
| `CommonAnnotationBeanPostProcessor` | `@Resource` injection, `@PostConstruct`, `@PreDestroy` |
| `ApplicationContextAwareProcessor` | `ApplicationContextAware`-style callbacks |
| `AbstractAutoProxyCreator` | Creates AOP proxies in the "after initialization" hook |
| `DefaultSingletonBeanRegistry` | Stores finished singletons (`singletonObjects`) |

---

## 9. Common Gotchas

1. `@Autowired` and `@PostConstruct` work **only because of BeanPostProcessors**; they are not Java features.
2. Dependencies are `null` inside the constructor with setter/field injection, but available in `@PostConstruct`.
3. `@PostConstruct` needs the `jakarta.annotation-api` dependency (JDK removed it in Java 11).
4. Constructor injection + circular dependency → `BeanCurrentlyInCreationException`.
5. The container stores the **proxy**, not your raw object (when AOP applies).
6. Calling a `@Transactional` / `@Async` / `@Cacheable` method via `this` bypasses the proxy.
7. Singletons must be stateless or thread-safe.
8. Prototype beans never get destruction callbacks.

---

## 10. 📋 End-of-Phase Checklist

**✔ Done by the end of Phase 3:**
- ✔ `@Autowired` / `@Value` / setter dependencies injected
- ✔ Aware callbacks executed
- ✔ `@PostConstruct` → `afterPropertiesSet()` → `init-method` executed in this order
- ✔ AOP proxy created if needed, and **the proxy is what gets stored**
- ✔ Bean placed in `singletonObjects` and ready for use

**❌ Not yet done:**
- ❌ Container shutdown
- ❌ Destruction callbacks (`@PreDestroy`, `DisposableBean`, `destroy-method`) → Phase 4

---

## 11. 🧾 Summary Card

```text
Phase 3 — DI & Initialization
──────────────────────────────────────────────────────────
WHEN   : right after the constructor (Phase 2), per bean
WHAT   : populateBean()  → @Autowired / @Value injection
         initializeBean() → Aware → BPP.before (@PostConstruct)
                          → afterPropertiesSet → init-method
                          → BPP.after (AOP proxy)
         → bean stored in singletonObjects → READY
ORDER  : constructor → injection → Aware → BPP.before → @PostConstruct
         → afterPropertiesSet → init-method → BPP.after
GOTCHA : annotation "magic" = BeanPostProcessors; the proxy replaces
         the raw bean in the container; self-invocation skips the proxy
NEXT   : Phase 4 — Destruction
```

> **One sentence to remember:** `@Autowired`, `@Value`, `@PostConstruct`, the Aware callbacks and AOP proxies are all just `BeanPostProcessor`s acting at well-defined points in a plain Java object's life — Spring's core orchestrates them, they do the work.

---

## 12. Next Up — Phase 4: Bean Destruction

- Why `@PreDestroy` doesn't run in a plain app unless you call `context.close()` or `registerShutdownHook()`
- Destruction order: `@PreDestroy` → `DisposableBean.destroy()` → custom `destroy-method`
- Why prototype beans are never destroyed by the container
- Full birth-to-death lifecycle recap