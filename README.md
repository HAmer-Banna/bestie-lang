# **Bestie Programming Language**
### **“Fast as light. Native with no GC. Control of C. Beauty of Kotlin.”**  
### **Elegance with Excellence**

---

## 🌊 **Vision**

Bestie is a modern compiled language designed to unify two historically separate worlds:

- **Systems programming** (C, Rust, Zig)  
- **Backend & application development** (Java, Kotlin, Go)

**Bestie’s mission:**

> **Bring the power, speed, and control of systems languages to backend developers —  
while giving system programmers the ergonomics, expressiveness, and elegance of high-level languages.**

---

## 🔥 **Core Principles**

### **No GC — ever**
Memory is explicit by design. No garbage collector, no hidden costs.

### **Native performance**
Compiles directly to machine code with deterministic behavior and predictable performance.

### **Elegant syntax**
Inspired by Kotlin: readable, expressive, joyful to write.

### **Safe control**
Bestie introduces a fresh memory model that provides:

- C-level control  
- Rust-like safety  
- Kotlin-like developer experience  
- **All enforced at compile time**

---

## 🌱 **Language Elements**

Bestie includes a minimal yet powerful set of constructs:

- `package`
- `data class` (immutable, inlined)
- `value class` (inlined)
- `abstract class`
- `single class` (inline singleton)
- `class` (final, inlined)
- `open class`
- `enum`
- `enum class`
- `protocol`
- `group protocol`
- `fun`
- `inline fun`
- Lambdas

Every class automatically gets:

- `.new()`
- `.free()`
- `.address()`
- `.deref()`

---

## ⚡ **Memory Model Overview**

Bestie exposes memory management in a **safe and friendly** way.

### **Allocation**
```bestie
val user = User.new()
