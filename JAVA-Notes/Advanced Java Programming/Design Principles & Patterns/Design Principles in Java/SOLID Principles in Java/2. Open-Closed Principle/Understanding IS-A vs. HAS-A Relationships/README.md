# Understanding IS-A vs. HAS-A Relationships

### 1. IS-A Relationship (Inheritance / Realization)
An **Is-A** relationship represents **Inheritance**. It means that a child class is a specialized type of a parent class. In Java, you implement an **Is-A** relationship using the `extends` keyword (for classes) or `implements` (for interfaces). It expresses classification and hierarchy.

* Real World: A `Dog` IS-A `Animal`. 
```java
// Parent Class
class Animal {
    void eat() {
        System.out.println("This animal eats food.");
    }
}

// Dog IS-A Animal
class Dog extends Animal {
    void bark() {
        System.out.println("The dog barks.");
    }
}

public class IsADemo {
    public static void main(String[] args) {
        Dog myDog = new Dog();
        
        // Inherited method from Animal class
        myDog.eat();  
        
        // Specific method in Dog class
        myDog.bark(); 
    }
}
```
---
### 2. HAS-A Relationship (Composition / Association)
A **Has-A** relationship represents Composition or Aggregation. It means that a class holds a reference to an instance of another class as a member variable. Instead of inheriting properties, it simply uses another object to do work.

* Real World: A `Car` HAS-A `Engine`
```java
// Independent Engine Class
class Engine {
    void start() {
        System.out.println("Engine starts running...");
    }
}

// Car HAS-A Engine
class Car {
    // Member field holding an Engine object
    private Engine engine = new Engine(); 

    void startCar() {
        // Delegating the work to the Engine object
        engine.start(); 
        System.out.println("Car is ready to drive.");
    }
}

public class HasADemo {
    public static void main(String[] args) {
        Car myCar = new Car();
        myCar.startCar();
    }
}
```
---
### ☑️ Summary Checklist

| Relationship | Concept | Java Keyword | Key Test |
| --- | --- | --- | --- |
| **Is-A** | Inheritance | `extends` / `implements` | "Is [Class A] a type of [Class B]?" |
| **Has-A** | Composition | Field instance (`new`) | "Does [Class A] contain/use [Class B]?" |

