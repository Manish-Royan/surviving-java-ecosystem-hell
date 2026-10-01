# Overview of Spring Bean Lifecycle 

#### **Why the Lifecycle Matters**  
The Spring bean lifecycle is **not just a sequence of callbacks**—it’s a **programmable pipeline** where the container manages bean creation, dependency injection, initialization, and destruction. The lifecycle describes every step the Spring IoC container takes from **creating** a bean until **destroying** it.

Understanding it is critical for:  
- **AOP proxy creation** (e.g., `@Transactional`),  
- **Resource management** (e.g., database connections, thread, caches...),  
- **Debugging initialization failures**,  
- **Customizing behavior** with `BeanPostProcessor`, `BeanFactoryPostProcessor`, etc.

---

# ⌛ Lifecycle of Bean

<img width="1322" height="615" alt="JavaBean" src="https://github.com/user-attachments/assets/73aa051f-6bf0-4eb2-bff7-bd6cfc621b84" />

## **1st Phase: Loading Bean Definitions**
- Spring loads bean definitions from various sources such as `XML-configuration` files, `Java-based` configuration classes or `component scanning`.
- These bean definitions contain information about the bean classes, dependencies & other configuration details.
- Internally, Spring converts them into **BeanDefinition objects** and stores them in the **BeanDefinitionRegistry**.

##
## **2nd Phase: Bean Instantiation**
- Bean objects are created based on the loaded definitions.
- Spring uses **Reflection** to invoke the bean’s constructor (usually the public no-args constructor).
- At this point, the bean exists in memory but dependencies are not yet injected.

##
## **3rd Phase: Dependency Injection & Initialization**
- **Dependency Injection**: Spring resolves and injects required dependencies into the bean (via constructor, setters, or field injection).
- **BeanPostProcessor**: After injection, Spring allows custom logic to run before and after initialization (e.g., `@Autowired`, `@PostConstruct`, AOP proxies).
- **Initialization**:  
  - Calls custom init methods (defined in XML or `@Bean(initMethod="...")`).  
  - Executes `@PostConstruct` annotated methods.  
- At this stage, the bean is **fully ready for use**.

### Example:
```java
@Component
public class StudentService {
    @Autowired
    private StudentRepository repo;

    @PostConstruct
    public void init() {
        System.out.println("StudentService initialized!");
    }
}
```

##
## **4th Phase: Bean Destruction**
- When the container shuts down, Spring manages bean cleanup.
- **@PreDestroy** annotated methods or custom destroy methods are called.
- This ensures resources like database connections, threads, or caches are properly released.

### Example:
```java
@Component
public class ResourceManager {
    @PreDestroy
    public void cleanup() {
        System.out.println("Cleaning up resources...");
    }
}
```

---
### 📊 High-Level Flow (Diagram – Text Version)

```
1. Bean Definition Loading & Parsing
   ↓
2. Instantiation (create raw object)
   ↓
3. Populate Properties / Dependency Injection
   ↓
4. Aware callbacks (BeanNameAware, BeanClassLoaderAware, BeanFactoryAware…)
   ↓
5. BeanPostProcessor.postProcessBeforeInitialization(…)   ← many times
   ↓
6. Initialization phase
     • @PostConstruct
     • InitializingBean.afterPropertiesSet()
     • Custom init-method (XML / @Bean)
   ↓
7. BeanPostProcessor.postProcessAfterInitialization(…)   ← many times (AOP proxies created here)
   ↓
8. Bean is READY – put into cache / used by application
   ↓               (lives here during normal operation)
9. Container shutdown / close()
   ↓
10. Destruction phase
      • @PreDestroy
      • DisposableBean.destroy()
      • Custom destroy-method
```

### 🗨️Visual Summary 

```
BeanDefinition ──► Instantiation ──► DI ──► Aware ──► BPP before ──► Init (@PostConstruct etc.) ──► BPP after (AOP here) ──► Ready
                                                                                                    │
                                                                                                    ▼
                                                                                             Shutdown ──► Destroy (@PreDestroy etc.)
```


---
## ✅ Summary of Lifecycle
1. **Loading Bean Definitions** → Metadata is read and stored.  
2. **Bean Instantiation** → Object created via reflection.  
3. **Dependency Injection & Initialization** → Dependencies injected, init methods executed, post-processors applied.  
4. **Bean Destruction** → Cleanup before container shutdown.  
---
