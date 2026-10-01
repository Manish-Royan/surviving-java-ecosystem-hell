## What is a Sealed Class in Java?

Introduced in Java 17, a **sealed class (or interface)** lets us explicitly declare **which specific classes or interfaces are allowed to extend or implement it**.

In traditional Java:

* An `interface` or non-final `class` is **open to everyone** (`public interface DiscountPolicy` can be implemented by any class in any project).
* A `final class` is **completely closed** (no one can extend it).
* A **sealed class/interface** is **controlled access**—you say: *"Only these specific classes are allowed to extend me, nobody else."*

```java
// Only RegularDiscount, PremiumDiscount, and VIPDiscount can implement this!
public sealed interface DiscountPolicy permits RegularDiscount, PremiumDiscount, VIPDiscount {
    double applyDiscount(double amount);
}

```

---

## OCP with Sealed Classes

Here is how our OCP refactored code looks using Java 17+ Sealed Interfaces, Records, and Pattern Matching.

### 1. The Sealed Abstraction

Every implementing class must explicitly declare its inheritance modifier: `final`, `sealed`, or `non-sealed`.

```java
// Sealed Interface: Permitting exactly 3 implementations
public sealed interface DiscountPolicy permits RegularDiscount, PremiumDiscount, VIPDiscount {
    double applyDiscount(double amount);
}

// 1. Regular Discount (final = no further extension)
public final record RegularDiscount() implements DiscountPolicy {
    @Override
    public double applyDiscount(double amount) {
        return amount * 0.10;
    }
}

// 2. Premium Discount
public final record PremiumDiscount() implements DiscountPolicy {
    @Override
    public double applyDiscount(double amount) {
        return amount * 0.20;
    }
}

// 3. VIP Discount
public final record VIPDiscount() implements DiscountPolicy {
    @Override
    public double applyDiscount(double amount) {
        return amount * 0.30;
    }
}

```
---

> ### NOTE: "Sealed classes restrict who can extend a class. Does this violate OCP? No. It makes OCP safer."

➡️ At first glance, restricting extension sounds like the exact opposite of *"Open for Extension"*. But in modern system design, **uncontrolled extension is a liability**.

➡️ Here is why sealing **enhances and secures** OCP instead of violating it:

### 1. It Protects Core Business Boundaries

OCP says a system should be open for *intended* extension. It does **not** mean your core domain model should allow third-party code or unauthorized developers to inject arbitrary, unsupported behavior.

* Sealing defines a **known, bounded set of extensions** within your domain module.
* If a new discount type (e.g., `EmployeeDiscount`) needs to be added, you update the `permits` clause and add the new record inside your module. The caller code (`OrderService`) remains completely untouched and closed for modification!

### 2. Exhaustive Pattern Matching (The Ultimate Developer Safety Net)

In modern Java (Java 17–21+), sealed types allow the compiler to check for **exhaustiveness** in `switch` expressions.

If `DiscountPolicy` is **unsealed**, a switch needs an unsafe `default` clause:

```java
// WITHOUT Sealed Classes: Compiler forces a 'default' fallback
public double getBonusPoints(DiscountPolicy policy) {
    return switch (policy) {
        case RegularDiscount r -> 10;
        case PremiumDiscount p -> 20;
        default -> 0; // Dangerous! If someone adds VIPDiscount, compiler won't warn you!
    };
}

```

With **Sealed Classes**, the compiler knows **all possible implementations**:

```java
// WITH Sealed Classes: Exhaustive compile-time safety!
public double getBonusPoints(DiscountPolicy policy) {
    return switch (policy) {
        case RegularDiscount r -> 10;
        case PremiumDiscount p -> 20;
        case VIPDiscount v     -> 30; // No 'default' branch needed!
    };
}

```

#### What happens when you add a new discount tomorrow?

If you add `SeasonalDiscount` to `permits`, the **Java compiler immediately gives a compile error** on the `switch` expression above, telling you: *"Hey! You added a new discount type, but you haven't handled its bonus points here yet."*

---

### 📌 Example:

```java
package com.example.ocp;

/**
 * STEP 1: Define the Abstraction (Sealed Interface)
 * --------------------------------------------------
 * We restrict extension to a known, bounded set of implementations.
 * This secures our domain model while keeping it open to planned additions.
 */
public sealed interface DiscountPolicy permits RegularDiscount, PremiumDiscount, VIPDiscount {
    /**
     * Calculates the discount amount based on the total order value.
     *
     * @param amount The original order total.
     * @return The monetary discount value.
     */
    double applyDiscount(double amount);
}


/**
 * STEP 2: Concrete Implementations (Records)
 * ------------------------------------------
 * Modern Java 'records' provide concise, immutable data carriers.
 * Each implementation represents a specific discount strategy (IS-A DiscountPolicy).
 */

// 1. Regular Customer Discount (10%)
public final record RegularDiscount() implements DiscountPolicy {
    @Override
    public double applyDiscount(double amount) {
        return amount * 0.10;
    }
}

// 2. Premium Customer Discount (20%)
public final record PremiumDiscount() implements DiscountPolicy {
    @Override
    public double applyDiscount(double amount) {
        return amount * 0.20;
    }
}

// 3. VIP Customer Discount (30%)
public final record VIPDiscount() implements DiscountPolicy {
    @Override
    public double applyDiscount(double amount) {
        return amount * 0.30;
    }
}


/**
 * STEP 3: Context Class (OrderService)
 * ------------------------------------
 * Demonstrates HAS-A Relationship (Composition) + Constructor Injection.
 * 
 * CLOSED FOR MODIFICATION: OrderService does not need to change when a new discount policy is added.
 * OPEN FOR EXTENSION: It can execute any new policy injected into it at runtime.
 */
public class OrderService {

    // HAS-A DiscountPolicy (Decoupled abstraction reference)
    private final DiscountPolicy discountPolicy;

    // Behavior injected from outside
    public OrderService(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    /**
     * Calculates final price after applying the injected discount policy.
     */
    public double processOrder(double totalAmount) {
        double discount = discountPolicy.applyDiscount(totalAmount);
        return totalAmount - discount;
    }

    /**
     * MODERN JAVA FEATURE: Exhaustive Pattern Matching with Sealed Classes
     * --------------------------------------------------------------------
     * Because 'DiscountPolicy' is sealed, the Java compiler knows ALL possible types.
     * No 'default' branch is needed! If a 4th policy is added to the 'permits' list later,
     * the compiler will FORCE us to handle it here (failing fast at compile-time).
     */
    public int calculateLoyaltyPoints(DiscountPolicy policy) {
        return switch (policy) {
            case RegularDiscount r -> 10;
            case PremiumDiscount p -> 25;
            case VIPDiscount v     -> 50;
            // No default clause needed! The compiler guarantees exhaustiveness.
        };
    }
}


/**
 * STEP 4: Main Execution
 * ----------------------
 * External wiring logic that instantiates strategies and injects them into the service.
 */
public class Main {
    public static void main(String[] args) {
        double orderTotal = 1000.0;

        System.out.println("=== Processing Orders with Modern OCP ===\n");

        // 1. Process Regular Customer Order
        DiscountPolicy regularPolicy = new RegularDiscount();
        OrderService regularOrder = new OrderService(regularPolicy);
        double regularFinal = regularOrder.processOrder(orderTotal);
        int regularPoints = regularOrder.calculateLoyaltyPoints(regularPolicy);

        System.out.printf("Regular Order -> Payable: $%.2f | Earned Points: %d%n", regularFinal, regularPoints);

        // 2. Process Premium Customer Order
        DiscountPolicy premiumPolicy = new PremiumDiscount();
        OrderService premiumOrder = new OrderService(premiumPolicy);
        double premiumFinal = premiumOrder.processOrder(orderTotal);
        int premiumPoints = premiumOrder.calculateLoyaltyPoints(premiumPolicy);

        System.out.printf("Premium Order -> Payable: $%.2f | Earned Points: %d%n", premiumFinal, premiumPoints);

        // 3. Process VIP Customer Order
        DiscountPolicy vipPolicy = new VIPDiscount();
        OrderService vipOrder = new OrderService(vipPolicy);
        double vipFinal = vipOrder.processOrder(orderTotal);
        int vipPoints = vipOrder.calculateLoyaltyPoints(vipPolicy);

        System.out.printf("VIP Order     -> Payable: $%.2f | Earned Points: %d%n", vipFinal, vipPoints);
    }
}

```

### Output

```text
=== Processing Orders with Modern OCP ===

Regular Order -> Payable: $900.00 | Earned Points: 10
Premium Order -> Payable: $800.00 | Earned Points: 25
VIP Order     -> Payable: $700.00 | Earned Points: 50

```

---

## Summary: Unsealed vs. Sealed OCP

| Feature | Classic OCP (Unsealed) | Modern OCP (Sealed Classes) |
| --- | --- | --- |
| **Who can extend?** | Anyone, anywhere in the classpath. | Only explicitly permitted domain classes. |
| **Switch statements** | Requires `default` fallback (hides missing cases). | Exhaustive check at compile time (fails fast if a type is missed). |
| **Domain Safety** | Low — vulnerable to unexpected third-party subclasses. | High — domain boundaries are strictly guarded. |
| **OCP Verdict** | Open to the entire world. | **Controlled extension** — open to planned domain additions, closed to external corruption. |