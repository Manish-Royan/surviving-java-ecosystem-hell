# Liskov Substitution Principle (LSP)

The Liskov Substitution Principle (LSP), introduced by Barbara Liskov in 1987, states:
> "Objects of a superclass shall be replaceable with objects of its subclasses without altering the correctness of the program."

> In formal terms: If $\phi(x)$ is a property provable about objects $x$ of type $T$, then $\phi(y)$ should be true for objects $y$ of type $S$ where $S$ is a subtype of $T$.

In plain software engineering: **A child class must honor every contract and expectation set by its parent class.** If substituting a child class causes unexpected exceptions, ignored behavior, or broken assumptions, you have violated LSP.

In Java, LSP is the principle that makes **polymorphism** safe. If a method accepts a parent class (e.g., `Employee`), you should be able to pass any child class (e.g., `ContractEmployee`) without the program crashing or behaving unexpectedly.

## 🚨 Example of Violating LSP

```java
// PARENT CLASS
class Employee {
    public void paySalary() {
        System.out.println("Paying base salary...");
    }
    
    public void payBonus() {
        System.out.println("Paying bonus...");
    }
}

// CHILD CLASS (LSP VIOLATION)
class ContractEmployee extends Employee {
    @Override
    public void payBonus() {
        // 🚨 RED FLAG: Throwing an exception breaks the parent's contract!
        throw new UnsupportedOperationException("Contract employees don't get a bonus");
    }
}

// CLIENT CODE
public class LSPViolationDemo {
    public static void main(String[] args) {
        Employee employee = new ContractEmployee(); // Polymorphism
        
        employee.paySalary(); // Works fine
        
        // 💥 CRASH! The client expected an Employee to handle payBonus() safely.
        // The subclass broke the promise made by the superclass.
        employee.payBonus(); 
    }
}
```

## 1. Deep Dive: Analyzing the Violation Code

### 📌 The Parent Class Contract

When `Employee` defines `public void payBonus()`, it makes an implicit behavioral promise to the rest of the application: *"Every `Employee` object can receive a bonus without failing."*

```java
// PARENT CLASS
class Employee {
    public void paySalary() { ... }
    public void payBonus()  { ... }
}

```

### 📌 The Subtype Breach

```java
class ContractEmployee extends Employee {
    @Override
    public void payBonus() {
        // 🚨 RED FLAG: Throwing an exception breaks the parent's contract!
        throw new UnsupportedOperationException("Contract employees don't get a bonus");
    }
}

```

By throwing `UnsupportedOperationException`, `ContractEmployee` lies about being an `Employee`. It inherits the method signature syntactically, but **refuses to support the behavior semantically**.

### Why This Destroys Polymorphism

In the client code:

```java
Employee employee = new ContractEmployee(); // Polymorphism promise
employee.paySalary(); // Works fine
employee.payBonus();  // 💥 CRASH! UnsupportedOperationException at runtime

```

The client code depends on the `Employee` type. Polymorphism promises that any valid `Employee` instance can be passed here. Because `ContractEmployee` violates LSP, the client code must now be modified to protect itself:

```java
// Forced workaround due to LSP violation (violates OCP as well!)
if (!(employee instanceof ContractEmployee)) {
    employee.payBonus();
}

```

---

## 2. Common Red Flags of LSP Violations

1. **Throwing Unhandled Exceptions:** Overriding a parent method to throw `UnsupportedOperationException` or `NotImplementedException`.
2. **Empty / No-Op Methods:** Overriding a method with an empty body `{ }` because the child doesn't need it.
3. **Hardcoded Type Checks (`instanceof` Sprawl):** Having to check `if (obj instanceof ChildClass)` before calling base class methods.
4. **Strengthened Preconditions:** Demanding stricter input conditions than the parent class required.
5. **Weakened Postconditions:** Returning broader or unfulfilled output conditions than the parent promised.

## 3. Deep Dive: Solution 1 (Interface Segregation)

Instead of forcing a single, fat `Employee` base class onto all employee types, Solution 1 splits capabilities into smaller, purpose-driven interfaces.

```java
// 1. Base contract for ALL employees
interface Payable {
    void paySalary();
}

// 2. Extended contract ONLY for bonus-eligible employees
interface BonusEligible extends Payable {
    void payBonus();
}

```

### How This Restores LSP

* **`FullTimeEmployee`** implements `BonusEligible` (and by extension `Payable`).
* **`ContractEmployee`** implements **only** `Payable`.

Now, if a method expects a `Payable`, both `FullTimeEmployee` and `ContractEmployee` can be passed without any fear of runtime crashes:

```java
public void processPayroll(Payable employee) {
    employee.paySalary(); // Guaranteed to work for ALL Payable instances
}

```

If a method specifically requires bonus processing, it accepts `BonusEligible`:

```java
public void processBonus(BonusEligible employee) {
    employee.payBonus(); // Guaranteed to work for ALL BonusEligible instances
}

```

The compiler now catches invalid operations at **compile time** instead of crashing at **runtime**.

---

## 4. Deep Dive: Solution 2 (Higher-Level Logic with Java 16+ Pattern Matching)

Sometimes your application processes a collection of base items (like `List<Payable>`), and you need to apply optional behavior safely.

```java
class BonusProcessor {
    public void processBonus(Payable employee) {
        // Safe, explicit type query using Pattern Matching for instanceof
        if (employee instanceof BonusEligible bonusEligibleEmployee) {
            bonusEligibleEmployee.payBonus();
        } else {
            System.out.println("Notice: This employee type is not eligible for a bonus.");
        }
    }
}

```

### Why Solution 2 Works Well

1. **LSP Compliant:** `Payable` never promises a bonus in the first place, so calling `processBonus(payable)` never breaks any contract.
2. **Type-Safe:** Pattern matching (`instanceof BonusEligible bonusEligibleEmployee`) extracts the capability cleanly without unsafe manual casting.
3. **Resilient:** Adding new non-eligible employee types (e.g., `Intern`, `Vendor`) won't require modifying `BonusProcessor`.

---

## ✅ Complete Solution 

```java

// 1. INTERFACE DEFINITIONS
interface Payable {
    void paySalary();
}

interface BonusEligible extends Payable {
    void payBonus();
}


// 2. CONCRETE IMPLEMENTATIONS
class FullTimeEmployee implements BonusEligible {
    private String name;

    public FullTimeEmployee(String name) {
        this.name = name;
    }

    @Override
    public void paySalary() {
        System.out.println("[FullTime] Paying monthly base salary to " + name);
    }

    @Override
    public void payBonus() {
        System.out.println("[FullTime] Paying annual performance bonus to " + name);
    }
}

class ContractEmployee implements Payable {
    private String name;

    public ContractEmployee(String name) {
        this.name = name;
    }

    @Override
    public void paySalary() {
        System.out.println("[Contract] Paying hourly wage to " + name);
    }
    
    // No payBonus() method. LSP is preserved.
}


// 3. HIGHER-LEVEL PROCESSOR
class PayrollProcessor {
    
    // Safely pays salary to ANY employee (LSP holds)
    public void payAllSalaries(Payable[] employees) {
        System.out.println("--- Processing Salaries ---");
        for (Payable emp : employees) {
            emp.paySalary();
        }
    }

    // Safely processes bonuses, handling eligibility gracefully
    public void processAllBonuses(Payable[] employees) {
        System.out.println("\n--- Processing Bonuses ---");
        for (Payable emp : employees) {
            if (emp instanceof BonusEligible bonusEligibleEmp) {
                bonusEligibleEmp.payBonus();
            } else {
                System.out.println("[Skipped] " + emp.getClass().getSimpleName() + " is not eligible for a bonus.");
            }
        }
    }
}


// 4. MAIN EXECUTION
public class LSPDeepDiveDemo {
    public static void main(String[] args) {
        // Create a mixed array of employees
        Payable[] staff = {
            new FullTimeEmployee("Alice"),
            new ContractEmployee("Bob"),
            new FullTimeEmployee("Charlie")
        };

        PayrollProcessor processor = new PayrollProcessor();

        // 1. Polymorphism works perfectly for base behavior (paySalary)
        processor.payAllSalaries(staff);

        // 2. Extended behavior (payBonus) is handled safely without crashes
        processor.processAllBonuses(staff);
    }
}
```

## 5. The Three Rules of LSP (Subcontracting Rules)
To deeply internalize LSP, remember that a subclass must respect three rules relative to its parent:
* Preconditions cannot be strengthened: A subclass method cannot require more strict input parameters than the parent method.
* Postconditions cannot be weakened: A subclass method must guarantee at least the same output/state as the parent method.
* Invariants must be preserved: The internal rules of the parent class (e.g., "salary must be > 0") must remain true in the subclass.
---