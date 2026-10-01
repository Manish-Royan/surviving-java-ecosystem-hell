# 📦 Spring Container Overview

<img width="1200" height="896" alt="Spring-Container" src="https://github.com/user-attachments/assets/a81d095a-14f6-4636-94be-eb376357ccc2" />

➡️ Before creating our first program, let's have a quick glimpse about **Spring Container**.

---
## **1. What is the Spring Container?**
➡️ The **Spring Container** is the **core engine** of the Spring Framework.  
➡️ It is responsible for:
- Creating and managing **beans** (objects).
- Handling **dependency injection (DI)**.
- Managing the **bean lifecycle** (creation → initialization → destruction).
- Providing **configuration and context** for the application.

### 💭 Think of the container as the **“brain” of Spring**—it knows how to assemble and manage all the parts of your application.

---

## **2. Responsibilities of the Spring Container**
➡️ The container does much more than just object creation:
- **Instantiate beans** → Creates objects defined in configuration.
- **Configure beans** → Injects dependencies (DI).
- **Manage lifecycle** → Calls initialization and destruction callbacks.
- **Provide context** → Supplies environment info, resources, and events.
- **Enable modularity** → Supports profiles, scopes, and lazy initialization.
- **Support AOP & Events** → Allows cross-cutting concerns and event-driven programming.

> ### 👉 In short: The container is both a **factory** and a **manager** for your application objects.

---

## **3. Types of Spring Containers**
Spring provides **two main types of containers**:

### 1. **BeanFactory**
- The simplest container.
- Provides basic DI functionality.
- Lazily initializes beans (created only when requested).
- Lightweight, suitable for simple applications.

> ### ⚠️ Using `BeanFactory` directly may cause missing features like event handling and annotation support.

> ### ⚠️ Rarely used directly in modern apps.

---

### 2. **ApplicationContext**
- A **superset of BeanFactory**.
- Provides advanced features:
  - Event propagation.
  - Internationalization (i18n).
  - Annotation-based configuration.
  - AOP integration.
- Eagerly initializes beans (created at startup).
- Common implementations:
  - `ClassPathXmlApplicationContext`
  - `FileSystemXmlApplicationContext`
  - `AnnotationConfigApplicationContext`

* **Fun Fact 💡**: `ApplicationContext` internally uses `BeanFactory`—so you can think of it as **BeanFactory + extra features**.
> ### 👉 Almost all modern Spring applications use **ApplicationContext**.

---
