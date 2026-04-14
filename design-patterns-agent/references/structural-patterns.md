# Structural Patterns Reference
Source: refactoring.guru/design-patterns/structural-patterns

---

## Adapter

**Intent**: Makes two incompatible interfaces work together by wrapping one in an adapter.

**Preferred variant**: Object Adapter (composition over inheritance).

```java
// Existing interface your client code expects
interface JsonDataProvider {
    String getJson();
}

// Third-party class with an incompatible interface (can't modify it)
class XmlDataService {
    public String fetchXml() {
        return "<data><item>value</item></data>";
    }
}

// Adapter — wraps XmlDataService, exposes JsonDataProvider
class XmlToJsonAdapter implements JsonDataProvider {
    private final XmlDataService xmlService;

    XmlToJsonAdapter(XmlDataService xmlService) {
        this.xmlService = xmlService;
    }

    @Override
    public String getJson() {
        String xml = xmlService.fetchXml();
        return convertXmlToJson(xml); // translation logic here
    }

    private String convertXmlToJson(String xml) {
        // ... conversion logic
        return "{\"item\": \"value\"}";
    }
}

// Client code works only with JsonDataProvider — unaware of XML
void processData(JsonDataProvider provider) {
    String json = provider.getJson();
    // ... process json
}

// Usage
XmlDataService xmlService = new XmlDataService();
JsonDataProvider adapter  = new XmlToJsonAdapter(xmlService);
processData(adapter);
```

---

## Bridge

**Intent**: Decouples abstraction from implementation so both can evolve independently. Avoids a class explosion when extending in two dimensions.

```java
// Implementation hierarchy
interface Renderer {
    void renderCircle(double x, double y, double radius);
}

class VectorRenderer implements Renderer {
    @Override public void renderCircle(double x, double y, double radius) {
        System.out.printf("Drawing vector circle at (%.0f,%.0f) r=%.0f%n", x, y, radius);
    }
}
class RasterRenderer implements Renderer {
    @Override public void renderCircle(double x, double y, double radius) {
        System.out.printf("Drawing raster circle at (%.0f,%.0f) r=%.0f%n", x, y, radius);
    }
}

// Abstraction hierarchy
abstract class Shape {
    protected final Renderer renderer; // bridge to implementation
    Shape(Renderer renderer) { this.renderer = renderer; }
    abstract void draw();
    abstract void resize(double factor);
}

class Circle extends Shape {
    private double x, y, radius;

    Circle(double x, double y, double radius, Renderer renderer) {
        super(renderer);
        this.x = x; this.y = y; this.radius = radius;
    }

    @Override public void draw()                  { renderer.renderCircle(x, y, radius); }
    @Override public void resize(double factor)   { radius *= factor; }
}

// Usage — mix any Shape with any Renderer
Shape c1 = new Circle(5, 10, 8, new VectorRenderer());
Shape c2 = new Circle(5, 10, 8, new RasterRenderer());
c1.draw(); // vector
c2.draw(); // raster
```

---

## Composite

**Intent**: Composes objects into tree structures for part-whole hierarchies. Clients treat leaves and composites uniformly.

```java
// Component interface
interface FileSystemItem {
    String getName();
    long getSize();
    void display(String indent);
}

// Leaf
class File implements FileSystemItem {
    private final String name;
    private final long size;

    File(String name, long size) { this.name = name; this.size = size; }

    @Override public String getName()              { return name; }
    @Override public long getSize()                { return size; }
    @Override public void display(String indent)   { System.out.println(indent + name + " (" + size + " bytes)"); }
}

// Composite
class Directory implements FileSystemItem {
    private final String name;
    private final List<FileSystemItem> children = new ArrayList<>();

    Directory(String name) { this.name = name; }

    public void add(FileSystemItem item)    { children.add(item); }
    public void remove(FileSystemItem item) { children.remove(item); }

    @Override public String getName() { return name; }
    @Override public long getSize()   { return children.stream().mapToLong(FileSystemItem::getSize).sum(); }
    @Override public void display(String indent) {
        System.out.println(indent + "[" + name + "]");
        children.forEach(c -> c.display(indent + "  "));
    }
}

// Usage — client treats File and Directory identically
Directory root = new Directory("root");
Directory src  = new Directory("src");
src.add(new File("Main.java", 1200));
src.add(new File("App.java", 850));
root.add(src);
root.add(new File("README.md", 300));
root.display("");
System.out.println("Total size: " + root.getSize());
```

---

## Decorator

**Intent**: Attaches additional responsibilities to an object dynamically by wrapping it. Stackable alternative to subclassing.

```java
// Component interface
interface TextProcessor {
    String process(String text);
}

// Concrete Component
class PlainTextProcessor implements TextProcessor {
    @Override public String process(String text) { return text; }
}

// Base Decorator
abstract class TextProcessorDecorator implements TextProcessor {
    protected final TextProcessor wrapped;
    TextProcessorDecorator(TextProcessor wrapped) { this.wrapped = wrapped; }
    @Override public String process(String text) { return wrapped.process(text); }
}

// Concrete Decorators
class TrimDecorator extends TextProcessorDecorator {
    TrimDecorator(TextProcessor wrapped) { super(wrapped); }
    @Override public String process(String text) { return super.process(text).trim(); }
}

class UpperCaseDecorator extends TextProcessorDecorator {
    UpperCaseDecorator(TextProcessor wrapped) { super(wrapped); }
    @Override public String process(String text) { return super.process(text).toUpperCase(); }
}

class HtmlEscapeDecorator extends TextProcessorDecorator {
    HtmlEscapeDecorator(TextProcessor wrapped) { super(wrapped); }
    @Override public String process(String text) {
        return super.process(text)
            .replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;");
    }
}

// Usage — stack decorators at runtime
TextProcessor processor = new UpperCaseDecorator(
                              new TrimDecorator(
                                  new PlainTextProcessor()));

String result = processor.process("  hello world  ");
// → "HELLO WORLD"

// Add HTML escaping dynamically
TextProcessor safeProcessor = new HtmlEscapeDecorator(processor);
System.out.println(safeProcessor.process("  <b>hello</b>  "));
// → "&LT;B&GT;HELLO&LT;/B&GT;"
```

---

## Facade

**Intent**: Provides a simplified interface to a complex subsystem.

```java
// Complex subsystem classes
class VideoDecoder {
    public void loadFile(String path) { System.out.println("Loading: " + path); }
    public byte[] decode()            { return new byte[0]; /* ... */ }
}
class AudioDecoder {
    public void init()                { System.out.println("Init audio"); }
    public int[] extractAudio(byte[] video) { return new int[0]; /* ... */ }
}
class VideoEncoder {
    public void setFormat(String fmt) { System.out.println("Format: " + fmt); }
    public void encode(byte[] data)   { System.out.println("Encoding video"); }
}
class AudioMixer {
    public void mix(int[] audio, byte[] video) { System.out.println("Mixing"); }
}

// Facade — hides all the above
class VideoConverterFacade {
    private final VideoDecoder  videoDecoder = new VideoDecoder();
    private final AudioDecoder  audioDecoder = new AudioDecoder();
    private final VideoEncoder  videoEncoder = new VideoEncoder();
    private final AudioMixer    audioMixer   = new AudioMixer();

    public void convertToMp4(String inputPath, String outputPath) {
        videoDecoder.loadFile(inputPath);
        byte[] videoData = videoDecoder.decode();

        audioDecoder.init();
        int[] audio = audioDecoder.extractAudio(videoData);

        audioMixer.mix(audio, videoData);

        videoEncoder.setFormat("mp4");
        videoEncoder.encode(videoData);
        System.out.println("Saved to: " + outputPath);
    }
}

// Client only talks to the facade
VideoConverterFacade converter = new VideoConverterFacade();
converter.convertToMp4("movie.avi", "movie.mp4");
```

---

## Flyweight

**Intent**: Reduces memory by sharing common (intrinsic) state among many similar objects.

**Key distinction**:
- **Intrinsic state** — stored inside the flyweight, shared, context-independent
- **Extrinsic state** — passed in by client, context-dependent, not stored

```java
// Flyweight — stores intrinsic (shared) state
class TreeType {
    private final String name;
    private final String color;
    private final String texture; // could be a large image

    TreeType(String name, String color, String texture) {
        this.name = name; this.color = color; this.texture = texture;
    }

    // Extrinsic state (x, y) is passed in, not stored
    public void draw(int x, int y) {
        System.out.printf("Drawing %s tree [%s] at (%d,%d)%n", name, color, x, y);
    }
}

// Flyweight factory / registry
class TreeTypeFactory {
    private static final Map<String, TreeType> cache = new HashMap<>();

    static TreeType get(String name, String color, String texture) {
        String key = name + "_" + color;
        return cache.computeIfAbsent(key, k -> {
            System.out.println("Creating new TreeType: " + key);
            return new TreeType(name, color, texture);
        });
    }
    static int getCacheSize() { return cache.size(); }
}

// Context object — stores extrinsic (unique) state
class Tree {
    private final int x, y;
    private final TreeType type; // shared flyweight

    Tree(int x, int y, TreeType type) { this.x = x; this.y = y; this.type = type; }
    void draw() { type.draw(x, y); }
}

// Usage — 1 million trees, only 3 TreeType objects in memory
List<Tree> forest = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    TreeType oak = TreeTypeFactory.get("Oak", "green", "oak_texture.png");
    forest.add(new Tree(random.nextInt(1000), random.nextInt(1000), oak));
}
System.out.println("TreeType objects: " + TreeTypeFactory.getCacheSize()); // → 1
```

---

## Proxy

**Intent**: Provides a placeholder for another object to control access, add logging, caching, etc.

| Proxy Type | Purpose |
|---|---|
| Virtual | Lazy initialization of expensive objects |
| Remote | Local representative for a remote service |
| Protection | Access control / permission checks |
| Logging | Log all requests to the real service |
| Caching | Cache expensive results |

```java
// Subject interface
interface DataService {
    List<String> fetchData(String query);
}

// Real subject — expensive
class DatabaseDataService implements DataService {
    @Override public List<String> fetchData(String query) {
        System.out.println("DB query: " + query); // expensive!
        return List.of("result1", "result2");
    }
}

// Caching + logging proxy
class DataServiceProxy implements DataService {
    private final DataService realService;
    private final Map<String, List<String>> cache = new HashMap<>();

    DataServiceProxy(DataService realService) { this.realService = realService; }

    @Override public List<String> fetchData(String query) {
        if (cache.containsKey(query)) {
            System.out.println("Cache hit for: " + query);
            return cache.get(query);
        }
        System.out.println("Cache miss — delegating: " + query);
        List<String> result = realService.fetchData(query);
        cache.put(query, result);
        return result;
    }
}

// Usage — client only knows DataService interface
DataService service = new DataServiceProxy(new DatabaseDataService());
service.fetchData("SELECT *"); // DB query
service.fetchData("SELECT *"); // Cache hit
```
