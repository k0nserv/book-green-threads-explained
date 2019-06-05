# Background information

This is the most technical part of this book but if we truly want to understand, we just have to go through it. I promise I will make it as fast and to the point as possible. We’ll soon enough move on to the code.

So here we go! First of all, we are going to interfere and control the CPU directly. This is not extremely portable since there are many kinds of CPU’s out there. The main ideas are the same, but a small part of the implementation details will differ.

We will cover one of the more commonly used architectures: x86-64.

**In this architecture the CPU features a set of 16 registers:**

![Click the picture to view an enlarged view](https://lh5.googleusercontent.com/qcybPMAwX9KpEBFFt86ioZDRREjn7MgdiOESIhTVNr4WOpjf-xvrBj2cF5XnIdjd0eUP27h_Ay-6wz_piQoCxQPJWzoi4Wmy0Z-pmUM13-Nefh8dYTFqiOf_wzAbgXmbVimKzJhF)

If you're interested you can find the rest of the specification here: [https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI](https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI).

Out of interest for us right now is the registers marked as “callee saved”. These are the registers that keep track of our context: the next instructions to run, the base pointer, the stack pointer and so on. We’ll get to know this more in detail later.

If we want to direct the CPU directly we need a some minimal code written in Assembly, fortunately we only need to know some very basic assembly instructions for our mission. How to move values to and from registers:

```text
mov     %rsp, %rax
```

#### A super quick introduction to Assembly <a id="docs-internal-guid-bc1ce7bf-7fff-2c5d-a4d5-c91055081781"></a>

First and foremost. Assembly language is not very portable, each CPU might have a special set of instructions, however some are common on most desktop computers today.

There are two popular dialects: AT&T dialect and Intel dialect.

The AT&T dialect is the standard when writing inline assembly in Rust, but in Rust we can specify that we want to use the “Intel” dialect instead if we want to. Rust mainly leaves it up to LLVM to deal with inline assembly, and the inline assembly for LLVM closely resembles the same syntax which is used when writing inline assembly in C. That makes it a lot easier to look at C inline ASM for learning since the syntax will be very familiar \(though not exactly the same\).

_We will use the AT&T dialect in our examples._

Assembly has a strong backwards compatibility guarantee. That’s why you will see that the same registers are addressed in different ways. Lets look at the \`%rax\` register we used as example above for an explanation:

```text
%rax    # 64 bit register (8 bytes)
%eax    # 32 low bits of the “rax” register
%ax     # 16 low bits of the “rax” register
%ah     # 8 high bits of the “ax” part of the "rax" register
%al     # 8 low bits of the “ax” part of the "rax" register
```

As you can see, this is basically watching the history of CPUs evolve in front of us. Since most CPUs today are 64 bits, we will use the 64 bit registers in our code.

We will go through a bit more of the syntax of inline assembly in the next chapter.

One more thing to note is that the stack alignment on x86-64 is **16 bytes.** Just remember this for later.

