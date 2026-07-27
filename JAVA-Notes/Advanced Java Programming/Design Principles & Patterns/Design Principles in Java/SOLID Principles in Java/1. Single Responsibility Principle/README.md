# Single Responsibility Principle (SRP)

The **Single Responsibility Principle (SRP)** is the cornerstone of SOLID. It states:  
> **"A class should have only one reason to change."**  

This isn't about *how many methods* a class has—it's about **cohesion**: *all parts of a class should evolve for the same reason*. Violating SRP leads to **"god classes"** that become fragile, hard to test, and impossible to maintain.

### 💭 Basic OOP thinking
```
A class represents real-world entities with properties and behavior
```
✔️ This is correct — this is basic OOP thinking.

### 🚨 But here’s the deeper truth (VERY IMPORTANT):
```
❌ A class is NOT just about representing an entity.
```

> ✅ A class is about: having ONE clear responsibility (reason to change)

---
## 🧾 Uncle Bob clarified SRP in two important ways:

1. **SRP is about people / actors, not technical “reasons”**  
   A module should be *responsible to one, and only one, actor* – where an actor is a group of stakeholders who request changes to that module.  
   Another wording for the Single Responsibility Principle is:  
   > Gather together the things that change for the same reasons. Separate those things that change for different reasons.

2. **“Reasons to change” are tied to business roles**  
   In his blog, he uses the example of an `Employee` class with:
   - `calculatePay()` – driven by the **CFO / finance** role  
   - `reportHours()` – driven by the **COO / operations** role  
   - `save()` – driven by the **CTO / DBA** role  
   If a change request from the CTO (database schema) forces changes that break `reportHours()`, the COO will be upset.  
   SRP is about **not mixing code that different people/roles care about for different reasons**.

So the deep definition is:

> **A module (class, package, service) should be responsible to a single actor / business role, so that changes requested by that actor don’t inadvertently break behavior other actors care about.**


---
## 🔍 Let's understand the real meaning of "Responsibility"

### Q. What is the responsibility❓
➡️ A responsibility isn’t just a method; it’s a family of functions that serves a particular actor. Actors can be:

- A business user (e.g., changing invoice calculation rules)
- A database administrator (e.g., changing how data is persisted)
- An operations team (e.g., changing logging format)
- Another system (e.g., changing an API contract)

> Each actor’s needs are a separate reason to change.

### Q. What acutally mean by "reason to change"❓
> "Reason to change" = Business rule or technical concern that triggers modification.

### **“Reason to change” = Responsibility.**  
> **A class should only be affected by one type of change in requirements.** 

- It means:  
  - If business rules change → one class should handle that.  
  - If formatting/output changes → another class should handle that.  
  - If persistence logic changes → yet another class should handle that.

👉 In short: **Each class should focus on one job.** If tomorrow your boss says “change how invoices are printed,” you should only touch the printing class, not the calculation or database class.

---
## 🚨 Example of Violating SRP
```java
public class Invoice {
    private double amount;

    public Invoice(double amount) {
        this.amount = amount;
    }

    // Business logic
    public double calculateTax() {
        return amount * 0.1;
    }

    // Persistence logic
    public void saveToDatabase() {
        System.out.println("Saving invoice to DB...");
    }

    // Presentation logic
    public void printInvoice() {
        System.out.println("Invoice amount: " + amount);
    }
}
```

### ⚠️ Problem:
- If tax rules change → modify `calculateTax()`.
- If database changes → modify `saveToDatabase()`.
- If printing format changes → modify `printInvoice()`.

This class has **three reasons to change**. That’s a violation of SRP.

---

## ✅ Refactored with SRP
```java
// Business logic
public class Invoice {
    private double amount;

    public Invoice(double amount) {
        this.amount = amount;
    }

    public double calculateTax() {
        return amount * 0.1;
    }

    public double getAmount() {
        return amount;
    }
}

// Persistence responsibility
public class InvoiceRepository {
    public void save(Invoice invoice) {
        System.out.println("Saving invoice to DB...");
    }
}

// Presentation responsibility
public class InvoicePrinter {
    public void print(Invoice invoice) {
        System.out.println("Invoice amount: " + invoice.getAmount());
    }
}
```

### Benefits:
- If tax rules change → only `Invoice`.
- If DB changes → only `InvoiceRepository`.
- If printing format changes → only `InvoicePrinter`.

<img width="1536" height="682" alt="SRP" src="https://github.com/user-attachments/assets/dd04320b-6f2c-4a31-b410-eb29c86434f4" />

Each class has **one clear responsibility** → one reason to change.

---
## 🌟 The Core Problem SRP Solves
When a class has multiple responsibilities, they become coupled. Changes to one responsibility can break another, making the class fragile, hard to test, and difficult to understand.

### 🚨 Consequences of Violating SRP
- **Fragility**: A change to logging logic breaks business rules.
- **Low cohesion**: Unrelated methods and fields are crammed together.
- **Poor testability**: You need to mock unrelated concerns just to test one thing.
- **Merge conflicts**: Multiple developers touch the same class for different reasons.
- **Code noise**: The class becomes a "**god class**" with thousands of lines.

## 🧩 Mental Model
### ❌ Instead of thinking:
> "What can I put inside this class?"

✅ Start thinking:
> "What should NOT be inside this class?"
> "What responsibility does this class have?"

### 🌟 Golden Rules of SRP: 
- Readable
- Maintainibility 
- Testibility 
- Modularity
- Reuseablity  
---
