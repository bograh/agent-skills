# Refactoring Techniques Reference
Source: refactoring.guru/refactoring/techniques

## Table of Contents
1. [Composing Methods](#composing-methods)
2. [Moving Features Between Objects](#moving-features)
3. [Organizing Data](#organizing-data)
4. [Simplifying Conditional Expressions](#simplifying-conditionals)
5. [Simplifying Method Calls](#simplifying-method-calls)
6. [Dealing with Generalization](#generalization)

---

## Composing Methods

### Extract Method
**Problem**: A code fragment can be grouped and named.
**Recipe**: Create a new method named for *what* it does. Move the fragment. Replace with a call.

```java
// Before
void printOrder(Order order) {
    System.out.println("Order: " + order.getId());
    double total = 0;
    for (OrderLine line : order.getLines()) {
        total += line.getPrice() * line.getQuantity();
    }
    System.out.println("Total: " + total);
}

// After
void printOrder(Order order) {
    System.out.println("Order: " + order.getId());
    System.out.println("Total: " + calculateTotal(order));
}

private double calculateTotal(Order order) {
    double total = 0;
    for (OrderLine line : order.getLines()) {
        total += line.getPrice() * line.getQuantity();
    }
    return total;
}
```

### Inline Method
**Problem**: A method body is as clear as its name; the indirection adds no value.
**Recipe**: Replace each call with the method body. Delete the method.

```java
// Before
int getRating(Driver driver) {
    return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}
boolean moreThanFiveLateDeliveries(Driver driver) {
    return driver.getLateDeliveries() > 5;
}

// After
int getRating(Driver driver) {
    return driver.getLateDeliveries() > 5 ? 2 : 1;
}
```

### Extract Variable
**Problem**: A complex expression is hard to understand.

```java
// Before
if (order.getQuantity() > 100 && "IT".equals(order.getDepartment())
        && order.getDiscount() > 0.05) { ... }

// After
boolean largeOrder   = order.getQuantity() > 100;
boolean itDepartment = "IT".equals(order.getDepartment());
boolean hasDiscount  = order.getDiscount() > 0.05;
if (largeOrder && itDepartment && hasDiscount) { ... }
```

### Replace Temp with Query
**Problem**: A temp variable stores an expression used in multiple places.

```java
// Before
double basePrice = quantity * itemPrice;
if (basePrice > 1000) return basePrice * 0.95;
else return basePrice * 0.98;

// After
if (basePrice() > 1000) return basePrice() * 0.95;
else return basePrice() * 0.98;

private double basePrice() { return quantity * itemPrice; }
```

### Replace Method with Method Object
**Problem**: A long method has many interacting local variables, making Extract Method hard.

```java
// Before — hard to extract pieces due to locals sharing state
double price(Order order) {
    double primaryBasePrice;
    double secondaryBasePrice;
    double tertiaryBasePrice;
    // ... hundreds of lines using all three
}

// After — each local becomes a field; sub-methods are easy to extract
class PriceCalculator {
    private final Order order;
    private double primaryBasePrice;
    private double secondaryBasePrice;
    private double tertiaryBasePrice;

    PriceCalculator(Order order) { this.order = order; }

    double compute() {
        primaryBasePrice   = computePrimary();
        secondaryBasePrice = computeSecondary();
        tertiaryBasePrice  = computeTertiary();
        return primaryBasePrice - secondaryBasePrice + tertiaryBasePrice;
    }

    private double computePrimary()   { ... }
    private double computeSecondary() { ... }
    private double computeTertiary()  { ... }
}

// Original method becomes:
double price(Order order) { return new PriceCalculator(order).compute(); }
```

---

## Moving Features Between Objects

### Move Method
**Problem**: A method uses data from another class more than its own.

```java
// Before — logic lives in Account but belongs in AccountType
class Account {
    double overdraftCharge() {
        if (type.isPremium()) {
            double result = 10;
            if (daysOverdrawn > 7) result += (daysOverdrawn - 7) * 0.85;
            return result;
        }
        return daysOverdrawn * 1.75;
    }
}

// After
class AccountType {
    double overdraftCharge(int daysOverdrawn) {
        if (isPremium()) {
            double result = 10;
            if (daysOverdrawn > 7) result += (daysOverdrawn - 7) * 0.85;
            return result;
        }
        return daysOverdrawn * 1.75;
    }
}

class Account {
    double overdraftCharge() { return type.overdraftCharge(daysOverdrawn); }
}
```

### Extract Class
**Problem**: One class is doing the work of two.

```java
// Before
class Person {
    private String name;
    private String officeAreaCode;
    private String officeNumber;

    String getTelephoneNumber() { return officeAreaCode + " " + officeNumber; }
}

// After
class TelephoneNumber {
    private String areaCode;
    private String number;

    String getNumber() { return areaCode + " " + number; }
    // getters/setters...
}

class Person {
    private String name;
    private TelephoneNumber officeTelephone = new TelephoneNumber();

    String getTelephoneNumber() { return officeTelephone.getNumber(); }
}
```

### Inline Class
**Problem**: A class does almost nothing and its responsibilities have been moved elsewhere.

```java
// Before — TelephoneNumber only wraps two strings now
class Person {
    TelephoneNumber officeTelephone;
    String getTelephoneNumber() { return officeTelephone.getNumber(); }
}

// After — absorb fields and remove TelephoneNumber
class Person {
    private String officeAreaCode;
    private String officeNumber;
    String getTelephoneNumber() { return officeAreaCode + " " + officeNumber; }
}
```

### Hide Delegate
**Problem**: Client navigates through an intermediate object to get to the real target.

```java
// Before — client depends on Department class
Manager manager = person.getDepartment().getManager();

// After — Person hides the Department
class Person {
    public Manager getManager() { return department.getManager(); }
}
Manager manager = person.getManager();
```

### Remove Middle Man
**Problem**: A class is doing too much simple delegation.

```java
// Before — Person just forwards everything to Department
class Person {
    public Manager getManager() { return department.getManager(); }
    public String getDeptName() { return department.getName(); }
    // ... many more thin wrappers
}

// After — client calls department directly
Manager manager = person.getDepartment().getManager();
```

---

## Organizing Data

### Replace Data Value with Object
**Problem**: A primitive represents a domain concept that needs behavior.

```java
// Before
class Order {
    private String customerName;
}

// After
class Customer {
    private final String name;
    Customer(String name) { this.name = name; }
    String getName() { return name; }
    // can add validation, formatting, etc.
}

class Order {
    private Customer customer;
}
```

### Encapsulate Field
**Problem**: A public field is directly accessible.

```java
// Before
class Person { public String name; }

// After
class Person {
    private String name;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

### Replace Magic Number with Symbolic Constant

```java
// Before
double potentialEnergy(double mass, double height) {
    return mass * height * 9.81;
}

// After
static final double GRAVITATIONAL_CONSTANT = 9.81;

double potentialEnergy(double mass, double height) {
    return mass * height * GRAVITATIONAL_CONSTANT;
}
```

### Replace Type Code with Subclasses
**Problem**: An integer type code affects behavior of the class.

```java
// Before
class Employee {
    static final int ENGINEER = 0, MANAGER = 1, SALESMAN = 2;
    private final int type;

    int payAmount() {
        return switch (type) {
            case ENGINEER  -> basePay;
            case MANAGER   -> basePay + bonus;
            case SALESMAN  -> basePay + commission;
            default        -> throw new IllegalStateException();
        };
    }
}

// After
abstract class Employee {
    abstract int payAmount();

    static Employee create(int type) {
        return switch (type) {
            case 0 -> new Engineer();
            case 1 -> new Manager();
            case 2 -> new Salesman();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

class Engineer  extends Employee { @Override int payAmount() { return basePay; } }
class Manager   extends Employee { @Override int payAmount() { return basePay + bonus; } }
class Salesman  extends Employee { @Override int payAmount() { return basePay + commission; } }
```

### Replace Type Code with State/Strategy
**Problem**: Type code can change during the object's lifetime.

```java
// EmploymentState can be swapped at runtime
interface EmploymentState {
    int payAmount(Employee e);
}

class FullTimeState implements EmploymentState {
    public int payAmount(Employee e) { return e.getBasePay(); }
}
class PartTimeState implements EmploymentState {
    public int payAmount(Employee e) { return e.getBasePay() / 2; }
}

class Employee {
    private EmploymentState state;
    void setState(EmploymentState s) { this.state = s; }
    int payAmount() { return state.payAmount(this); }
}
```

---

## Simplifying Conditional Expressions

### Decompose Conditional
```java
// Before
if (date.before(SUMMER_START) || date.after(SUMMER_END)) {
    charge = quantity * winterRate + winterServiceCharge;
} else {
    charge = quantity * summerRate;
}

// After
charge = isWinter(date) ? winterCharge(quantity) : summerCharge(quantity);

private boolean isWinter(Date date) {
    return date.before(SUMMER_START) || date.after(SUMMER_END);
}
private double winterCharge(int qty) { return qty * winterRate + winterServiceCharge; }
private double summerCharge(int qty)  { return qty * summerRate; }
```

### Replace Nested Conditional with Guard Clauses
```java
// Before — deeply nested, hard to find the normal path
double getPayAmount() {
    if (!isDead) {
        if (!isSeparated) {
            if (!isRetired) {
                return normalPayAmount();
            } else return retiredPayAmount();
        } else return separatedPayAmount();
    } else return deadPayAmount();
}

// After — guard clauses; normal path is obvious
double getPayAmount() {
    if (isDead)      return deadPayAmount();
    if (isSeparated) return separatedPayAmount();
    if (isRetired)   return retiredPayAmount();
    return normalPayAmount();
}
```

### Replace Conditional with Polymorphism
```java
// Before
double getSpeed() {
    return switch (type) {
        case EUROPEAN  -> getBaseSpeed();
        case AFRICAN   -> getBaseSpeed() - getLoadFactor() * numberOfCoconuts;
        case NORWEGIAN -> getBaseSpeed() + voltage;
        default        -> throw new IllegalStateException();
    };
}

// After
abstract class Bird {
    abstract double getSpeed();
    protected double getBaseSpeed() { ... }
}
class EuropeanBird extends Bird {
    @Override double getSpeed() { return getBaseSpeed(); }
}
class AfricanBird extends Bird {
    @Override double getSpeed() { return getBaseSpeed() - getLoadFactor() * numberOfCoconuts; }
}
class NorwegianBlueParrot extends Bird {
    @Override double getSpeed() { return getBaseSpeed() + voltage; }
}
```

### Introduce Null Object
```java
// Before — null checks scattered everywhere
String plan;
if (customer != null) plan = customer.getPlan();
else plan = BillingPlan.BASIC;

// After — NullCustomer handles the missing case
class NullCustomer extends Customer {
    @Override public String getPlan() { return BillingPlan.BASIC; }
    @Override public boolean isNull()  { return true; }
}

// A factory method returns NullCustomer instead of null:
Customer customer = customerRepo.findById(id); // never returns null
String plan = customer.getPlan();               // no null check needed
```

### Introduce Assertion
```java
// Before — implicit assumption with comment
// Assumes discount rate is positive
double applyDiscount(double price) {
    return price * (1 - discountRate);
}

// After
double applyDiscount(double price) {
    assert discountRate >= 0 && discountRate < 1
        : "Discount rate must be between 0 and 1, was: " + discountRate;
    return price * (1 - discountRate);
}
```

---

## Simplifying Method Calls

### Separate Query from Modifier
```java
// Before — method both returns data AND sends email (violates CQS)
double getTotalOutstandingAndSendBill() {
    double total = orders.stream().mapToDouble(Order::amount).sum();
    sendBill();
    return total;
}

// After — pure query + explicit command
double getTotalOutstanding() {
    return orders.stream().mapToDouble(Order::amount).sum();
}
void sendBill() { emailGateway.send(formatBill()); }
```

### Introduce Parameter Object
```java
// Before — start/end always travel together
double amountInvoicedIn(Date start, Date end) { ... }
double amountReceivedIn(Date start, Date end) { ... }
double amountOverdueIn(Date start, Date end)  { ... }

// After — bundle them; add validation/logic in one place
class DateRange {
    private final Date start;
    private final Date end;
    DateRange(Date start, Date end) { this.start = start; this.end = end; }
    boolean includes(Date date) { return !date.before(start) && !date.after(end); }
}

double amountInvoicedIn(DateRange range) { ... }
double amountReceivedIn(DateRange range) { ... }
double amountOverdueIn(DateRange range)  { ... }
```

### Replace Constructor with Factory Method
```java
// Before
Employee engineer = new Employee(Employee.ENGINEER);

// After
class Employee {
    private Employee(int type) { this.type = type; }

    public static Employee createEngineer() { return new Employee(ENGINEER); }
    public static Employee createManager()  { return new Employee(MANAGER); }
    public static Employee createSalesman() { return new Employee(SALESMAN); }
}

Employee engineer = Employee.createEngineer();
```

### Replace Error Code with Exception
```java
// Before
int withdraw(int amount) {
    if (amount > balance) return -1;
    balance -= amount;
    return 0;
}

// After
void withdraw(int amount) {
    if (amount > balance)
        throw new InsufficientFundsException(
            "Requested: " + amount + ", available: " + balance);
    balance -= amount;
}
```

---

## Dealing with Generalization

### Pull Up Method
**Problem**: Two subclasses have methods with identical or near-identical results.

```java
// Before
class Engineer extends Employee {
    String getType() { return "Engineer"; }
}
class Manager extends Employee {
    String getType() { return "Manager"; }
}

// After — extract common *structure* to superclass
abstract class Employee {
    abstract String getType(); // or implement default if truly identical
}
```

### Form Template Method
**Problem**: Two methods in subclasses perform similar steps in the same order.

```java
abstract class ReportGenerator {
    // Template method — defines the algorithm skeleton
    final String generate() {
        return header() + "\n" + body() + "\n" + footer();
    }
    abstract String header();
    abstract String body();
    String footer() { return "--- End of Report ---"; } // hook with default
}

class HtmlReport extends ReportGenerator {
    String header() { return "<html><body>"; }
    String body()   { return "<p>Content</p>"; }
    String footer() { return "</body></html>"; }
}

class MarkdownReport extends ReportGenerator {
    String header() { return "# Report"; }
    String body()   { return "Content"; }
}
```

### Replace Inheritance with Delegation
```java
// Before — Stack extends Vector, exposing 50+ unwanted methods
class Stack<T> extends Vector<T> {
    public T push(T item) { addElement(item); return item; }
    public T pop() { T top = peek(); removeElementAt(size() - 1); return top; }
}

// After — compose instead
class Stack<T> {
    private final Vector<T> storage = new Vector<>();

    public T push(T item) { storage.addElement(item); return item; }
    public T pop()        { T top = peek(); storage.removeElementAt(storage.size()-1); return top; }
    public T peek()       { return storage.lastElement(); }
    public boolean isEmpty() { return storage.isEmpty(); }
}
```

### Extract Superclass
**Problem**: Two classes share common behavior but have no common parent.

```java
// Before — Department and Employee both have name + annual cost logic but share no parent
class Department { String name; double annualCost() { ... } }
class Employee   { String name; double annualCost() { ... } }

// After
abstract class Party {
    protected String name;
    abstract double annualCost();
    String getName() { return name; }
}
class Department extends Party { @Override double annualCost() { ... } }
class Employee   extends Party { @Override double annualCost() { ... } }
```
