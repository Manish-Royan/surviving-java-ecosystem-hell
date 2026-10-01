## Why HAS-A (Composition) Powers OCP?

Now let's examine why OCP relies so heavily on **HAS-A (Composition)** rather than relying purely on **IS-A (Inheritance)**.

---

### **Approach 1: Trying to achieve OCP using pure Inheritance (IS-A)**

Imagine trying to handle different discounts by making `DiscountCalculator` a base class and creating subclasses for every discount type:

```java
// Base class
public class DiscountCalculator {
    public double calculate(double amount) {
        return 0; // Default behavior
    }
}

// Subclassing via IS-A
public class RegularDiscountCalculator extends DiscountCalculator {
    @Override
    public double calculate(double amount) {
        return amount * 0.1;
    }
}

public class PremiumDiscountCalculator extends DiscountCalculator {
    @Override
    public double calculate(double amount) {
        return amount * 0.2;
    }
}

```

#### Why this fails scale & flexibility tests:

1. **Tight Coupling:** The child classes are tightly coupled to the base `DiscountCalculator` implementation. If `DiscountCalculator` needs extra calculation steps (e.g., logging, tax computation, rounding rules), we either duplicate code across every subclass or bake it into the base class.
2. **Rigid Runtime Behavior:** An instance of `RegularDiscountCalculator` is stuck being a regular discount calculator forever. We cannot swap rules dynamically at runtime without instantiating entirely different calculator classes.
3. **Class Explosion:** If our calculator needs to handle multiple varying behaviors (e.g., Discount Type + Tax Rule + Currency Converter), inheritance forces us to create classes like `RegularDiscountVatTaxUSDCalculator`, leading to an unmanageable class hierarchy.

---

### **Approach 2: OCP using Composition (HAS-A)**

In our refactored code, we separated the **Calculator** (the context/runner) from the **Policy** (the strategy/behavior):

```java
public class DiscountCalculator {
    // Composition: DiscountCalculator HAS-A DiscountPolicy
    private DiscountPolicy discountPolicy;

    public DiscountCalculator(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public double calculate(double amount) {
        // Delegation
        return discountPolicy.applyDiscount(amount);
    }
}

```

#### Why Composition works best for OCP:

1. **Dynamic Extension:** We can swap strategies on the fly without changing the calculator instance:
```java
DiscountCalculator calculator = new DiscountCalculator(new RegularDiscount());
// Need to change policy dynamically?
calculator.setDiscountPolicy(new PremiumDiscount()); 

```


2. **Single Responsibility & True OCP:** `DiscountCalculator` manages **how & when** to trigger calculations (and could handle tax, logging, validation). `DiscountPolicy` implementations handle **what percentage** applies.
3. **Zero Modification Required:** To add a `VIPDiscount`, you create one small class implementing `DiscountPolicy` (an **IS-A** relationship between the policy and interface). The `DiscountCalculator` (which **HAS-A** policy) remains completely untouched and closed for modification.

---

## Summary Comparison

| Metric | IS-A (Inheritance-based OCP) | HAS-A (Composition-based OCP) |
| --- | --- | --- |
| **Relationship** | Calculator *is a* specific discount. | Calculator *has a* discount rule inside it. |
| **Flexibility** | Rigid — decided at compile time. | Flexible — swap policies easily at runtime. |
| **Coupling** | High — subclasses depend on parent implementation details. | Low — decoupled via interface abstraction. |
| **Code Modification** | Risk of breaking parent/child class hierarchies. | Zero risk to context/calculator code. |

---