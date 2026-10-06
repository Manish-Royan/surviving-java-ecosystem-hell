## How Spring's container simply replaces the `main` method's manual assembly work automatically

How Spring Core (`AnnotationConfigApplicationContext`) takes over object creation and assembly automatically using annotations (`@Component`, `@Repository`, `@Service`, and `@Qualifier` or `@Primary`).

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Primary;


// 1. Abstraction: Contract for Data Storage
interface StudentRepository {
    void save(String name);
}


// 2. Concrete Dependency Implementations (Managed Spring Beans)
@Repository("databaseRepo") // @Repository tells Spring to manage this class as a Data Access bean
@Primary // Marks this as the default bean when multiple StudentRepository implementations exist
class DatabaseStudentRepository implements StudentRepository {
    @Override
    public void save(String name) {
        System.out.println("[Database] Successfully saved student: " + name);
    }
}

@Repository("cloudRepo")
class CloudStudentRepository implements StudentRepository {
    @Override
    public void save(String name) {
        System.out.println("[AWS Cloud] Uploaded student JSON payload for: " + name);
    }
}


// 3. Dependent Class: Business Logic Layer
@Service // @Service tells Spring to instantiate this class and manage its lifecycle
class StudentService {
    private final StudentRepository repository;

    /* IMPLICIT CONSTRUCTOR INJECTION: Spring automatically inspects this constructor, finds StudentRepository, 
        and passes the @Primary bean (DatabaseStudentRepository) automatically. */
    public StudentService(StudentRepository repository) {
        this.repository = repository;
    }

    public void registerStudent(String name) {
        if (name == null || name.trim().isEmpty()) {
            System.out.println("Error: Student name cannot be empty!");
            return;
        }

        System.out.println("Processing student registration for: " + name);
        repository.save(name);
        System.out.println("Registration complete!\n");
    }
}


// 4. Spring Configuration Class
@Configuration
@ComponentScan // Tells Spring to automatically scan for @Service, @Repository, and @Component
class AppConfig {
}


// 5. Main Class: Spring Container Replaces Manual Assembly
public class MainApp {
    public static void main(String[] args) {
        // STEP 1: Boot up the Spring IoC Container
        // Spring automatically scans, creates instances, and wires dependencies together!
        AnnotationConfigApplicationContext context = 
                new AnnotationConfigApplicationContext(AppConfig.class);

        // STEP 2: Ask the Spring Container for the ready-to-use StudentService bean
        StudentService studentService = context.getBean(StudentService.class);

        // STEP 3: Execute business workflow
        System.out.println("--- Spring Core Auto-Injected Execution ---");
        studentService.registerStudent("John Doe");

        // Close the container context
        context.close();
    }
}

```

### Differences from Manual DI:

| Feature / Aspect | Plain Java POJO (Manual DI) | Spring Core (Framework-Managed DI) |
| --- | --- | --- |
| **Object Creation** | **Manual:** You explicitly create instances using the `new` keyword (`new DatabaseStudentRepository()`). | **Automated:** Spring's **Inversion of Control (IoC) Container** scans, instantiates, and manages objects (Beans) automatically. |
| **Dependency Wiring** | **Explicit:** You pass dependencies into constructors manually inside `main` or a factory class. | **Implicit:** Spring inspects classes (via annotations like `@Service`, `@Repository`) and automatically injects matching dependencies into constructors. |
| **Role of `main()`** | **Assembler + Executor:** Acts as the "manual injector" responsible for assembling the object graph before running business logic. | **Bootstrapper:** Simply starts the Spring context (`AnnotationConfigApplicationContext`) and retrieves top-level components to execute. |
| **Class Annotations** | **None:** Relies on standard Java interfaces, classes, and plain constructors. | **Required:** Uses stereotype annotations (`@Component`, `@Repository`, `@Service`, `@Qualifier`, `@Primary`) to mark classes for management. |
| **Handling Multiple Implementations** | **Code Swap:** You manually change which concrete object is passed to `new StudentService(...)`. | **Declarative Swap:** You use annotations like `@Primary` or `@Qualifier("cloudRepo")` to tell Spring which implementation to wire without modifying assembly code. |
| **Lifecycle & Scope Management** | **Manual:** You control when objects are created, garbage-collected, or reused. | **Container-Managed:** Spring handles bean lifecycle (creation, initialization, Destruction) and scopes (Singleton, Prototype, Request, Session). |
| **External Overhead** | **Zero:** Runs on standard JDK with no external libraries or runtime startup delay. | **Framework Overhead:** Requires Spring Core dependencies and adds a small startup time to scan classpath and build the IoC context. |

---