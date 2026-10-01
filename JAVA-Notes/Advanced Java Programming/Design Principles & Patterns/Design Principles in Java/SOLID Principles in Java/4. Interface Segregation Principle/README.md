# Interface Segregation Principle (ISP)

The **Interface Segregation Principle (ISP)**, the "**I**" in SOLID, states:
> "Clients should not be forced to depend upon interfaces/methods that they do not use."

👉 In simpler terms: ***Many client-specific interfaces are better than one general-purpose ("fat") interface.***

> ISP is essentially the Single Responsibility Principle (SRP) applied to interfaces. Just as a class should have one reason to change, an interface should represent a single, cohesive role.

👉 Instead of creating one large, "fat" interface that attempts to cover all possible features, you should break it down into smaller, role-specific interfaces.

---

## 🚨 Example of Violating ISP

```java
// FAT INTERFACE: Groups unrelated responsibilities
public interface Machine {
    void print();
    void scan();
    void fax();
    void staple();
}

// VIOLATION: Forced to implement methods it cannot support
public class BasicPrinter implements Machine {
    @Override
    public void print() {
        System.out.println("Printing document...");
    }

    @Override
    public void scan() { 
        throw new UnsupportedOperationException("BasicPrinter cannot scan");
    }

    @Override
    public void fax() { 
        throw new UnsupportedOperationException("BasicPrinter cannot fax");
 }

    @Override
    public void staple() { 
        throw new UnsupportedOperationException("BasicPrinter cannot staple");
    }
}
```

### Q. Why this is a critical design flaw:
1. Forced Dependency: `BasicPrinter` is forced to know about `scan`, `fax`, and `staple`, even though it’s just a printer.
2. LSP Violation: As we learned in the LSP deep dive, throwing `UnsupportedOperationException` breaks the contract of the interface.
3. Client Confusion: If a method accepts a `Machine` parameter, the caller assumes it can safely call `.scan()`. If they pass a BasicPrinter, the app crashes.
4. Ripple Effect: If you add a new method to `Machine` (e.g., `void shred()`), every single class implementing `Machine` must be updated, even if 90% of them don't shred.

---

## 1. Deep Dive: Analyzing the Violation Code

### 📌 The "Fat" Interface

```java
// FAT INTERFACE: Groups unrelated responsibilities
public interface Machine {
    void print();
    void scan();
    void fax();
    void staple();
}

```

The `Machine` interface bundles four completely distinct capabilities—printing, scanning, faxing, and stapling—into a single contract. It assumes every machine in the system is a high-end multi-function device.

### 📌 The Forced Lie in `BasicPrinter`

```java
public class BasicPrinter implements Machine {
    @Override
    public void print() {
        System.out.println("Printing document...");
    }

    @Override
    public void scan() { 
        throw new UnsupportedOperationException("BasicPrinter cannot scan");
    }

    @Override
    public void fax() { 
        throw new UnsupportedOperationException("BasicPrinter cannot fax");
    }

    @Override
    public void staple() { 
        throw new UnsupportedOperationException("BasicPrinter cannot staple");
    }
}

```

### Why This Destroys Code Quality

1. **Polluted API:** `BasicPrinter` exposes `scan()`, `fax()`, and `staple()` to callers, even though it cannot perform any of them.
2. **Breaks Liskov Substitution Principle (LSP):** A client expecting a `Machine` might call `machine.scan()`. Passing a `BasicPrinter` causes an unhandled runtime crash (`UnsupportedOperationException`).
3. **Unnecessary Recompilation:** If the `fax()` signature changes (e.g., adding a phone number parameter), `BasicPrinter` must be recompiled and redeployed, even though it doesn't use faxing.

---

## ✅ Refactored Code

```java
// 1. SEGREGATED INTERFACES
interface Printer {
    void print();
}

interface Scanner {
    void scan();
}

interface Fax {
    void fax();
}

interface Stapler {
    void staple();
}


// 2. CONCRETE IMPLEMENTATIONS
class BasicPrinter implements Printer {
    @Override
    public void print() {
        System.out.println("[BasicPrinter] Printing document...");
    }
}

class MultiFunctionMachine implements Printer, Scanner, Fax, Stapler {
    @Override
    public void print() { System.out.println("[MFM] Printing document..."); }
    @Override
    public void scan() { System.out.println("[MFM] Scanning document..."); }
    @Override
    public void fax() { System.out.println("[MFM] Sending fax..."); }
    @Override
    public void staple() { System.out.println("[MFM] Stapling pages..."); }
}

// A dedicated scanner device
class DedicatedScanner implements Scanner {
    @Override
    public void scan() {
        System.out.println("[DedicatedScanner] High-res scanning...");
    }
}


// 3. CLIENT CODE (The real benefit of ISP)
class OfficeWorker {
    
    // ✅ SAFE: This method ONLY requires printing capability.
    // It can accept a BasicPrinter OR a MultiFunctionMachine.
    // It is completely unaware of scanning/faxing, preventing misuse.
    public void printDocument(Printer device) {
        System.out.println("Worker: Preparing to print...");
        device.print();
    }

    // ✅ SAFE: This method specifically requires scanning capability.
    // Passing a BasicPrinter here would be a compile-time error (which is good!).
    public void scanDocument(Scanner device) {
        System.out.println("Worker: Preparing to scan...");
        device.scan();
    }
}


// 4. MAIN EXECUTION
public class ISPDeepDiveDemo {
    public static void main(String[] args) {
        OfficeWorker worker = new OfficeWorker();
        
        Printer basicPrinter = new BasicPrinter();
        MultiFunctionMachine mfm = new MultiFunctionMachine();
        DedicatedScanner scanner = new DedicatedScanner();

        System.out.println("--- Printing Tasks ---");
        worker.printDocument(basicPrinter); // Works perfectly
        worker.printDocument(mfm);          // Works perfectly

        System.out.println("\n--- Scanning Tasks ---");
        worker.scanDocument(mfm);           // Works perfectly
        worker.scanDocument(scanner);       // Works perfectly
        
        // COMPILE-TIME SAFETY: 
        /* worker.scanDocument(basicPrinter); */
        // ^ This line would cause a compilation error, protecting us from runtime crashes. 
        // The fat interface would have allowed this, leading to a crash.
    }
}
```

### Benefits:
* `BasicPrinter` only depends on Printer.
* `MultiFunctionMachine` can implement multiple interfaces.
* No class is forced to implement unused methods.

## 2. Deep Dive: Analyzing the Refactored Code

By segregating the interface into single-behavior role interfaces, each component stays focused, honest, and decoupled.

```java
// 1. Segregated Interfaces (Single Responsibility per interface)
public interface Printer { void print(); }
public interface Scanner { void scan(); }
public interface Fax     { void fax();   }
public interface Stapler { void staple(); }

```

### 📌 Clean Implementation (`BasicPrinter`)

```java
public class BasicPrinter implements Printer {
    @Override
    public void print() {
        System.out.println("BasicPrinter: Printing document...");
    }
}

```

`BasicPrinter` subscribes **only** to the capability it actually possesses. It contains zero dead code, zero thrown exceptions, and zero dummy methods.

### 📌 Flexible Composition (`MultiFunctionMachine`)

```java
public class MultiFunctionMachine implements Printer, Scanner, Fax, Stapler {
    @Override
    public void print()  { System.out.println("MFM: Printing..."); }

    @Override
    public void scan()   { System.out.println("MFM: Scanning..."); }

    @Override
    public void fax()    { System.out.println("MFM: Faxing..."); }

    @Override
    public void staple() { System.out.println("MFM: Stapling..."); }
}

```

Instead of inheriting a massive base interface, `MultiFunctionMachine` **composes multiple fine-grained behaviors**.

---

## 3. The Mental Model

```text
BEFORE (Monolithic Interface):
┌─────────────────────────────────────────┐
│                 Machine                 │
│  [print()]  [scan()]  [fax()] [staple()]│
└────────────────────┬────────────────────┘
                     │
           ┌─────────┴─────────┐
           ▼                   ▼
    BasicPrinter      MultiFunctionMachine
 (Forced to throw      (Uses all methods)
   exceptions!)

NOW (Role-Based Segregated Interfaces):
  ┌─────────┐   ┌─────────┐   ┌─────┐   ┌─────────┐
  │ Printer │   │ Scanner │   │ Fax │   │ Stapler │
  └────┬────┘   └────┬────┘   └──┬──┘   └────┬────┘
       │             │           │           │
       ├─────────────┼───────────┼───────────┤
       │             │           │           │
       ▼             ▼           ▼           ▼
  BasicPrinter      MultiFunctionMachine
 (Implements ONLY    (Composes ALL four
   Printer)            interfaces)

```

* **Before:** One monolithic interface for everything $\rightarrow$ High coupling, fragile contracts, runtime exceptions.
* **Now:** Multiple small interfaces based on behavior $\rightarrow$ Modular, composable, type-safe at compile time.

---

## 4. Industry Thinking Shift

How senior developers design object relationships:

```
❌ Novice Developer Thinking:  "What objects ARE?"
   └── "A BasicPrinter IS A Machine, so it must implement Machine."

✅ Senior Developer Thinking: "What BEHAVIORS do they support?"
   └── "A BasicPrinter SUPPORTS printing, so it implements Printer."

```

When you focus on **what objects are** (taxonomy), you build rigid, heavy inheritance hierarchies. When you focus on **what behaviors they support** (contracts), you create lean, decoupled interfaces that fit together seamlessly.

---

## 5. ISP vs. Other SOLID Principles

| Principle | Core Difference |
| --- | --- |
| **Single Responsibility Principle (SRP)** | Focuses on **classes** (A class should have only one reason to change). |
| **Interface Segregation Principle (ISP)** | Focuses on **interfaces** (Clients should not see methods they don't need). |
| **Liskov Substitution Principle (LSP)** | Focuses on **behavioral consistency** (Subtypes must fulfill supertype contracts without throwing unsupported exceptions). |

> **Key Rule of Thumb:** Small, single-purpose interfaces naturally prevent LSP violations by eliminating empty or exception-throwing dummy methods.

---

## 6. How ISP Interlocks with Other SOLID Principles

| Principle Connection | Relationship |
| --- | --- |
| **ISP + SRP** | **SRP** keeps classes focused on a single responsibility; **ISP** keeps interfaces focused on a single role. |
| **ISP + LSP** | **ISP prevents LSP violations**. When interfaces are small, classes never need to override methods with empty bodies or `UnsupportedOperationException`. |
| **ISP + DIP** | **Dependency Inversion** relies on depending on abstractions. ISP ensures those abstractions are lightweight and don't leak unnecessary methods to clients. |

---

### 📝 Takeaway
* ISP keeps interfaces lean and meaningful.
* Classes only implement what they truly need.
* In Spring Boot, this principle is reflected in how you often see small, role-specific interfaces (e.g., `CrudRepository`, `JpaRepository`) instead of one giant “`DatabaseOperations`” interface.

---