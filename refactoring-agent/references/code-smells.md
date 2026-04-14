# Code Smells Reference
Source: refactoring.guru/refactoring/smells

## Table of Contents
1. [Bloaters](#bloaters)
2. [Object-Orientation Abusers](#oo-abusers)
3. [Change Preventers](#change-preventers)
4. [Dispensables](#dispensables)
5. [Couplers](#couplers)

---

## Bloaters

Code, methods, and classes that grow so big they become hard to work with.

### Long Method
**What**: A method contains too many lines (rule of thumb: >10 lines warrants scrutiny).
**Why it hurts**: Hard to understand, test, and reuse.
**Fix**: Extract Method repeatedly. If local variables make extraction hard, use Replace Method with Method Object.

```java
// Smell — one method doing far too much
void processOrder(Order order) {
    // validate (10 lines)
    // calculate price (15 lines)
    // apply discounts (20 lines)
    // send confirmation email (10 lines)
    // update inventory (12 lines)
}

// Fix — extract each concern
void processOrder(Order order) {
    validateOrder(order);
    double price = calculatePrice(order);
    price = applyDiscounts(order, price);
    sendConfirmation(order, price);
    updateInventory(order);
}
```

### Large Class
**What**: A class contains too many fields, methods, or lines of code.
**Why it hurts**: Violates Single Responsibility Principle; hard to maintain.
**Fix**: Extract Class, Extract Subclass, or Extract Interface.

```java
// Smell — UserService does everything
class UserService {
    void register(User u) { ... }
    void login(User u) { ... }
    void sendPasswordResetEmail(User u) { ... }
    void formatUserReport(User u) { ... }
    void exportUserToCsv(User u) { ... }
}

// Fix — split by responsibility
class UserAuthService   { void register(User u); void login(User u); }
class UserEmailService  { void sendPasswordResetEmail(User u); }
class UserReportService { String formatReport(User u); String exportCsv(User u); }
```

### Primitive Obsession
**What**: Using primitives (int, String, double) for domain concepts like money, phone numbers, or coordinates.
**Why it hurts**: Validation logic scattered everywhere; no type safety; duplication.
**Fix**: Replace Data Value with Object, Introduce Parameter Object.

```java
// Smell
class Order {
    private String customerPhone;   // no validation, no formatting
    private double price;           // no currency, no precision rules
    private String status;          // just a string — could be anything
}

// Fix
class PhoneNumber {
    private final String number;
    PhoneNumber(String number) {
        if (!number.matches("\\+?[0-9]{10,15}")) throw new IllegalArgumentException(...);
        this.number = number;
    }
}
class Money {
    private final BigDecimal amount;
    private final Currency currency;
}
enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
```

### Long Parameter List
**What**: More than 3–4 parameters in a method.
**Why it hurts**: Hard to call correctly; parameter order matters; fragile.
**Fix**: Introduce Parameter Object, Preserve Whole Object.

```java
// Smell
Booking createBooking(String guestName, String guestEmail,
                      Date checkIn, Date checkOut,
                      int adults, int children, String roomType) { ... }

// Fix
record GuestInfo(String name, String email) {}
record StayDetails(Date checkIn, Date checkOut, int adults, int children) {}

Booking createBooking(GuestInfo guest, StayDetails stay, String roomType) { ... }
```

### Data Clumps
**What**: The same group of variables appears together in multiple places.
**Why it hurts**: Changes must be made in many places; scattered duplication.
**Fix**: Extract Class to bundle them.

```java
// Smell — street/city/zip always appear together
void printInvoice(String street, String city, String zip) { ... }
void validateAddress(String street, String city, String zip) { ... }
void shipOrder(Order order, String street, String city, String zip) { ... }

// Fix
class Address {
    final String street, city, zip;
    boolean isValid() { ... }
    String format() { ... }
}
void printInvoice(Address address) { ... }
```

---

## Object-Orientation Abusers

### Switch Statements
**What**: Complex switch/if-else chains, especially on type codes.
**Why it hurts**: Must update every switch when adding a new type (OCP violation).
**Fix**: Replace Conditional with Polymorphism.

```java
// Smell
double getShippingCost(String type) {
    return switch (type) {
        case "STANDARD" -> 5.0;
        case "EXPRESS"  -> 15.0;
        case "OVERNIGHT"-> 25.0;
        default -> throw new IllegalArgumentException(type);
    };
}

// Fix — add new types without touching existing code
interface ShippingMethod {
    double getCost();
}
class StandardShipping  implements ShippingMethod { public double getCost() { return 5.0; } }
class ExpressShipping   implements ShippingMethod { public double getCost() { return 15.0; } }
class OvernightShipping implements ShippingMethod { public double getCost() { return 25.0; } }
```

### Temporary Field
**What**: An instance field that is only set and used in certain circumstances.
**Why it hurts**: Object can be in undefined/null state; confusing.
**Fix**: Extract Class for the field and related behavior.

```java
// Smell — weightForShipping only valid after calculateShipping() is called
class Order {
    private double weightForShipping; // only set during shipping calc — confusing!
    void calculateShipping() {
        weightForShipping = items.stream().mapToDouble(Item::getWeight).sum();
        shippingCost = carrier.quote(weightForShipping);
    }
}

// Fix — extract into a dedicated ShippingCalculator class
class ShippingCalculator {
    private final List<Item> items;
    ShippingCalculator(List<Item> items) { this.items = items; }
    double calculate(Carrier carrier) {
        double weight = items.stream().mapToDouble(Item::getWeight).sum();
        return carrier.quote(weight);
    }
}
```

### Refused Bequest
**What**: A subclass inherits methods or data it doesn't need or want.
**Why it hurts**: Wrong inheritance hierarchy; Liskov Substitution Principle violation.
**Fix**: Replace Inheritance with Delegation if the subclass doesn't want the parent interface.

```java
// Smell — Chair doesn't need most Animal behavior
class Animal {
    void eat() { ... }
    void sleep() { ... }
    void breathe() { ... }
}
class RubberDuck extends Animal { // refuses eat/sleep/breathe
    @Override void eat()    { /* does nothing */ }
    @Override void sleep()  { /* does nothing */ }
}

// Fix — use interface + delegation instead
interface Quackable { void quack(); }
class RubberDuck implements Quackable {
    @Override public void quack() { System.out.println("Squeak!"); }
}
```

---

## Change Preventers

### Divergent Change
**What**: One class is changed for many different reasons (adding a product type AND changing the DB schema both require editing the same class).
**Why it hurts**: Violates SRP; hard to identify what changed and why.
**Fix**: Extract Class to separate each responsibility.

```java
// Smell — OrderProcessor handles both business rules AND persistence AND notifications
class OrderProcessor {
    void process(Order order) {
        validateBusinessRules(order);  // business logic
        saveToDatabase(order);         // persistence
        sendConfirmationEmail(order);  // notification
    }
}

// Fix — each class has one reason to change
class OrderValidator  { void validate(Order order) { ... } }
class OrderRepository { void save(Order order)    { ... } }
class OrderNotifier   { void notify(Order order)  { ... } }
```

### Shotgun Surgery
**What**: Making any modification requires many small changes in many different classes.
**Why it hurts**: Easy to miss a spot; risky changes.
**Fix**: Move Method and Move Field to consolidate related behavior.

```java
// Smell — changing tax calculation requires editing 5 classes
class Cart    { double getTax() { return total * 0.1; } }
class Invoice { double getTax() { return amount * 0.1; } }
class Receipt { double getTax() { return subtotal * 0.1; } }

// Fix — centralize
class TaxCalculator {
    static double calculate(double amount) { return amount * TAX_RATE; }
    private static final double TAX_RATE = 0.1;
}
```

### Parallel Inheritance Hierarchies
**What**: Every time you add a subclass to one hierarchy, you must add one to another.
**Fix**: Make instances of one hierarchy refer to instances of the other; eliminate the duplication.

```java
// Smell — adding SalesEmployee requires adding SalesEmployeeReport too
class Employee { ... }
class Engineer extends Employee { ... }
class SalesEmployee extends Employee { ... }

class EmployeeReport { ... }
class EngineerReport extends EmployeeReport { ... }
class SalesEmployeeReport extends EmployeeReport { ... }  // forced parallel

// Fix — decouple: Report takes Employee as a parameter
class EmployeeReport {
    EmployeeReport(Employee employee) { ... }
    // adapts to the employee type internally
}
```

---

## Dispensables

### Comments
**What**: A method is full of comments explaining *what* the code does (not *why*).
**Why it hurts**: Usually signals unclear code; comments lie when code changes but comments don't.
**Fix**: Extract Method with a descriptive name replaces most comments.

```java
// Smell
// Check if customer is eligible for premium discount
if (customer.getAge() >= 65 || customer.getYearsActive() >= 10
        || customer.getTotalPurchases() > 10000) { ... }

// Fix — the name is the comment
if (isEligibleForPremiumDiscount(customer)) { ... }

private boolean isEligibleForPremiumDiscount(Customer customer) {
    return customer.getAge() >= 65
        || customer.getYearsActive() >= 10
        || customer.getTotalPurchases() > 10000;
}
```

### Duplicate Code
**What**: The same or very similar code exists in more than one place.
**Fix**: Extract Method and call from both places. Pull Up Method if duplicates are in sibling classes.

```java
// Smell — same validation in two places
class SavingsAccount {
    void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
    }
}
class CheckingAccount {
    void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive"); // dup
        balance += amount;
    }
}

// Fix — Pull Up Method to common parent
abstract class Account {
    protected void validateAmount(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
    }
}
class SavingsAccount extends Account {
    void deposit(double amount) { validateAmount(amount); balance += amount; }
}
```

### Lazy Class
**What**: A class doesn't do enough to justify its existence.
**Fix**: Inline Class to merge it into its caller.

```java
// Smell — TemperatureConverter just wraps a one-liner
class TemperatureConverter {
    double celsiusToFahrenheit(double c) { return c * 9/5 + 32; }
}

// Fix — inline into the caller or use a static utility method directly
// No separate class needed; just call the formula where required.
```

### Data Class
**What**: A class has only fields, getters, and setters — no behavior.
**Why it hurts**: Behavior that should be here is scattered in other classes.
**Fix**: Move methods that work with this data into the class.

```java
// Smell — all behavior about Customer lives in CustomerService
class Customer {
    private String firstName;
    private String lastName;
    private String email;
    // getters/setters only
}
class CustomerService {
    String getFullName(Customer c) { return c.getFirstName() + " " + c.getLastName(); }
    boolean isValidEmail(Customer c) { return c.getEmail().contains("@"); }
}

// Fix — move behavior where data lives
class Customer {
    private String firstName, lastName, email;
    String getFullName() { return firstName + " " + lastName; }
    boolean hasValidEmail() { return email != null && email.contains("@"); }
}
```

### Dead Code
**What**: Variables, parameters, fields, methods, or classes that are no longer used.
**Fix**: Delete it. Use your IDE's "Find Usages" feature.

```java
// Smell
class ReportService {
    private Logger legacyLogger; // never used since migration

    @Deprecated
    String generateOldFormat(Report r) { ... } // never called
}
```

### Speculative Generality
**What**: Code written "just in case" for future needs that never materialized.
**Fix**: Remove unused parameters, collapse empty abstract classes, delete unused hooks.

```java
// Smell — AbstractSingleItemProcessor exists only for a subclass that could be concrete
abstract class AbstractSingleItemProcessor<T> {
    abstract T process(T input);
    // no other subclasses exist or are planned
}

// Fix — just make it a concrete class or a static method
```

---

## Couplers

### Feature Envy
**What**: A method in class A spends most of its time accessing data from class B.
**Why it hurts**: The method belongs in B, not A.
**Fix**: Move Method to class B.

```java
// Smell — calculateRent() is more interested in Apartment than in Tenant
class Tenant {
    double calculateRent() {
        return apartment.getBaseRent()
             * apartment.getLocation().getPriceMultiplier()
             + apartment.getUtilitiesCost();
    }
}

// Fix
class Apartment {
    double calculateRent() {
        return baseRent * location.getPriceMultiplier() + utilitiesCost;
    }
}
class Tenant {
    double calculateRent() { return apartment.calculateRent(); }
}
```

### Inappropriate Intimacy
**What**: Class A digs into the private parts of class B constantly.
**Fix**: Move Method and Move Field. Extract Class if there's overlapping behavior.

```java
// Smell — Client reaches into Order internals
class InvoiceService {
    void generateInvoice(Order order) {
        String customerEmail = order.customer.email;    // direct field access
        List<Item> items = order.lineItems;             // direct field access
        double tax = order.taxRate * order.subtotal;    // direct field access
    }
}

// Fix — Order exposes what it knows; InvoiceService only calls public API
class Order {
    String getCustomerEmail() { return customer.getEmail(); }
    double getTotalWithTax()  { return subtotal * (1 + taxRate); }
    List<Item> getLineItems() { return Collections.unmodifiableList(lineItems); }
}
```

### Message Chains
**What**: `a.getB().getC().getD().doSomething()` — long navigation chains.
**Why it hurts**: Client depends on the entire structure; brittle.
**Fix**: Hide Delegate — add a shortcut method.

```java
// Smell
String city = order.getCustomer().getAddress().getCity();

// Fix — add a shortcut on Order
class Order {
    String getCustomerCity() { return customer.getAddress().getCity(); }
}
String city = order.getCustomerCity();
```

### Middle Man
**What**: A class whose methods mostly just delegate to another class.
**Fix**: Remove Middle Man — let the client call the delegate directly.

```java
// Smell — Manager does nothing but forward to Department
class Manager {
    Department department;
    String getDeptName()    { return department.getName(); }
    List<Employee> getStaff() { return department.getStaff(); }
    double getBudget()      { return department.getBudget(); }
    // ... 10 more thin wrappers
}

// Fix
Department dept = manager.getDepartment();
String name = dept.getName();
```
