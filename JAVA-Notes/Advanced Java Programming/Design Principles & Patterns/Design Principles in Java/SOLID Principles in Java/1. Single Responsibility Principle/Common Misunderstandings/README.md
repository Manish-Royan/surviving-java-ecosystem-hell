# Common Misunderstandings about SRP

The **Single Responsibility Principle (SRP)**—the "S" in SOLID—is often summarized as *"A class should have one reason to change."* Robert C. Martin (Uncle Bob) defines that "reason to change" specifically around **a single actor or business stakeholder**.

---

## 1. SRP does NOT mean "one method per class"

* **Misunderstanding:** "If a class does one thing, it should only have one method."
* **Reality:** A class can have many methods as long as they all belong to the same cohesive role or boundary.

```java
// ✅ GOOD: Multiple methods, but SINGLE responsibility (Managing User Domain Data)
public class User {
    private String name;
    private String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    // Cohesive getters/setters & domain validation
    public String getName() { return name; }
    public String getEmail() { return email; }

    public boolean isValid() {
        return email != null && email.contains("@");
    }
}

```

> **Why this satisfies SRP:** Every method in `User` answers to the same actor: **User Domain Management**. If the rules for what constitutes a valid user change, only this class changes.

---

## 2. SRP does NOT apply only to classes

* **Misunderstanding:** "SRP is just an Object-Oriented Class rule."
* **Reality:** SRP applies at every architecture level—methods, classes, packages, modules, and microservices.

```java
// -------------------------------------------------------------
// Package Level Separation (Architecture Level SRP)
// -------------------------------------------------------------

// Package: com.app.billing (Serves the Finance/Accounting Actor)
package com.app.billing;

public class InvoiceCalculator {
    public double calculateTotalWithTax(double amount) {
        return amount * 1.20; // 20% VAT logic
    }
}

// Package: com.app.notification (Serves the Marketing/Comms Actor)
package com.app.notification;

public class EmailService {
    public void sendInvoiceEmail(String email, double total) {
        // Code to talk to SMTP server...
        System.out.println("Sending receipt of $" + total + " to " + email);
    }
}

```

> **Why this satisfies SRP:** The `billing` package changes when tax laws change (CFO / Accounting actor). The `notification` package changes when email templates or providers change (Marketing / Ops actor). Separating them by **packages/modules** enforces SRP beyond just classes.

---

## 3. SRP is NOT "make classes as small as possible"

* **Misunderstanding:** "To follow SRP, break everything into tiny 5-line classes."
* **Reality:** Hyper-fragmentation creates high complexity, hard-to-read code, and arbitrary boundaries that aren't tied to business roles.

```java
// ❌ BAD: Over-fragmented nightmare in the pursuit of "small classes"
class UserNameGetter { public String get(User u) { return u.getName(); } }
class UserEmailValidator { public boolean validate(User u) { return u.getEmail().contains("@"); } }

// ✅ GOOD: Focused, well-aligned class (Size is irrelevant, cohesion is key)
public class OrderProcessor {
    private final InventoryService inventory;
    private final PaymentGateway payment;

    public OrderProcessor(InventoryService inventory, PaymentGateway payment) {
        this.inventory = inventory;
        this.payment = payment;
    }

    public void process(Order order) {
        inventory.reserveStock(order.getItems());
        payment.charge(order.getAmount());
        order.setStatus(OrderStatus.COMPLETED);
    }
}

```

> **Why this satisfies SRP:** `OrderProcessor` coordinates fulfillment. Even if it grows to 100 lines, it has **one focus**: executing the order fulfillment workflow.

---

## 4. SRP is NOT about avoiding all coupling

* **Misunderstanding:** "Classes shouldn't depend on each other or pass objects between them."
* **Reality:** Software *must* collaborate to work. SRP means coupling things that **change together** and decoupling things that **change for different reasons**.

```java
// ❌ BAD: Mixing two different reasons to change in one class
public class OrderReport {
    public void generateAndSaveToDb(Order order) {
        // Reason to change 1: Report formatting (Product team)
        String report = "ORDER ID: " + order.getId() + ", TOTAL: $" + order.getAmount();

        // Reason to change 2: Persistence mechanism (DBA team)
        System.out.println("Executing SQL: INSERT INTO reports VALUES ('" + report + "')");
    }
}

// ✅ GOOD: Separating concerns while maintaining necessary, healthy coupling
public class OrderReportFormatter {
    public String format(Order order) {
        // Changes ONLY if report design changes
        return "ORDER ID: " + order.getId() + ", TOTAL: $" + order.getAmount();
    }
}

public class OrderReportRepository {
    public void save(String reportContent) {
        // Changes ONLY if database strategy changes
        System.out.println("Executing SQL: INSERT INTO reports VALUES ('" + reportContent + "')");
    }
}

```

> **Why this satisfies SRP:** `OrderReportFormatter` and `OrderReportRepository` can still be used together by a service layer, but a SQL database migration won't break your PDF/Text formatting code, and a UI redesign won't break your database queries.