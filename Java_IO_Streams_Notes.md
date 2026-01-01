# Java I/O: Streams and Files

## Module Topics
- Streams basics
- Stream errors & cleanup
- Chaining streams
- Specialized stream classes (file, buffer)
- `java.nio.file` package
- File systems
- Zip file systems

---

## Streams Overview

### Definition
- Ordered sequence of data
- Common I/O model abstraction
- Abstracts source/destination (memory, disk, network)

### Characteristics
- **Unidirectional**: Read OR write (not both)
- **Two types**:
  - Byte streams (binary)
  - Text streams (character-based)

---

## Reading and Writing

### Base Classes
- **Byte streams**: `InputStream` (read), `OutputStream` (write)
- **Text streams**: `Reader` (read), `Writer` (write)

### Reading

#### Single Value
- `read()` returns `int` (not byte/char)
- End of stream = `-1`
- Cast `int` to `byte` (byte stream) or `char` (text stream)

```java
// Byte stream
int intVal = input.read();
if (intVal >= 0) {
    byte b = (byte) intVal;
}

// Text stream
int intVal = reader.read();
if (intVal >= 0) {
    char c = (char) intVal;
}
```

#### Array Reading
- `read(byte[] buf)` / `read(char[] buf)`
- Returns `int` = count of values read
- May read less than array size
- Loop up to returned length (not array length)

### Writing
- `OutputStream.write(int)` - single byte
- `OutputStream.write(byte[])` - byte array
- `Writer.write(char)` - single character
- `Writer.write(char[])` - char array
- `Writer.write(String)` - string

---

## Common Stream Classes

### Byte Streams (inherit InputStream/OutputStream)
- `ByteArrayInputStream/OutputStream` - memory (byte array)
- `PipedInputStream/OutputStream` - producer-consumer
- `FileInputStream/OutputStream` - files

### Text Streams (inherit Reader/Writer)
- `CharArrayReader/Writer` - memory (char array)
- `StringReader/Writer` - string buffer
- `PipedReader/Writer` - producer-consumer
- `InputStreamReader/OutputStreamWriter` - bridge binary→text
- `FileReader/FileWriter` - files

---

## Stream Errors and Cleanup

### Issues
- Streams throw exceptions (need error handling)
- Physical resources (files, network) need explicit cleanup
- GC can't reliably clean external resources

### Closeable Interface
- `close()` method
- Must call when done

### AutoCloseable & Try-with-Resources
- `Closeable extends AutoCloseable`
- `try-with-resources` auto-calls `close()`
- Catch blocks handle errors from try block AND close()

```java
try (Reader reader = ...) {
    // use reader
} catch (IOException e) {
    // handles both try block and close() errors
}
```

