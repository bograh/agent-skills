---
name: design-patterns-agent
description: >
  Expert design patterns agent based on refactoring.guru methodology covering all 23 GoF patterns
  (Creational, Structural, Behavioral). Use this skill whenever a user wants to apply, identify,
  explain, or choose a design pattern — even if they don't say "design pattern" explicitly.
  Triggers include: "how should I structure this", "what's the best way to architect this",
  "I need objects that can be created flexibly", "I want to decouple these classes",
  "how do I add behavior without modifying this class", "I need a plugin system",
  "how do I handle multiple algorithms", "what pattern fits here", "should I use strategy or
  state here", "help me design this component", code reviews asking about architecture quality,
  and any time the user is designing a new system or component. Also trigger when the user
  describes a problem that clearly maps to a pattern (e.g., "I need one instance only" → Singleton,
  "I want to notify many objects when state changes" → Observer).
---

# Design Patterns Agent

A structured agent for identifying, explaining, and implementing all 23 GoF design patterns
using the refactoring.guru taxonomy.

## Agent Workflow

### For "What pattern should I use?" questions:
1. Ask the user to describe their problem (if not already stated).
2. Map the problem to a pattern category using the **Problem → Pattern** decision guide below.
3. Recommend 1–2 patterns with a brief rationale.
4. Optionally compare trade-offs if multiple patterns fit.
5. Offer to show an implementation in the user's language.

### For "Explain this pattern" questions:
1. Give the **intent** in 1–2 sentences.
2. Describe the **problem it solves** (the "when to use" case).
3. Show the **structure** (roles/participants).
4. Provide a **code example** in the user's language.
5. Note **trade-offs and when NOT to use it**.

### For "Implement this pattern" questions:
1. Confirm the language/framework.
2. Write clean, idiomatic code with proper naming.
3. Label each participant (Creator, ConcreteProduct, etc.).
4. Show a usage example.
5. Flag common mistakes or pitfalls.

---

## Problem → Pattern Decision Guide

### Object Creation Problems
| Problem | Pattern |
|---|---|
| Need one global instance | Singleton |
| Create objects without specifying concrete class | Factory Method |
| Create families of related objects | Abstract Factory |
| Build complex objects step by step | Builder |
| Copy existing objects without coupling to their class | Prototype |

### Structural / Organization Problems
| Problem | Pattern |
|---|---|
| Incompatible interfaces need to work together | Adapter |
| Decouple abstraction from implementation | Bridge |
| Treat individual objects and compositions uniformly | Composite |
| Add responsibilities without subclassing | Decorator |
| Simplify a complex subsystem | Facade |
| Share common state among many fine-grained objects | Flyweight |
| Control access to an object | Proxy |

### Behavioral / Algorithm Problems
| Problem | Pattern |
|---|---|
| Pass a request through a chain of handlers | Chain of Responsibility |
| Encapsulate a request as an object | Command |
| Traverse a collection without exposing internals | Iterator |
| Reduce chaotic dependencies between objects | Mediator |
| Save and restore object state | Memento |
| Notify many objects when state changes | Observer |
| Object changes behavior based on internal state | State |
| Swap algorithms at runtime | Strategy |
| Define skeleton of algorithm; let subclasses fill steps | Template Method |
| Add operations to objects without changing their classes | Visitor |

---

## Pattern Quick Reference

### Creational Patterns

**Factory Method** — Defines an interface for creating objects, but lets subclasses decide which class to instantiate. Use when the exact type isn't known until runtime or you want subclasses to control creation.

**Abstract Factory** — Creates families of related objects without specifying their concrete classes. Use when your code needs to work with various families of related products.

**Builder** — Separates construction of a complex object from its representation. Use when constructing complex objects step-by-step, or when you need different representations from the same process.

**Prototype** — Creates new objects by copying existing ones. Use when object creation is expensive and a clone is cheaper, or when classes to instantiate are only known at runtime.

**Singleton** — Ensures a class has only one instance and provides a global access point. Use sparingly — prefer dependency injection. Appropriate for logging, config, thread pools.

---

### Structural Patterns

**Adapter** — Wraps an incompatible object in an adapter that translates its interface. Two variants: class adapter (via inheritance) and object adapter (via composition, preferred).

**Bridge** — Splits a large class into abstraction and implementation hierarchies that can evolve independently. Use when you want to avoid an explosion of subclasses from combining two dimensions.

**Composite** — Composes objects into tree structures to represent part-whole hierarchies. Clients treat individual objects and compositions uniformly. Use for file systems, UI components, org charts.

**Decorator** — Attaches additional responsibilities dynamically by wrapping objects. Preferred over subclassing for adding behavior. Stackable.

**Facade** — Provides a simplified interface to a complex subsystem. Doesn't add new behavior — just simplifies the existing interface for common use cases.

**Flyweight** — Shares intrinsic state among many objects to reduce memory. Use when you need a huge number of similar objects. Separate intrinsic (shared) from extrinsic (unique) state.

**Proxy** — Provides a placeholder for another object to control access. Types: virtual (lazy init), remote (local rep of remote), protection (access control), logging, caching.

---

### Behavioral Patterns

**Chain of Responsibility** — Passes a request along a chain; each handler decides to process or pass. Use for event handling, middleware pipelines, request filters.

**Command** — Encapsulates a request as an object with all needed info. Enables: undo/redo, queuing, logging, transactional behavior.

**Iterator** — Provides sequential access to collection elements without exposing internal structure. Use for custom collections or when you need multiple simultaneous traversals.

**Mediator** — Centralizes complex communication between objects. Objects communicate only through the mediator. Use to reduce spaghetti object relationships (e.g., chat rooms, air traffic control, UI form logic).

**Memento** — Captures and restores object state without exposing internals. Use for undo/redo, snapshots, checkpointing.

**Observer** — Defines a one-to-many dependency so when one object changes state, dependents are notified automatically. Foundation of event systems, MVC, reactive programming.

**State** — Allows an object to alter its behavior when its internal state changes. Appears to change class. Use when behavior depends on state and must change at runtime with large state-specific code blocks.

**Strategy** — Defines a family of algorithms, encapsulates each, and makes them interchangeable. Use when you want to switch algorithms at runtime or eliminate large conditionals over algorithm selection.

**Template Method** — Defines the skeleton of an algorithm in a base class, deferring specific steps to subclasses. Use when steps are fixed but their implementation varies. Prefer Strategy when you want runtime flexibility.

**Visitor** — Adds operations to object structures without modifying them. Use when you need to perform many distinct/unrelated operations on an object structure without polluting it.

---

## Common Pattern Confusions

| "Should I use X or Y?" | Guidance |
|---|---|
| Strategy vs State | Strategy: swap algorithm from outside. State: object changes behavior based on internal state. |
| Factory Method vs Abstract Factory | Factory Method: one product, subclass decides. Abstract Factory: families of products. |
| Decorator vs Proxy | Decorator: adds behavior, same interface. Proxy: controls access, same interface. |
| Adapter vs Facade | Adapter: makes two incompatible things work together. Facade: simplifies one complex thing. |
| Command vs Strategy | Command: encapsulates *a request* (what to do + when). Strategy: encapsulates *an algorithm* (how to do it). |
| Composite vs Decorator | Composite: tree structures, same treatment. Decorator: adds behavior via wrapping. |
| Observer vs Mediator | Observer: broadcast, direct dependency. Mediator: centralized hub, no direct dependencies. |

---

## Reference Files

- `references/creational-patterns.md` — Detailed structure, participants, and code patterns for Singleton, Factory Method, Abstract Factory, Builder, Prototype
- `references/structural-patterns.md` — Detailed structure for Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
- `references/behavioral-patterns.md` — Detailed structure for Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor

Load the relevant reference file when you need precise class diagrams, participant names, or implementation recipes.
