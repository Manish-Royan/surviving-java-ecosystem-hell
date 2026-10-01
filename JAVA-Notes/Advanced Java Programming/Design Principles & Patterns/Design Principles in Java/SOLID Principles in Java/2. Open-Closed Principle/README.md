# Open/Closed Principle (OCP)

The **Open/Closed Principle** states:

> “Software entities (classes, modules, functions) should be open for extension, but closed for modification.”

- **Open for extension** → You can add new behavior without touching existing code.

- **Closed for modification** → Once a class is tested and stable, you shouldn’t keep editing it every time requirements change.

> 👉 This is where: You learn how to add new features without breaking old code

> 👉 The idea is: *Don’t keep rewriting old code; instead, extend it with new classes or methods*

---
### Q. What "open" and "closed" really mean?

| Term | Meaning in practice |
|---|---|
| **Open for extension** | New behavior can be added — typically by implementing an existing interface, subclassing an abstract type, or supplying a new strategy/handler. |
| **Closed for modification** | The source code of the entity (and especially the code that *uses* the abstraction) does not need to be edited, retested, or redeployed when behavior is added. |

The crucial nuance: it's the consumer code that should be closed. The new code (the new implementation) is obviously being written — that's the extension. OCP is violated when adding a new variant forces you to open up and edit existing code that ought to be stable.

---
## 🚨 Example of Violating OCP
The most common OCP violation in Java is a growing `if/else` or switch block that handles different types.
```java
public class DiscountCalculator {
    public double calculate(String customerType, double amount) {
        if (customerType.equals("Regular")) {
            return amount * 0.1;
        } else if (customerType.equals("Premium")) {
            return amount * 0.2;
        } else {
            return 0;
        }
    }
}
```

### Problem:
- If tomorrow you add a new customer type (e.g., “VIP”), you must modify this class.
- Every change risks breaking existing logic.

---
## ✅ Refactored with OCP (Polymorphism & Abstraction)

To comply with OCP, we shift behavior into pluggable components.

```java
// Abstraction
public interface DiscountPolicy { // Open for Extension: New behaviors are added by creating new implementations of DiscountPolicy.
    double applyDiscount(double amount);
}

// Extension 1 (Regular Customer)
public class RegularDiscount implements DiscountPolicy {
    public double applyDiscount(double amount) {
        return amount * 0.1;
    }
}

// Extension 2 (Premium)
public class PremiumDiscount implements DiscountPolicy {
    public double applyDiscount(double amount) {
        return amount * 0.2;
    }
}

// Context class - Open for extension but close for modification
public class DiscountCalculator { 
    // NOTE: Closed for Modification: You don't touch DiscountCalculator or existing discount implementations when adding new policies.
    private DiscountPolicy discountPolicy; // DiscountCalculator HAS-A DiscountPolicy

    // Constructor Injection
    public DiscountCalculator(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public double calculate(double amount) {
        return discountPolicy.applyDiscount(amount);
    }
} 

// Main Class
public class Main {
    public static void main(String[] args) {
        double originalAmount = 1000.0;

        // Using Regular Discount
        DiscountPolicy regularPolicy = new RegularDiscount();
        DiscountCalculator regularCalculator = new DiscountCalculator(regularPolicy);
        double regularDiscount = regularCalculator.calculate(originalAmount);

        // Using Premium Discount
        DiscountPolicy premiumPolicy = new PremiumDiscount();
        DiscountCalculator premiumCalculator = new DiscountCalculator(premiumPolicy);
        double premiumDiscount = premiumCalculator.calculate(originalAmount);

        // Output results
        System.out.println("Original Amount: $" + originalAmount);
        System.out.println("Regular Discount: $" + regularDiscount + " (Final: $" + (originalAmount - regularDiscount) + ")");
        System.out.println("Premium Discount: $" + premiumDiscount + " (Final: $" + (originalAmount - premiumDiscount) + ")");
    }
}
```

### Benefits:
- If you add VIPDiscount, you just create a new class → no need to touch DiscountCalculator.
- The system is closed for modification but open for extension.

---
### 👁️‍🗨️ Visual Mental Model - Think of OCP like a power socket:
- The socket itself doesn’t change (closed for modification). | `DiscountPolicy (interface)` = contract (like a socket shape).
- You can plug in new devices (open for extension). | `RegularDiscount`, `PremiumDiscount`, `VIPDiscount` (classes) = different plugs that fit the socket.
- The socket remains stable, but supports new functionality. | `DiscountCalculator` (context) = the wall socket. It doesn’t change when new plugs are added.

---
### 📝 Takeaway
- OCP prevents “code rot” by avoiding constant edits to stable classes.
- It encourages polymorphism and interfaces/abstract classes.
- In Spring Boot, this principle is everywhere: you extend behavior by adding `beans`, not by editing the framework’s core classes.

----