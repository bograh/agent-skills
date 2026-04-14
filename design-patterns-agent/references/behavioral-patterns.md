# Behavioral Patterns Reference
Source: refactoring.guru/design-patterns/behavioral-patterns

---

## Chain of Responsibility

**Intent**: Passes requests along a chain of handlers; each decides to process or pass forward.

```java
// Handler interface
abstract class RequestHandler {
    private RequestHandler next;

    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next; // enables chaining: h1.setNext(h2).setNext(h3)
    }

    protected boolean passToNext(HttpRequest req) {
        if (next != null) { next.handle(req); return true; }
        return false;
    }

    public abstract void handle(HttpRequest req);
}

// Concrete handlers
class AuthHandler extends RequestHandler {
    @Override public void handle(HttpRequest req) {
        if (req.getToken() == null) {
            System.out.println("401 Unauthorized");
            return;
        }
        System.out.println("Auth passed");
        passToNext(req);
    }
}

class RateLimitHandler extends RequestHandler {
    private int requestCount = 0;
    @Override public void handle(HttpRequest req) {
        if (++requestCount > 100) {
            System.out.println("429 Too Many Requests");
            return;
        }
        passToNext(req);
    }
}

class BusinessLogicHandler extends RequestHandler {
    @Override public void handle(HttpRequest req) {
        System.out.println("Processing: " + req.getPath());
    }
}

// Usage — build the chain
RequestHandler auth  = new AuthHandler();
RequestHandler limit = new RateLimitHandler();
RequestHandler logic = new BusinessLogicHandler();
auth.setNext(limit).setNext(logic);

auth.handle(new HttpRequest("/api/orders", "Bearer token123"));
```

---

## Command

**Intent**: Encapsulates a request as an object. Enables undo/redo, queuing, logging.

```java
// Command interface
interface Command {
    void execute();
    void undo();
}

// Receiver
class TextEditor {
    private final StringBuilder text = new StringBuilder();
    void insertText(String str)   { text.append(str); }
    void deleteText(int length)   { text.delete(text.length() - length, text.length()); }
    String getText()              { return text.toString(); }
}

// Concrete Commands
class InsertTextCommand implements Command {
    private final TextEditor editor;
    private final String textToInsert;

    InsertTextCommand(TextEditor editor, String text) {
        this.editor = editor; this.textToInsert = text;
    }
    @Override public void execute() { editor.insertText(textToInsert); }
    @Override public void undo()    { editor.deleteText(textToInsert.length()); }
}

// Invoker — history stack enables undo
class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();

    public void execute(Command cmd) {
        cmd.execute();
        history.push(cmd);
    }

    public void undo() {
        if (!history.isEmpty()) history.pop().undo();
    }
}

// Usage
TextEditor editor    = new TextEditor();
CommandHistory hist  = new CommandHistory();

hist.execute(new InsertTextCommand(editor, "Hello"));
hist.execute(new InsertTextCommand(editor, " World"));
System.out.println(editor.getText()); // → "Hello World"
hist.undo();
System.out.println(editor.getText()); // → "Hello"
```

---

## Iterator

**Intent**: Provides sequential access to collection elements without exposing internal structure.

```java
// Custom collection — binary tree with in-order iterator
class TreeNode<T extends Comparable<T>> {
    T value;
    TreeNode<T> left, right;
    TreeNode(T value) { this.value = value; }
}

class BinarySearchTree<T extends Comparable<T>> implements Iterable<T> {
    private TreeNode<T> root;

    public void insert(T value) { root = insert(root, value); }

    private TreeNode<T> insert(TreeNode<T> node, T value) {
        if (node == null) return new TreeNode<>(value);
        if (value.compareTo(node.value) < 0) node.left  = insert(node.left, value);
        else                                 node.right = insert(node.right, value);
        return node;
    }

    @Override
    public Iterator<T> iterator() {
        return new InOrderIterator<>(root);
    }
}

class InOrderIterator<T> implements Iterator<T> {
    private final Deque<TreeNode<T>> stack = new ArrayDeque<>();

    InOrderIterator(TreeNode<T> root) { pushLeft(root); }

    private void pushLeft(TreeNode<T> node) {
        while (node != null) { stack.push(node); node = node.left; }
    }

    @Override public boolean hasNext() { return !stack.isEmpty(); }

    @Override public T next() {
        if (!hasNext()) throw new NoSuchElementException();
        TreeNode<T> node = stack.pop();
        pushLeft(node.right);
        return node.value;
    }
}

// Usage
BinarySearchTree<Integer> bst = new BinarySearchTree<>();
bst.insert(5); bst.insert(3); bst.insert(7); bst.insert(1);

for (int val : bst) System.out.print(val + " "); // → 1 3 5 7
```

---

## Mediator

**Intent**: Centralizes communication between objects so they don't depend on each other directly.

```java
// Mediator interface
interface DialogMediator {
    void notify(Component sender, String event);
}

// Components — know their mediator, but not each other
abstract class Component {
    protected DialogMediator mediator;
    Component(DialogMediator mediator) { this.mediator = mediator; }
    void trigger(String event)         { mediator.notify(this, event); }
}

class Checkbox extends Component {
    boolean checked = false;
    Checkbox(DialogMediator m) { super(m); }
    void toggle() { checked = !checked; trigger("toggle"); }
}

class TextBox extends Component {
    boolean enabled = true;
    TextBox(DialogMediator m) { super(m); }
    void setEnabled(boolean v) { this.enabled = v; System.out.println("TextBox enabled: " + v); }
}

class SubmitButton extends Component {
    boolean enabled = true;
    SubmitButton(DialogMediator m) { super(m); }
    void setEnabled(boolean v) { this.enabled = v; System.out.println("SubmitButton enabled: " + v); }
}

// Concrete Mediator — knows all components, wires them up
class RegistrationDialog implements DialogMediator {
    private Checkbox    termsCheckbox;
    private TextBox     nameField;
    private SubmitButton submitBtn;

    void setComponents(Checkbox cb, TextBox tf, SubmitButton sb) {
        this.termsCheckbox = cb; this.nameField = tf; this.submitBtn = sb;
    }

    @Override public void notify(Component sender, String event) {
        if (sender == termsCheckbox && event.equals("toggle")) {
            nameField.setEnabled(termsCheckbox.checked);
            submitBtn.setEnabled(termsCheckbox.checked);
        }
    }
}

// Usage — components are decoupled from each other
RegistrationDialog dialog = new RegistrationDialog();
Checkbox     cb = new Checkbox(dialog);
TextBox      tf = new TextBox(dialog);
SubmitButton sb = new SubmitButton(dialog);
dialog.setComponents(cb, tf, sb);
cb.toggle(); // mediator enables name field + button
```

---

## Memento

**Intent**: Captures and restores object state without exposing its internals.

```java
// Memento — immutable snapshot (value object)
final class EditorState {
    private final String content;
    private final int cursorPosition;

    EditorState(String content, int cursorPosition) {
        this.content        = content;
        this.cursorPosition = cursorPosition;
    }
    String getContent()    { return content; }
    int getCursorPosition(){ return cursorPosition; }
}

// Originator — creates and restores from mementos
class TextEditor {
    private String content = "";
    private int cursor = 0;

    public void type(String text) { content += text; cursor += text.length(); }
    public void delete(int n)     { content = content.substring(0, content.length() - n); cursor -= n; }

    public EditorState save()              { return new EditorState(content, cursor); }
    public void restore(EditorState state) { content = state.getContent(); cursor = state.getCursorPosition(); }

    public String getContent() { return content; }
}

// Caretaker — manages history
class History {
    private final TextEditor editor;
    private final Deque<EditorState> snapshots = new ArrayDeque<>();

    History(TextEditor editor) { this.editor = editor; }

    public void backup() { snapshots.push(editor.save()); }
    public void undo()   { if (!snapshots.isEmpty()) editor.restore(snapshots.pop()); }
}

// Usage
TextEditor editor  = new TextEditor();
History    history = new History(editor);

editor.type("Hello");
history.backup();

editor.type(" World");
history.backup();

editor.type("!!!");
System.out.println(editor.getContent()); // → "Hello World!!!"
history.undo();
System.out.println(editor.getContent()); // → "Hello World"
history.undo();
System.out.println(editor.getContent()); // → "Hello"
```

---

## Observer

**Intent**: One-to-many dependency — when one object changes state, all dependents are notified automatically.

```java
// Observer interface
interface StockObserver {
    void onPriceChanged(String ticker, double newPrice);
}

// Subject
class StockMarket {
    private final Map<String, Double>          prices   = new HashMap<>();
    private final List<StockObserver>          observers = new ArrayList<>();

    public void addObserver(StockObserver o)    { observers.add(o); }
    public void removeObserver(StockObserver o) { observers.remove(o); }

    public void setPrice(String ticker, double price) {
        prices.put(ticker, price);
        notifyObservers(ticker, price);
    }

    private void notifyObservers(String ticker, double price) {
        observers.forEach(o -> o.onPriceChanged(ticker, price));
    }
}

// Concrete Observers
class AlertService implements StockObserver {
    private final double threshold;
    AlertService(double threshold) { this.threshold = threshold; }
    @Override public void onPriceChanged(String ticker, double price) {
        if (price > threshold) System.out.printf("ALERT: %s hit %.2f%n", ticker, price);
    }
}

class PortfolioTracker implements StockObserver {
    @Override public void onPriceChanged(String ticker, double price) {
        System.out.printf("Portfolio update: %s = %.2f%n", ticker, price);
    }
}

// Usage
StockMarket market   = new StockMarket();
market.addObserver(new AlertService(150.0));
market.addObserver(new PortfolioTracker());

market.setPrice("AAPL", 145.0); // Portfolio update only
market.setPrice("AAPL", 152.0); // Portfolio update + ALERT
```

---

## State

**Intent**: Lets an object alter its behavior when its internal state changes.

```java
// State interface
interface TrafficLightState {
    void handle(TrafficLight light);
    String getColor();
}

// Concrete States
class RedState implements TrafficLightState {
    @Override public void handle(TrafficLight light) { light.setState(new GreenState()); }
    @Override public String getColor() { return "RED"; }
}
class GreenState implements TrafficLightState {
    @Override public void handle(TrafficLight light) { light.setState(new YellowState()); }
    @Override public String getColor() { return "GREEN"; }
}
class YellowState implements TrafficLightState {
    @Override public void handle(TrafficLight light) { light.setState(new RedState()); }
    @Override public String getColor() { return "YELLOW"; }
}

// Context
class TrafficLight {
    private TrafficLightState state;

    TrafficLight()         { state = new RedState(); }
    void setState(TrafficLightState s) { this.state = s; }
    void change()          { state.handle(this); }
    String getColor()      { return state.getColor(); }
}

// Usage
TrafficLight light = new TrafficLight();
for (int i = 0; i < 6; i++) {
    System.out.println(light.getColor());
    light.change();
}
// RED, GREEN, YELLOW, RED, GREEN, YELLOW
```

---

## Strategy

**Intent**: Defines a family of algorithms, encapsulates each, and makes them interchangeable.

```java
// Strategy interface
interface SortStrategy {
    void sort(int[] data);
}

// Concrete strategies
class BubbleSort implements SortStrategy {
    @Override public void sort(int[] data) {
        // O(n²) — good for nearly-sorted small arrays
        for (int i = 0; i < data.length - 1; i++)
            for (int j = 0; j < data.length - i - 1; j++)
                if (data[j] > data[j+1]) { int t = data[j]; data[j] = data[j+1]; data[j+1] = t; }
    }
}
class QuickSort implements SortStrategy {
    @Override public void sort(int[] data) { Arrays.sort(data); /* simplified */ }
}

// Context
class DataSorter {
    private SortStrategy strategy;

    DataSorter(SortStrategy strategy) { this.strategy = strategy; }

    // Swap strategy at runtime
    void setStrategy(SortStrategy strategy) { this.strategy = strategy; }

    int[] sort(int[] data) {
        int[] copy = Arrays.copyOf(data, data.length);
        strategy.sort(copy);
        return copy;
    }
}

// Usage
int[] data = {5, 2, 8, 1, 9, 3};
DataSorter sorter = new DataSorter(new BubbleSort());
System.out.println(Arrays.toString(sorter.sort(data)));

// Switch to QuickSort for larger datasets
sorter.setStrategy(new QuickSort());
System.out.println(Arrays.toString(sorter.sort(data)));
```

---

## Template Method

**Intent**: Defines the skeleton of an algorithm in a base class; subclasses override specific steps.

```java
// Abstract class with template method
abstract class DataImporter {
    // Template method — final; defines the algorithm skeleton
    public final void importData(String source) {
        String raw  = readData(source);    // step 1: varies
        List<?> parsed = parseData(raw);   // step 2: varies
        validateData(parsed);              // step 3: hook (optional override)
        saveData(parsed);                  // step 4: common
        System.out.println("Import complete: " + parsed.size() + " records");
    }

    protected abstract String readData(String source);
    protected abstract List<?> parseData(String raw);

    // Hook — subclasses may override but don't have to
    protected void validateData(List<?> data) {
        System.out.println("Default validation passed");
    }

    private void saveData(List<?> data) {
        System.out.println("Saving " + data.size() + " records to DB");
    }
}

class CsvImporter extends DataImporter {
    @Override protected String readData(String path) {
        return Files.readString(Path.of(path)); // simplified
    }
    @Override protected List<String[]> parseData(String raw) {
        return Arrays.stream(raw.split("\n"))
                     .map(line -> line.split(","))
                     .collect(Collectors.toList());
    }
}

class JsonImporter extends DataImporter {
    @Override protected String readData(String url) {
        return httpGet(url); // simplified
    }
    @Override protected List<?> parseData(String raw) {
        return new ObjectMapper().readValue(raw, List.class); // simplified
    }
    @Override protected void validateData(List<?> data) {
        System.out.println("JSON schema validation...");
    }
}

// Usage
DataImporter importer = new CsvImporter();
importer.importData("users.csv");
```

---

## Visitor

**Intent**: Lets you add new operations to an object structure without modifying those objects.

```java
// Element interface
interface Shape {
    void accept(ShapeVisitor visitor);
}

// Concrete elements
class Circle implements Shape {
    final double radius;
    Circle(double radius) { this.radius = radius; }
    @Override public void accept(ShapeVisitor visitor) { visitor.visitCircle(this); }
}

class Rectangle implements Shape {
    final double width, height;
    Rectangle(double w, double h) { this.width = w; this.height = h; }
    @Override public void accept(ShapeVisitor visitor) { visitor.visitRectangle(this); }
}

class Triangle implements Shape {
    final double base, height;
    Triangle(double base, double height) { this.base = base; this.height = height; }
    @Override public void accept(ShapeVisitor visitor) { visitor.visitTriangle(this); }
}

// Visitor interface — one method per element type
interface ShapeVisitor {
    void visitCircle(Circle c);
    void visitRectangle(Rectangle r);
    void visitTriangle(Triangle t);
}

// Concrete Visitors — add new operations without touching Shape classes
class AreaCalculator implements ShapeVisitor {
    private double totalArea = 0;
    @Override public void visitCircle(Circle c)      { totalArea += Math.PI * c.radius * c.radius; }
    @Override public void visitRectangle(Rectangle r){ totalArea += r.width * r.height; }
    @Override public void visitTriangle(Triangle t)  { totalArea += 0.5 * t.base * t.height; }
    double getTotalArea() { return totalArea; }
}

class XmlExporter implements ShapeVisitor {
    private final StringBuilder xml = new StringBuilder();
    @Override public void visitCircle(Circle c)      { xml.append("<circle r='").append(c.radius).append("'/>"); }
    @Override public void visitRectangle(Rectangle r){ xml.append("<rect w='").append(r.width).append("' h='").append(r.height).append("'/>"); }
    @Override public void visitTriangle(Triangle t)  { xml.append("<triangle b='").append(t.base).append("'/>"); }
    String getXml() { return xml.toString(); }
}

// Usage
List<Shape> shapes = List.of(new Circle(5), new Rectangle(4, 6), new Triangle(3, 8));

AreaCalculator calc = new AreaCalculator();
shapes.forEach(s -> s.accept(calc));
System.out.printf("Total area: %.2f%n", calc.getTotalArea());

XmlExporter exporter = new XmlExporter();
shapes.forEach(s -> s.accept(exporter));
System.out.println(exporter.getXml());
```
