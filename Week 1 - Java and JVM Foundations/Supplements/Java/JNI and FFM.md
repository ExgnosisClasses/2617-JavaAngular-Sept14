# Native Interop in Java: JNI./ FFM API

---

## Importance

Production Java system eventually touches native code
- database drivers
- networking stacks
- cryptography
- media codecs
- hardware SDKs
  
Legacy solution was JNI Java Native Interface (JNI).

JDK 22 introduces an updated solution: the Foreign Function and Memory (FFM) API.

You will still see JNI in production codebases and in third-party libraries, so you cannot ignore it. 

But new code you write should almost always use FFM. 

---

## JNI

Introduced in JDK 1.1 (1997) as the primary bridge between Java and native code, at that time mostly  C.
- Pragmatic reason was to avoid having to rewrite existing libraries (mostly written in C) into Java
  - To make adoption of Java easier for existing codebases

Java declares a method as `native`; the implementation lives in a shared library (`.dll`, `.so`, `.dylib`).
- Native code receives a `JNIEnv*` pointer and uses it to read and write Java objects, call Java methods, and manage references.
- Two-way: Java can call native code (downcalls) and native code can call back into Java (upcalls).
- Language-neutral in principle: any language that can export C-callable symbols (C++, Rust, Go, Zig, Fortran) 
  - Anything with a C-compatible calling convention works.

---

## The JNI Workflow (Classic)

1. Declare `native` methods in a Java class.
2. Generate a C header from the class (historically `javah`, now `javac -h`).
3. Write the C implementation against the generated signatures, using `JNIEnv` functions.
4. Compile into a platform-specific shared library.
5. Load it at runtime with `System.loadLibrary` or `System.load`.

Every platform and architecture needs its own compiled artifact.
- Any change to the Java signature requires regenerating headers and recompiling native code.
- Steps 2 through 4 above are entirely outside Java
  - A mismatch between the Java declaration and the C signature is not caught until runtime, if at all. 
- This is the "brittleness" that JEP 454 refers to.

---

## Why JNI Is Hard

1. Manual reference management: local references, global references, and the leaks that follow from getting them wrong.
2. Thread affinity: a `JNIEnv*` is valid only on the thread that received it.
3. Exceptions raised in native code must be checked explicitly after every JNI call.
4. Functions such as `GetPrimitiveArrayCritical` can stall the garbage collector if misused.
5. Native code can corrupt JVM memory with no safety net.
6. Debugging spans two toolchains and two memory models.

These are the types of bugs you will encounter in legacy code that you may be tasked to fix.
- Reference leaks and thread misuse are the two most common
- A crash in native code usually takes the whole JVM down with a `hs_err_pid` log rather than a Java exception
  - This is a very different failure mode from what happens in pure Java.

---

## Project Panama and the FFM API

Project Panama: OpenJDK effort to improve Java-to-native interoperability.
- Main deliverable is the Foreign Function and Memory API in the `java.lang.foreign` package.
- Timeline
  - Incubator in JDK 17 through 19
  - Preview in JDK 19 through 21
  - Final in JDK 22 (JEP 454).
  - Available without flags in JDK 22 and later, including JDK 25 (LTS). 
  
Stated Goal: replace `native` methods and JNI with a concise, readable, pure-Java API at comparable performance.

If you are using JDK 21, FFM is preview-only there and requires `--enable-preview`

If you are using JDK 25, FFM is the normal, supported path. 

---

## FFM API Building Blocks

- `MemorySegment`: a bounded view of on-heap or off-heap memory, checked for bounds and lifetime.
- `Arena`: controls the lifetime of segments; `Arena.ofConfined()` in a try-with-resources block frees everything on exit.
- `MemoryLayout` and `ValueLayout`: describe the shape of native data (structs, arrays, primitives).
- `SymbolLookup`: finds a function or variable in a loaded library.
- `Linker`: turns a symbol plus a `FunctionDescriptor` into a `MethodHandle` for downcalls, or wraps a Java `MethodHandle` as a native function pointer for upcalls.
- `jextract`: a companion tool that generates all of the above from a C header file.

The mental model is:
- Describe the function in Java, obtain a method handle, invoke it. 
- No C code is written and no platform-specific artifact is compiled by the Java developer. 
- The `Arena` is the key safety improvement over JNI; memory has an explicit, enforced lifetime instead of a manually tracked one.

---

## FFM: Calling `strlen`

```java
import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;

public class StrLen {
    public static void main(String[] args) throws Throwable {
        Linker linker = Linker.nativeLinker();
        SymbolLookup stdlib = linker.defaultLookup();

        MethodHandle strlen = linker.downcallHandle(
            stdlib.find("strlen").orElseThrow(),
            FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS)
        );

        try (Arena arena = Arena.ofConfined()) {
            MemorySegment cString = arena.allocateFrom("Hello, Panama");
            long length = (long) strlen.invokeExact(cString);
            System.out.println(length);   // 13
        }
    }
}
```

- Run with `java --enable-native-access=ALL-UNNAMED StrLen.java` on JDK 22 or later.

Read the code top to bottom. 
- `defaultLookup` finds symbols in the C standard library. 
- The `FunctionDescriptor` says "returns a long, takes a pointer". 
- The `Arena` allocates a C string and frees it when the block exits. 

---


---

## Integrity by Default

Existing JNI libraries keep working, but on current JDKs they print a warning at load time unless the application opts in. 
- If you are running older code on a current JVM and see an unexpected warning
  - You should recognise it as this policy, not a bug.
---

## JNI in 2026: Status Summary

Still supported and still ubiquitous in native transports, many JDBC drivers, graphics and media bindings, the entire Android NDK model, and parts of the JDK itself.
- Not deprecated, and no removal is planned; breaking it would break the ecosystem.
- No longer the recommended way to write new native interop.
- Being placed behind the same explicit opt-in as FFM.
- Best described as a maintenance-mode API: read it, debug it, migrate away from it when you can.

---

## Beyond C: Other Languages and Runtimes

Both JNI and FFM use the C ABI (Application Binary Interface)
-  Contract at the machine-code level for how compiled functions talk to each other
  - How arguments are passed (which registers, which go on the stack, in what order)
  - How return values come back, how the stack is set up and cleaned up
  - How data types are laid out in memory (sizes, alignment
  - Struct field ordering), and how symbol names appear in the compiled library.
- Rust (`extern "C"`), C++ (`extern "C"`), Go (cgo), and Zig all work as native targets.
 
C ABI is the universal adapter, and neither JNI nor FFM needed per-language extensions. 

---


## References: Tutorials and Examples

- dev.java FFM tutorial series (memory segments, calling C, structs, upcalls, troubleshooting, jextract): https://dev.java/learn/ffm/
- dev.java jextract walkthrough: https://dev.java/learn/ffm/jextract
- jextract tool and pre-built binaries: https://github.com/openjdk/jextract
- Panama FFI design document from the OpenJDK team: https://github.com/openjdk/panama-foreign/blob/foreign-memaccess+abi/doc/panama_ffi.md
- Standalone FFM example projects (memory layouts, jextract, a deliberate memory leak): https://github.com/dgroomes/java-foreign-function-and-memory-api-playground
- Real-world port from JNI to FFM, serial port library: https://github.com/calimero-project/serial-ffm
- JavaOne 2025 talk, "Interconnecting Java and Native Code with the FFM API": https://inside.java/2025/06/14/javaone-ffm/


