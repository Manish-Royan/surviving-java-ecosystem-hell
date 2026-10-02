# 🌱 PHASE 2: Bean Instantiation (Object Creation + Mid BeanPostProcessor)

🌱 Welcome to Phase 2. Now we move from **“Spring knows how to create the bean”** to **“Spring actually creates the Java object.”**

Now that Spring has collected all the `BeanDefinition` blueprints, it’s time to **build the actual Java objects**. This is the phase where your bean **finally becomes a real Java object** — the **constructor runs**.


### 🌿 Our starting point is the result of Phase 1:

```text
                    Spring Container
                           │
                           ▼
                 BeanDefinition Registry
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
      userService                  userRepository
      BeanDefinition               BeanDefinition
```

Remember:

> A `BeanDefinition` is only **instructions/metadata**.
> It is NOT the actual `UserService` object.

Now Spring needs to turn this:

```text
BeanDefinition
```

into this:

```text
UserService object
```

👉 That is [**instantiation**](https://github.com/Manish-Royan/surviving-java-ecosystem-hell/tree/main/JAVA-Notes/Advanced%20Java%20Programming/Spring%20Framework/Spring%20Core/Chapter%2002%20-%20Spring%20IoC%20Container%20and%20Beans/2.2%20-%20Understanding%20Spring%20Bean/2.2.3%20-%20Spring%20Bean%20Lifecycle/Phase%202%20-%20Bean%20Instantiation/Understand%20Bean%20Instantiation).

### Important distinction (most beginners confuse this):

> ‼️ **Instantiation ≠ Initialization**
>
> - **Instantiation** = creating the object (constructor call). Object exists, but it's **incomplete** (dependencies not injected, nothing initialized).
> - **Initialization** = making the object **ready to use** (DI, `@PostConstruct`, `init-method`).

> ⚠️ **Crucial Distinction:** Instantiation is **NOT** Dependency Injection. Instantiation only creates the raw, empty object. Wiring, property setting, and lifecycle callbacks happen in later phases.

### 😵‍💫 Most confusion in this phase comes from mixing up three distinct stages:

| Stage | What happens | Spring internal method | Phase |
|---|---|---|---|
| **Instantiation** | Constructor runs → raw object allocated | `createBeanInstance()` | **2** |
| **Population (DI)** | `@Autowired` / `@Value` / setters filled | `populateBean()` | 3 |
| **Initialization** | `@PostConstruct`, `afterPropertiesSet()`, `init-method` | `initializeBean()` | 3 |

💭 Think of it like:

* **Instantiation** = *🏗️ Building an empty house*
* **Population** = *🚪 installing doors/windows (dependencies)*
* **Initialization** = *🛋️ Furniture, electricity, water — making it livable*

---
## Q. When Does Phase 2 Run? (Triggers)

A bean is instantiated only **when needed**. All paths funnel into `getBean()` → `AbstractBeanFactory.doGetBean()` — the master method:

| Trigger | When instantiation happens |
|---|---|
| **Eager singleton** (default) | Container pre-creates it at startup (`finishBeanFactoryInitialization` → `preInstantiateSingletons()`) |
| **`@Lazy` bean** | On first `getBean()` call |
| **Prototype bean** | On **every** `getBean()` call (never cached) |


Internally, everything funnels into:

```java
Object bean = beanFactory.getBean("userService");
```

→ which calls `AbstractBeanFactory.doGetBean()` — the **master method** of bean creation.

---

## 🎴 The Master Flow

```text
getBean("userService")
        │
        ▼
AbstractBeanFactory.doGetBean()              ← master entry point
        │
        ├─ (1) Check 3-level singleton cache   ← found? return immediately
        ├─ (2) Merge BeanDefinition            ← parent+child → RootBeanDefinition
        │       (bean marked "currently in creation")
        ▼
createBean()
        │
        ├─ (3) HOOK ① postProcessBeforeInstantiation()
        │        └─ returns an object? → SHORT-CIRCUIT (skip everything)
        ▼
doCreateBean()
        │
        ├─ (4) createBeanInstance()
        │        ├─ Supplier?                → obtainFromSupplier()
        │        ├─ Factory method (@Bean)?  → instantiateUsingFactoryMethod()
        │        ├─ determineCandidateConstructors()      ← ctor selection
        │        ├─ ConstructorResolver + SimpleInstantiationStrategy
        │        └─ Constructor.newInstance()  ★ OBJECT EXISTS ★
        │           (wrapped in BeanWrapperImpl)
        │
        ├─ (5) postProcessMergedBeanDefinition()   ← AFTER the constructor (see §5.6)
        │
        ├─ (6) addSingletonFactory()          ← early exposure (circular deps)
        │
        ├─ (7) HOOK ② postProcessAfterInstantiation()
        │
        └─ (8) populateBean() ...             ← PHASE 3 begins (DI)
```

---

# 🔎 Step-by-Step Execution Flow

### 🧩 Step 1 — Check the Cache First (Singletons Only)

Before creating anything, Spring checks three maps in `DefaultSingletonBeanRegistry`:

```java
singletonObjects        // 1st-level: fully finished beans
earlySingletonObjects   // 2nd-level: semi-created beans (circular deps)
singletonFactories      // 3rd-level: factories of in-creation beans
```

- ✅ Found → returned immediately, **no instantiation happens**
- ❌ Not found → mark bean "in creation" → proceed

> 💡 **Deep insight:** This 3-level cache is exactly how Spring solves circular dependencies (`A` needs `B`, `B` needs `A`). If `B` is mid-creation when `A` asks for it, `A` receives an *early reference* of `B` — not a new object.

---
### 🧩 Step 2 — Merge the BeanDefinition

If the definition has a parent definition, Spring merges parent + child into a **`RootBeanDefinition`** — the final blueprint for creation.

---
### 🧩 Step 3 — HOOK ①: `postProcessBeforeInstantiation()`

Before calling the constructor, every registered `InstantiationAwareBeanPostProcessor` gets a chance:

```java
Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
```

- Return `null` (normal case) → Spring continues with normal creation
- Return an **object** → Spring **skips creation entirely**: it applies `postProcessAfterInitialization()` on that object and returns it as the bean — no constructor, no DI, no init callbacks

### ⚡ What is this used for? — **AOP Short-circuiting**

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    return null; // normal flow — let Spring create it
}
```

> 💡 **What is this used for?** This is the "escape hatch" — *"Can I create the object myself instead of Spring?"* It's rare, but this is how some Spring AOP `TargetSource` scenarios create beans lazily, from a pool, or per-invocation without invoking their constructor.

---
### 🧩 Step 4 — Constructor Resolution: *Which* Constructor?

This is where the biggest beginner myth dies:

> ❌ **Myth:** "Spring always needs a no-argument constructor."
> ✅ **Truth:** *Spring determines an appropriate constructor according to its constructor-resolution rules.*

**Resolution order (decision tree):**

```text
1. Supplier defined in BeanDefinition (programmatic, Spring 5+)  → use it
2. Factory method (@Bean method / XML factory-method)            → invoke the method
        (reflection invokes the FACTORY METHOD, not the target class constructor)
3. Explicit constructor args in the BeanDefinition               → ConstructorResolver
                                                                   finds a match
4. determineCandidateConstructors()  (AutowiredAnnotationBeanPostProcessor):
     a. Exactly ONE constructor exists        → use it automatically (Spring 4.3+,
                                                no @Autowired needed — the
                                                "greedy constructor" behavior)
     b. Multiple constructors, one annotated
        @Autowired(required = true)           → use that one
     c. Multiple constructors, none marked    → fall back to the NO-ARG constructor
5. Nothing resolvable + no no-arg constructor → BeanInstantiationException
```

**Why does the "no-args constructor" myth exist?** Because in the simplest case —

```java
@Component
public class UserService { }   // no explicit constructor
```

— Java itself provides a default no-arg constructor, so `new UserService()` is possible. The myth generalizes from the simplest case.

**Constructor injection proves the myth wrong:**

```java
@Component
public class UserService {
    private final UserRepository repository;

    // No no-arg constructor exists — Spring still instantiates this bean,
    // because it resolves the dependency first and passes it to THIS constructor.
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

> 💡 **Performance fact:** resolved constructors are cached — `AutowiredAnnotationBeanPostProcessor` keeps a `candidateConstructorsCache` per class, and the resolved constructor is stored on the `RootBeanDefinition` — so resolution cost is paid once.

---
### 🧩 Step 5 — Reflection: The Object Is Actually Created

**Why reflection at all?** Spring is a *generic framework*. It cannot hardcode `new UserService(); new OrderService(); ...` for every class — that would defeat the container. It must answer, at runtime: *"What class should I instantiate, and which constructor should I use?"* Java Reflection provides exactly this capability.

**The mechanics (simplified internal logic):**

```java
Class<?> clazz = UserService.class;                       // from the BeanDefinition

Constructor<?>[] all = clazz.getDeclaredConstructors();   // discovery
Constructor<?> ctor = /* ...resolution from step 5.4... */;

ctor.setAccessible(true);                                 // private ctors work!
Object rawInstance = ctor.newInstance(resolvedArgs);      // ★ CONSTRUCTOR RUNS ★
```

**`new` vs `newInstance()` — the crucial difference:**

| | Normal Java | Reflection |
|---|---|---|
| Class known at | **compile time** | **runtime** (dynamic) |
| Code | `new UserService()` | `clazz` → `ctor.newInstance()` |
| Used by | your application | generic frameworks (Spring, Jackson, JPA…) |

`constructor.newInstance()` is conceptually similar to `new UserService()` — the crucial difference is that the **framework determines the class dynamically**. That's why Spring can manage thousands of different classes it has never seen.

**The internal chain:**

```text
createBeanInstance()
   └─ SimpleInstantiationStrategy (default strategy)
        └─ BeanUtils.instantiateClass(constructor)
             └─ Constructor.newInstance(args)   ← JVM allocates & runs ctor
   └─ result wrapped in BeanWrapperImpl          ← prepares Phase 3 property injection
```

- `CglibSubclassingInstantiationStrategy` is the variant used when a bean definition declares **method injection** (`lookup-method` / `replaced-method`) — Spring generates a CGLIB subclass at runtime. (Same CGLIB tech behind AOP proxies later.)
- **Reflection isn't "Spring's constructor":** your class keeps its normal constructor; Spring merely invokes it dynamically. The result is a plain Java object — the *container's ownership* is what makes it a bean.

> **Where does the object live?** In the **JVM heap**, like any object. The container merely holds a *reference*. For a singleton, the object only enters the finished `singletonObjects` cache **after the entire lifecycle completes** — not at instantiation.


### If hook ① returned `null`, Spring instantiates the object via:

```
SimpleInstantiationStrategy.instantiate()
```

### How does Spring choose a constructor?

1. If BeanDefinition has explicit constructor args → matching constructor
2. If there's **one constructor** → that one
3. If **multiple constructors**:
   - Constructor with `@Autowired(required=true)` → preferred
   - Otherwise → **default no-arg constructor**

```java
@Component
public class UserService {

    private final EmailSender emailSender;

    // Spring uses THIS constructor (only one present)
    public UserService(EmailSender emailSender) {
        this.emailSender = emailSender;
        System.out.println("CONSTRUCTOR: UserService object created (still incomplete!)");
    }
}
```

> ### ‼️ At this moment:
> ✅ Object exists in heap memory
> ❌ `@Autowired` **fields** are still `null`
> ❌ `@PostConstruct` not yet called
> ❌ Bean is **NOT usable yet** — it's a half-built house

---
### 🧩 Step 6 — `postProcessMergedBeanDefinition()` (Runs **After** the Constructor)

Immediately after `createBeanInstance()`, this hook fires:

```java
void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName)
```

> 💡 **Deep insight:** This is where `AutowiredAnnotationBeanPostProcessor` **scans your fields/setters and caches *where* `@Autowired` / `@Value` annotations exist**. Injection hasn't happened yet — Spring is just preparing the injection map for Phase 3.

---
### 🧩 Step 7 — Early Exposure (Circular Dependency Prep)

For singletons currently in creation, Spring immediately registers a factory that can hand out this **half-built** object:

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

This only happens when: the bean is a singleton + it's currently in creation + `allowCircularReferences` is enabled (default `true` in raw Spring Framework; **Spring Boot 2.6+ disables it by default**).

This is why `B` can receive a reference of `A` while `A` is still mid-creation: **the object exists — it's just not finished.**

> 🚨 **Known limitation — constructor injection + circular dependency:**
> ```text
> A(B) needs B to be constructed, B(A) needs A to be constructed → 💥 BeanCurrentlyInCreationException
> ```
> Circular dependencies only work with **setter/field injection**, because the object must *exist* before it can be early-exposed — and with constructor injection it doesn't exist yet.
>
> 💡 **This fact alone proves instantiation and population are separate steps.**

---
### 🧩 Step  — HOOK ②: `postProcessAfterInstantiation()`

Right **after** the constructor, **before** dependency injection (technically the first step inside `populateBean()`):

```java
boolean postProcessAfterInstantiation(Object bean, String beanName)
```

- Return `true` → Spring continues with **property population** (`@Autowired`, XML `<property>`)
- Return `false` → Spring **skips all property injection** for this bean

> 💡 **Use case:** some frameworks (certain ORM / serialization tools) inject dependencies in their **own way** and return `false` here to disable Spring's injection entirely.

**This is the Phase 2 / Phase 3 boundary.**

---
# 🖼️ Big Picture First

At the end of Phase 1, the container had:

```java
Map<String, BeanDefinition> beanDefinitionMap  // only blueprints
```

Now Spring starts converting blueprints into objects:

```
getBean("userService")
        │
        ▼
Merged BeanDefinition (prepare + postProcessMergedBeanDefinition)
        │
        ▼
InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()  ← MID HOOK ①
        │
        ▼
Constructor runs → OBJECT EXISTS (but empty/incomplete)
        │
        ▼
InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()   ← MID HOOK ②
        │
        ▼
(Property population / @Autowired happens — covered in Phase 3)
```

💭 Think of `BeanDefinition` as an architectural blueprint. Bean Instantiation is the moment:

- ✅ A Java object is allocated in heap memory
- ✅ The constructor has executed
- ❌ `@Autowired` / `@Value` fields are still **null** (field/setter injection)
- ❌ `Aware` callbacks have not run
- ❌ `@PostConstruct` has not run
- ❌ No AOP proxy exists yet
- ❌ Bean is **not yet** in the finished-singleton cache — it's "in creation"

> Spring now holds a **raw instance** that is completely unaware of its surroundings.

---
## ‼️ Don't Confuse the Two "Post-Processor" Families

| | `BeanFactoryPostProcessor` | `InstantiationAwareBeanPostProcessor` |
|---|---|---|
| **Operates on** | BeanDefinition (metadata) | Bean **instance** |
| **When** | Phase 1 (before any bean creation) | Phase 2/3 (around each bean's creation) |
| **Analogy** | 📐 Modifies the blueprint | 🏗️/🛋️ Participates in building & furnishing |

*(Note: `InstantiationAwareBeanPostProcessor` extends `BeanPostProcessor`, so it also participates in the init-phase hooks — that's Phase 3 territory.)*

---

## 📋 End-of-Phase Checklist

### ✔ By the end of Phase 2:
- ✔ Constructor has run — **object exists in the JVM heap**
- ✔ Bean is marked "in creation" and **early-exposed** via singleton factory (circular dependency support)
- ✔ `postProcessMergedBeanDefinition` ran — injection metadata collected
- ✔ Both **mid hooks** ran: ① `postProcessBeforeInstantiation` (can short-circuit), ② `postProcessAfterInstantiation` (can disable DI)

### ❌ Not yet done (Phase 3+):
- ❌ `@Autowired` / `@Value` injection (property population)
- ❌ `BeanNameAware` / `ApplicationContextAware` callbacks
- ❌ `@PostConstruct` / `afterPropertiesSet()` / `init-method`
- ❌ AOP proxy creation decision

---