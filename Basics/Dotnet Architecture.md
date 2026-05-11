## 1. Simple explanation
When you write C# code, you're not writing instructions for your CPU directly. You're writing instructions for a **virtual machine** called the **CLR (Common Language Runtime)**. .NET is a platform that sits between your code and the operating system.

Think of it like this:
> You write a recipe in English. A translator converts it to French. A chef reads the French and cooks the food.
- Your C# code = English recipe
- The compiler + IL = French translation
- CLR = The chef
- Your CPU = The kitchen

## 2. Real-World Analogy
Imagine a **universal power adapter**.
Different countries have different power sockets (Windows, Linux, macOS, ARM, x64). If you plug your laptop directly into the wall, you need a different laptop per country.
.NET is the adapter. You write code once, .NET handles making it work on every "country" (OS/CPU).
But there's more — this adapter is **smart**. It:
- Manages your memory (so you don't electrocute yourself)
- Optimizes how power flows (JIT compilation)
- Handles errors before they explode the circuit (exception handling)
- Recycles used power (Garbage Collection)

## 3. The Architecture - Full Stack Visual
![](../../assets/full_stack_architecture_dotnet_arch_drawio.png)

## 4. Technical Deep Dive
#### Layer 1: Your C# Code
You write `.cs` files. These are plain text files — your CPU cannot run these directly. A CPU only understands binary machine instructions. Something needs to translate.
#### Layer 2: The Roslyn Compiler — C# → IL
When you run `dotnet build`, the **Roslyn compiler** (`csc.exe` / `dotnet-csc`) reads your `.cs` files and produces a `.dll` or `.exe` file.
But — and this is critical — **this `.dll` does NOT contain native machine code**. It contains **IL (Intermediate Language)**, also called **CIL (Common Intermediate Language)** or historically **MSIL**.

**Why IL? Why not compile directly to machine code like C++?**
This is the genius of the design. IL is:
- CPU-architecture agnostic (one `.dll` runs on x64, ARM, x86)
- OS-agnostic (same `.dll` on Windows and Linux)
- Runtime-optimizable (JIT can optimize for the actual CPU at runtime)
- Verifiable (CLR can security-check the IL before running it)

If you compiled C# directly to machine code like C++, you'd need a different binary for Windows x64, Linux ARM, macOS x64, etc. IL solves this.
#### Layer 3: The CLR — The Heart of .NET
The **Common Language Runtime** is a virtual machine. When you run your app, the OS starts the CLR process. The CLR then:
1. Loads your `.dll` assembly
2. Reads the IL
3. **JIT compiles** it to native machine code
4. Executes the native code
5. Manages memory via the **GC**

**Why is CLR called "Common"?** Because C#, F#, VB.NET, and other languages all compile to the same IL format. The CLR doesn't know or care which language you wrote. They all share one runtime. This is called **language interoperability** — you can call a C# class from F# because they share the same IL and CLR.
#### Layer 4: JIT — Just In Time Compiler
Your IL is compiled to machine code **not ahead of time, not at build time** — but **at runtime, method by method, just before each method is first called**.

**How does JIT work internally?**
1. You call `MyMethod()` for the first time
2. CLR sees: "I have IL for `MyMethod`, but no native code yet"
3. CLR calls the JIT compiler on that method's IL
4. JIT produces native machine code optimized for **this specific CPU** (your Intel Core i9, or AWS Graviton ARM)
5. CLR replaces the method's stub with the new native code
6. Next time `MyMethod()` is called — the native code runs directly. JIT does NOT re-compile

**Why is JIT better than pre-compiling?**
- JIT knows your **actual CPU's capabilities** — it can use AVX-512 SIMD instructions on CPUs that support it
- JIT can **inline methods** it couldn't have known to inline at compile time
- JIT can **profile** which branches are taken most often and optimize for them (tiered compilation)

**Tiered Compilation (since .NET Core 2.1):**
- Tier 0: JIT compiles quickly with minimal optimization (fast startup)
- Tier 1: After a method is called many times ("hot path"), JIT recompiles it with aggressive optimization

This is why .NET apps get **faster the longer they run**.
#### Layer 5: The Garbage Collector (GC)
In C/C++, you `malloc()` and `free()` manually. Forget to free? Memory leak. Free twice? Crash. .NET's GC does this automatically.

**How does GC work internally (simplified)?**
.NET divides the heap into **generations**:
- **Gen 0** — newly allocated objects. GC runs here most frequently. Most objects die young (e.g., a temporary string in a loop)
- **Gen 1** — objects that survived Gen 0 collection
- **Gen 2** — long-lived objects (singletons, caches, static fields)
- **LOH (Large Object Heap)** — objects > 85KB, collected with Gen 2

**Why generations?** The "generational hypothesis" — most objects die young. By collecting Gen 0 frequently (it's small and fast), GC avoids scanning the entire heap.
#### Layer 6: BCL — Base Class Library
This is the standard library that ships with .NET. It's the thing that gives you:
- `System.String`, `System.Int32`, `System.Boolean`
- `System.Collections.Generic.List<T>`
- `System.IO.File`
- `System.Net.Http.HttpClient`
- `System.Threading.Tasks.Task`
The BCL is implemented partly in C# and partly in native code (for performance-critical paths). You can actually read the BCL source code on GitHub at `dotnet/runtime`.

## 5. .NET History - Why Does any of this exist?
|Year|What happened|
|---|---|
|2002|.NET Framework 1.0 released — Windows only, closed source|
|2002–2014|.NET Framework evolved (2.0, 3.5, 4.x) — still Windows-only|
|2016|.NET Core 1.0 — cross-platform, open source, rewritten from scratch|
|2019|.NET Core 3.x — can build Windows desktop apps on .NET Core|
|2020|.NET 5 — Microsoft dropped "Core", unified everything under ".NET"|
|2021–now|.NET 6, 7, 8, 9 — yearly releases, massive performance gains|
**Why did Microsoft rewrite .NET from scratch?**
.NET Framework was designed in the Windows era. It had deep Windows coupling (`System.Web`, `System.Windows.Forms` baked into the core). When cloud and Linux became dominant, Microsoft needed a version that could run in Docker, on Linux servers, on ARM — without carrying 20 years of Windows baggage.
.NET Core was a clean-room rewrite keeping the C# language and the IL/CLR concepts, but discarding the Windows-specific layers.

## 6. Common Mistakes / Misconceptions
**Misconception 1:** "C# compiles to machine code" → Wrong. C# compiles to IL. The CLR's JIT produces machine code at runtime.

**Misconception 2:** ".NET is slow because of JIT" → Wrong. JIT warm-up adds startup latency (milliseconds), but steady-state .NET is extremely fast. Benchmarks show .NET 8 competing with Go and sometimes C++ on throughput.

**Misconception 3:** "GC causes lag" → Partially true for Gen 2 collections, but .NET's GC has improved drastically. Gen 0 collections take microseconds. Server GC mode uses multiple threads.

**Misconception 4:** ".NET Framework and .NET Core are the same" → Completely different codebases. .NET Framework is legacy/Windows-only. Modern .NET (5+) is cross-platform.

---
### Cross-Questions for You

Here are your challenges — think carefully before reading further answers:

**Q1.** If IL is CPU-agnostic, and JIT compiles at runtime — what happens when you deploy a .NET app inside a Docker container to AWS Lambda (ARM)? Does your `.dll` need to be different than on x64? Think through the full flow.

**Q2.** .NET has AOT (Ahead of Time) compilation — it skips JIT entirely and compiles directly to native code at publish time. Given everything I explained about JIT's advantages... what do you think AOT _loses_ when it gains faster start-up? What's the trade-off?

**Q3.** A colleague says: _"I'll put all my objects in static fields so the GC never collects them and my app will be faster."_ What's wrong with this reasoning? What problems does it cause?

**Q4.** You have a C# method called 10 million times in a tight loop. The JIT compiles it on the first call. On the 5 millionth call, the CLR's tiered compilation kicks in and recompiles it. What happens to the calls happening between Tier 0 and Tier 1 compilation? Does the app pause?

**Q5.** F# is a functional language. C# is object-oriented. They both run on the CLR. Explain how this is possible — what does the CLR actually understand?

---

Tags:
#dotnet
