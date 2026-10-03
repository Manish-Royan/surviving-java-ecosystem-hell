# 🌿 Phase 4: Bean Destruction

## 0. Where We Are

```text
Phase 1 → Spring reads the blueprint (BeanDefinition)
Phase 2 → Spring creates the raw object (constructor runs)
Phase 3 → Spring wires it, initializes it, makes it ready
Phase 4 → Spring cleans it up when the container shuts down   ← we are here (last phase)
```

Phases 1 to 3 build a bean. Phase 4 is the goodbye: the container is closing, and each bean gets one last chance to **release what it was holding**.

---

## 1. The Big Picture 🖼️

### What "destruction" really means

Spring does **not** delete objects from memory. The garbage collector does that.

Destruction means Spring **calls your cleanup methods** so the bean can release *external* resources:

- close database connections
- close files and sockets
- stop background threads
- flush caches or buffers to disk

```text
Container closing
      │
      ▼
For each singleton bean (that has cleanup logic):
      │
      ├─ 1. @PreDestroy method
      ├─ 2. DisposableBean.destroy()
      └─ 3. custom destroy-method
      │
      ▼
Container closed → beans become garbage → GC frees memory
```

### Phase 3 vs Phase 4 — they mirror each other

| Phase 3 (start-up) | Phase 4 (shutdown) |
|---|---|
| `@PostConstruct` | `@PreDestroy` |
| `InitializingBean.afterPropertiesSet()` | `DisposableBean.destroy()` |
| `init-method` | `destroy-method` |

Same three styles, same order, just on the way out.

---

## 2. The Trigger — Destruction Only Runs If the Container Shuts Down Properly

Nothing happens to beans on its own. Destruction starts **only when the container is closed**.

### 2.1 Three ways to close it

```java
// 1. Close manually
context.close();

// 2. Register a JVM shutdown hook (closes automatically when the program exits)
context.registerShutdownHook();

// 3. try-with-resources (the context is Closeable)
try (AnnotationConfigApplicationContext context =
         new AnnotationConfigApplicationContext(AppConfig.class)) {
    // use beans...
}   // close() is called automatically here
```

### 2.2 🚨 The #1 gotcha

In a plain Java app (a `main()` method), if you do **none** of the above, `main()` just ends, the JVM exits, and **`@PreDestroy` never runs**. No error, no warning. The cleanup code is silently skipped.

| Situation | Who closes the container? |
|---|---|
| Plain `main()` app | **You** (`close()` or `registerShutdownHook()`) |
| Spring Boot app | Spring Boot registers the shutdown hook for you |
| Web app in a servlet container | The container closes the context when the app stops |

### 2.3 When the shutdown hook still won't run

The hook runs on normal exit, `Ctrl+C` (SIGINT) and `kill` (SIGTERM). It does **not** run on `kill -9`, a JVM crash, power loss or `System.halt()`. So never rely on `@PreDestroy` as your only protection for critical data.

---

## 3. The Three Destruction Callbacks

### 3.1 `@PreDestroy`

```java
@PreDestroy
public void cleanup() {
    System.out.println("Closing connection...");
}
```

- Comes from `jakarta.annotation` (Spring 6+; `javax.annotation` in older versions). It is not a Spring annotation, and needs the same `jakarta.annotation-api` dependency as `@PostConstruct`.
- Run by `CommonAnnotationBeanPostProcessor`, the same helper that runs `@PostConstruct`.

### 3.2 `DisposableBean.destroy()`

```java
@Component
public class UserService implements DisposableBean {
    @Override
    public void destroy() {
        System.out.println("destroy() called");
    }
}
```

Works, but it ties your class to a Spring interface. Mostly used by framework code.

### 3.3 Custom `destroy-method`

Declared outside the class, so the class needs no Spring annotation or interface:

```java
@Bean(destroyMethod = "shutdown")
public MyPool myPool() {
    return new MyPool();
}
```

### 3.4 Order (memorize this)

If a bean uses all three, they run in this order:

```text
@PreDestroy  →  DisposableBean.destroy()  →  custom destroy-method
```

| Option | Coupling | Verdict |
|---|---|---|
| `@PreDestroy` | Standard annotation, minimal | ✅ Preferred for your own classes |
| `DisposableBean` | Ties the class to Spring | ⚠️ Mostly legacy / framework code |
| `destroy-method` | None (declared outside) | ✅ Best for third-party classes you can't modify |

### 3.5 Bonus: Spring guesses `close()` / `shutdown()` for `@Bean` methods

For beans created with `@Bean`, Spring automatically looks for a public no-argument method named `close()` or `shutdown()` and calls it on shutdown. This is why connection pools (`DataSource`) get closed without extra code.

To switch this off, use `@Bean(destroyMethod = "")`.

---

## 4. Order Between Different Beans

If `UserService` uses `UserRepository`, which one should close first?

Obviously the **service first**, then the repository. Otherwise the service could call a repository that is already closed.

Spring follows this rule:

```text
Created:    UserRepository → UserService
Destroyed:  UserService → UserRepository      (reverse order)
```

Two things guarantee it:

1. Singletons are destroyed in the **reverse order of creation**.
2. Before a bean is destroyed, Spring first destroys every bean that **depends on it**.

> 💡 This is also why dependencies are created first during start-up (Phase 3, section 3.3): the same dependency information is reused in reverse on shutdown.

---

## 5. Internals — How Spring Does It (Simple Version)

### 5.1 Spring decides *at creation time*

At the very end of a singleton's creation (right after Phase 3 finishes), Spring asks:

> "Does this bean have any cleanup logic?"

It has cleanup logic if it:
- has a `@PreDestroy` method, or
- implements `DisposableBean` or `AutoCloseable`, or
- has a `destroy-method` (explicit, or `close()`/`shutdown()` found for `@Bean`)

If **yes**, Spring wraps it in a small helper object, a **`DisposableBeanAdapter`**, and stores it in a list inside `DefaultSingletonBeanRegistry`. Beans with no cleanup logic are not tracked at all.

### 5.2 What happens on `context.close()`

```text
context.close()
   │
   ├─ 1. Publish ContextClosedEvent        (your @EventListener can react here, beans still alive)
   ├─ 2. Stop Lifecycle beans              (e.g. SmartLifecycle)
   ├─ 3. Destroy singletons                ← the part this note is about
   │       for each tracked bean, in reverse order:
   │         a. destroy the beans that depend on it first
   │         b. call its DisposableBeanAdapter.destroy():
   │              ① @PreDestroy         (via DestructionAwareBeanPostProcessor)
   │              ② DisposableBean.destroy()
   │              ③ custom destroy-method
   │         c. remove it from the container's caches
   ├─ 4. Close the bean factory
   └─ 5. Context is now inactive           (getBean() will throw an exception)
```

### 5.3 Why `@PreDestroy` is first

Just like `@PostConstruct`, `@PreDestroy` is not handled by Spring's core. A special kind of post-processor, `DestructionAwareBeanPostProcessor`, has a hook called `postProcessBeforeDestruction()`. `CommonAnnotationBeanPostProcessor` uses it to invoke your `@PreDestroy` method. This hook runs before the interface and the custom method.

### 5.4 What if a destroy method throws an exception?

Spring **logs it and carries on** with the remaining beans. One broken cleanup does not stop the others. It is still better to catch exceptions yourself inside cleanup code.

---

## 6. Destruction and Scopes

| Scope | Does Spring call destroy callbacks? |
|---|---|
| **Singleton** (default) | ✅ Yes, when the container closes |
| **Prototype** | ❌ **Never.** Spring creates it, hands it over and forgets it |
| Request / Session (web) | ✅ Yes, when the request or session ends |

### 🚨 The Prototype Trap

```java
@Component
@Scope("prototype")
public class ReportGenerator {
    @PreDestroy
    public void cleanup() {
        System.out.println("This will NEVER print");
    }
}
```

Spring does not keep prototype instances, so it has nothing to destroy later. If a prototype holds a resource (file, connection), **the code that asked for it must clean it up itself**, for example:

```java
ReportGenerator report = context.getBean(ReportGenerator.class);
try {
    // use it...
} finally {
    report.cleanup();   // you call it manually
}
```

---

## 7. 📌 Full Demo — See Destruction in Action

A plain Spring project (no Spring Boot), using the same dependencies as the Phase 3 demo (`spring-context` + `jakarta.annotation-api`).

All files go in the package `com.example.destruction`. Use a fresh package so it doesn't mix with the Phase 3 classes.

**UserRepository.java**

```java
package com.example.destruction;

import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
public class UserRepository {

    public UserRepository() {
        System.out.println("UserRepository created");
    }

    @PreDestroy
    public void closeConnection() {
        System.out.println("[destroy] UserRepository -> @PreDestroy (closing connection)");
    }
}
```

**UserService.java**

```java
package com.example.destruction;

import jakarta.annotation.PreDestroy;
import org.springframework.beans.factory.DisposableBean;

// No @Component here: registered in AppConfig with @Bean so we can use destroyMethod
public class UserService implements DisposableBean {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
        System.out.println("UserService created (repository injected via constructor)");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("[destroy] UserService    -> 1. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("[destroy] UserService    -> 2. DisposableBean.destroy()");
    }

    public void customDestroy() {
        System.out.println("[destroy] UserService    -> 3. custom destroy-method");
    }
}
```

**ReportGenerator.java** (prototype)

```java
package com.example.destruction;

import jakarta.annotation.PreDestroy;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope("prototype")
public class ReportGenerator {

    public ReportGenerator() {
        System.out.println("ReportGenerator created");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("[destroy] ReportGenerator -> @PreDestroy (you will never see this)");
    }
}
```

**AppConfig.java**

```java
package com.example.destruction;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.example.destruction")
public class AppConfig {

    @Bean(destroyMethod = "customDestroy")
    public UserService userService(UserRepository userRepository) {
        return new UserService(userRepository);
    }
}
```

**Main.java**

```java
package com.example.destruction;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {

    public static void main(String[] args) {
        System.out.println("=== Starting container ===");

        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(AppConfig.class);

        // Without this line (or context.close()), NO destroy callback will run
        context.registerShutdownHook();

        System.out.println("=== Container ready ===");

        ReportGenerator r1 = context.getBean(ReportGenerator.class);
        ReportGenerator r2 = context.getBean(ReportGenerator.class);
        System.out.println("Same prototype instance? " + (r1 == r2));

        System.out.println("=== main() finished, JVM is exiting ===");
    }
}
```

**Output:**

```text
=== Starting container ===
UserRepository created
UserService created (repository injected via constructor)
=== Container ready ===
ReportGenerator created
ReportGenerator created
Same prototype instance? false
=== main() finished, JVM is exiting ===
[destroy] UserService    -> 1. @PreDestroy
[destroy] UserService    -> 2. DisposableBean.destroy()
[destroy] UserService    -> 3. custom destroy-method
[destroy] UserRepository -> @PreDestroy (closing connection)
```

**How to read it:**

| What you see | Why |
|---|---|
| `[destroy]` lines appear **after** `main() finished` | The shutdown hook runs when the JVM exits |
| `UserService` closes before `UserRepository` | Reverse order, and the service depends on the repository |
| `UserService` shows 1 → 2 → 3 | `@PreDestroy` → `DisposableBean` → `destroy-method` |
| `ReportGenerator` never appears in the destroy lines | Prototype beans are not destroyed by the container |
| `ReportGenerator created` printed twice | Each `getBean()` makes a new prototype |

**Try this experiment:** delete the `registerShutdownHook()` line and run again. All `[destroy]` lines disappear. That is the #1 gotcha in action.

---

## 8. The Complete Bean Lifecycle — Creation to Destruction

For a normal singleton bean:

```text
PHASE 1 — Blueprint
   Bean definitions are found and loaded (BeanDefinition)

PHASE 2 — Creation
   Constructor runs → raw object exists

PHASE 3 — Wiring & Initialization
   @Autowired / @Value injection
   → Aware callbacks
   → BeanPostProcessor.before
   → @PostConstruct
   → afterPropertiesSet()
   → init-method
   → BeanPostProcessor.after (AOP proxy may replace the bean)
   → stored in singletonObjects  ✔ READY

        ... the bean serves the application ...

PHASE 4 — Destruction (container shuts down)
   @PreDestroy
   → DisposableBean.destroy()
   → destroy-method
   → bean becomes garbage → GC frees the memory
```

Prototype beans are different: they go through Phases 2 and 3 on every `getBean()`, and **never** reach Phase 4.

---

## 9. Common Gotchas

1. `@PreDestroy` does not run in a plain `main()` app unless you call `close()` or `registerShutdownHook()`.
2. It also won't run on `kill -9`, a crash or `System.halt()`.
3. Prototype beans never get destruction callbacks, so clean them up yourself.
4. `@PreDestroy` needs the `jakarta.annotation-api` dependency, just like `@PostConstruct`.
5. For `@Bean` beans, `close()` / `shutdown()` are called automatically. Use `destroyMethod = ""` if you don't want that.
6. Destruction order is the reverse of creation, and dependents are destroyed first.
7. An exception in one destroy method is logged, and Spring continues with the others.
8. After the context is closed, `getBean()` throws an exception.

---

## 10. 📋 End-of-Phase Checklist

**✔ After Phase 4 you should know:**
- ✔ Destruction means running cleanup callbacks, not freeing memory
- ✔ It starts only when the container closes (`close()`, shutdown hook, or try-with-resources)
- ✔ Callback order: `@PreDestroy` → `DisposableBean.destroy()` → `destroy-method`
- ✔ Beans are destroyed in reverse order, dependents first
- ✔ Spring tracks cleanup-capable singletons with a `DisposableBeanAdapter`
- ✔ Prototype beans are never destroyed by the container

---

## 11. 🧾 Summary Card

```text
Phase 4 — Bean Destruction
──────────────────────────────────────────────────────────
WHEN   : container closes (close() / shutdown hook / try-with-resources)
WHAT   : @PreDestroy → DisposableBean.destroy() → destroy-method
ORDER  : reverse of creation; beans that depend on it go first
HOW    : bean registered as "disposable" at the end of creation
         → DisposableBeanAdapter runs the 3 callbacks on shutdown
         (@PreDestroy handled by a DestructionAwareBeanPostProcessor)
GOTCHA : no close()/hook = no cleanup; prototype beans never destroyed;
         kill -9 skips everything
MIRROR : @PostConstruct ↔ @PreDestroy,
         afterPropertiesSet ↔ destroy,
         init-method ↔ destroy-method
```

> **One sentence to remember:** Spring built the bean through post-processors, and it tears the bean down through the same post-processors. The only catch is that the container has to be shut down properly for the cleanup to run.

---

## 12. 📝 Series Recap — Quick Order Cheat-Sheet

```text
Constructor
  → Dependency injection (@Autowired / @Value)
  → Aware callbacks
  → BeanPostProcessor.before
  → @PostConstruct
  → afterPropertiesSet()
  → init-method
  → BeanPostProcessor.after (proxy)
  → READY (runtime)
  → @PreDestroy
  → DisposableBean.destroy()
  → destroy-method
```
---