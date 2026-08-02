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