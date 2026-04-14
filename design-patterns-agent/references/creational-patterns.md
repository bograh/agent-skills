# Creational Patterns Reference
Source: refactoring.guru/design-patterns/creational-patterns

---

## Factory Method

**Intent**: Define an interface for creating an object, but let subclasses decide which class to instantiate.

**When to use**: The exact class to create isn't known until runtime; you want subclasses to control creation.

**Participants**: `Creator` · `ConcreteCreator` · `Product` · `ConcreteProduct`

```java
// Product interface
interface Notification {
    void send(String message);
}

// Concrete Products
class EmailNotification implements Notification {
    private final String email;
    EmailNotification(String email) { this.email = email; }
    @Override public void send(String message) {
        System.out.println("Email to " + email + ": " + message);
    }
}

class SMSNotification implements Notification {
    private final String phone;
    SMSNotification(String phone) { this.phone = phone; }
    @Override public void send(String message) {
        System.out.println("SMS to " + phone + ": " + message);
    }
}

// Creator — declares the factory method
abstract class NotificationService {
    public void notifyUser(String message) {
        Notification n = createNotification(); // factory method
        n.send(message);
    }
    protected abstract Notification createNotification(); // subclasses decide
}

// Concrete Creators
class EmailService extends NotificationService {
    private final String email;
    EmailService(String email) { this.email = email; }
    @Override protected Notification createNotification() {
        return new EmailNotification(email);
    }
}

class SMSService extends NotificationService {
    private final String phone;
    SMSService(String phone) { this.phone = phone; }
    @Override protected Notification createNotification() {
        return new SMSNotification(phone);
    }
}

// Usage
NotificationService service = new EmailService("user@example.com");
service.notifyUser("Your order has shipped!");
```

**Pitfalls**: Can lead to parallel class hierarchies. If you have multiple product families, use Abstract Factory instead.

---

## Abstract Factory

**Intent**: Create families of related objects without specifying their concrete classes.

**When to use**: Code must work with various families of related products (e.g., cross-platform UI, multiple DB vendors).

**Participants**: `AbstractFactory` · `ConcreteFactory` · `AbstractProduct` · `ConcreteProduct` · `Client`

```java
// Abstract products
interface Button    { void render(); void onClick(); }
interface Checkbox  { void render(); boolean isChecked(); }

// Windows family
class WindowsButton implements Button {
    @Override public void render()   { System.out.println("Render Windows button"); }
    @Override public void onClick()  { System.out.println("Windows button click"); }
}
class WindowsCheckbox implements Checkbox {
    private boolean checked;
    @Override public void render()        { System.out.println("Render Windows checkbox"); }
    @Override public boolean isChecked()  { return checked; }
}

// Mac family
class MacButton implements Button {
    @Override public void render()   { System.out.println("Render Mac button"); }
    @Override public void onClick()  { System.out.println("Mac button click"); }
}
class MacCheckbox implements Checkbox {
    private boolean checked;
    @Override public void render()        { System.out.println("Render Mac checkbox"); }
    @Override public boolean isChecked()  { return checked; }
}

// Abstract factory
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete factories
class WindowsUIFactory implements UIFactory {
    @Override public Button   createButton()   { return new WindowsButton(); }
    @Override public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}
class MacUIFactory implements UIFactory {
    @Override public Button   createButton()   { return new MacButton(); }
    @Override public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client
class Application {
    private final Button button;
    private final Checkbox checkbox;

    Application(UIFactory factory) {
        button   = factory.createButton();
        checkbox = factory.createCheckbox();
    }
    void render() { button.render(); checkbox.render(); }
}

// Usage — swap entire family at runtime
UIFactory factory = isWindows() ? new WindowsUIFactory() : new MacUIFactory();
Application app = new Application(factory);
app.render();
```

**Pitfalls**: Adding a new product type (e.g., `Slider`) requires changing ALL factory interfaces and all implementations.

---

## Builder

**Intent**: Construct complex objects step by step. Allows different representations from the same construction process.

**When to use**: Many optional parameters (telescoping constructor anti-pattern); multi-step construction; need different object variants.

```java
// Product
class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder b) {
        this.method    = b.method;
        this.url       = b.url;
        this.headers   = Collections.unmodifiableMap(b.headers);
        this.body      = b.body;
        this.timeoutMs = b.timeoutMs;
    }

    // Builder (static nested class)
    public static class Builder {
        private final String method;   // required
        private final String url;      // required
        private Map<String, String> headers = new HashMap<>();
        private String body      = "";
        private int    timeoutMs = 5000;

        public Builder(String method, String url) {
            this.method = method;
            this.url    = url;
        }
        public Builder header(String key, String value) {
            headers.put(key, value); return this;
        }
        public Builder body(String body)         { this.body = body; return this; }
        public Builder timeout(int ms)           { this.timeoutMs = ms; return this; }
        public HttpRequest build()               { return new HttpRequest(this); }
    }
}

// Usage — fluent API; no ambiguous constructor overloads
HttpRequest req = new HttpRequest.Builder("POST", "https://api.example.com/orders")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"item\": \"book\"}")
    .timeout(3000)
    .build();
```

**Pitfalls**: Overkill for simple objects. Builder is stateful — don't reuse a Builder instance after calling `build()`.

---

## Prototype

**Intent**: Create new objects by copying existing ones (cloning), without coupling to their concrete class.

**When to use**: Object creation is expensive (DB lookup, network call) and a clone is cheaper; concrete classes are known only at runtime.

```java
// Prototype interface
interface Shape extends Cloneable {
    Shape clone();
    void draw();
}

// Concrete prototypes
class Circle implements Shape {
    private int x, y, radius;
    private String color;

    Circle(int x, int y, int radius, String color) {
        this.x = x; this.y = y; this.radius = radius; this.color = color;
    }

    // Copy constructor used by clone
    Circle(Circle source) {
        this.x = source.x; this.y = source.y;
        this.radius = source.radius; this.color = source.color;
    }

    @Override public Circle clone() { return new Circle(this); }
    @Override public void draw()    { System.out.println("Circle at (" + x + "," + y + ")"); }
    public void setColor(String c)  { this.color = c; }
}

// Prototype registry
class ShapeCache {
    private final Map<String, Shape> cache = new HashMap<>();

    void register(String key, Shape shape) { cache.put(key, shape); }

    Shape get(String key) {
        Shape prototype = cache.get(key);
        if (prototype == null) throw new IllegalArgumentException("Unknown shape: " + key);
        return prototype.clone(); // return a fresh copy each time
    }
}

// Usage
ShapeCache cache = new ShapeCache();
cache.register("red-circle", new Circle(0, 0, 10, "red"));

Circle c1 = (Circle) cache.get("red-circle");
Circle c2 = (Circle) cache.get("red-circle");
c2.setColor("blue"); // c1 is unaffected
```

**Pitfalls**: Deep vs. shallow copy — be explicit. Watch for objects with non-cloneable fields (open streams, locks). Prefer copy constructors over `Object.clone()` in Java.

---

## Singleton

**Intent**: Ensure a class has only one instance and provide a global access point to it.

**When to use** (sparingly): Logger, config reader, thread pool, connection pool. Prefer dependency injection when possible.

**When NOT to use**: Just to get a convenient global; in unit-tested code (singletons resist mocking); when "single" is an assumption that may change.

```java
// Thread-safe lazy initialization (Bill Pugh / Initialization-on-demand holder)
public class AppConfig {
    private final Properties props;

    private AppConfig() {
        props = new Properties();
        try (InputStream in = getClass().getResourceAsStream("/app.properties")) {
            props.load(in);
        } catch (IOException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    // Holder is loaded only when getInstance() is first called — thread-safe without sync
    private static class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }

    public static AppConfig getInstance() { return Holder.INSTANCE; }

    public String get(String key) { return props.getProperty(key); }
}

// Usage
String dbUrl = AppConfig.getInstance().get("db.url");
```

**Alternative — enum Singleton** (simplest, serialization-safe):
```java
public enum AppLogger {
    INSTANCE;

    private final Logger logger = LoggerFactory.getLogger(AppLogger.class);
    public void log(String message) { logger.info(message); }
}

// Usage
AppLogger.INSTANCE.log("Application started");
```

**Pitfalls**: Hidden global state; hard to unit test (use dependency injection instead); be careful with classloaders in app servers; enum form is preferred over double-checked locking in Java.
