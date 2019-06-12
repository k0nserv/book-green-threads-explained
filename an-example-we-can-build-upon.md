# An example we can build upon

{% hint style="info" %}
In this example we will create our own stack and make our CPU return out of it's current execution context and over to the stack we just created. We will build on these concepts in the following chapters \(we will not build upon the code though\).
{% endhint %}

### Setting up our project

First let’s start a new project in a folder named “green\_threads”. Run:

`cargo init`

We need to use the Rust Nightly since we will use some features that are not stabilized yet:

`rustup override set nightly`

In our `main.rs` we start by setting a feature flag that lets us use the `asm!`macro:

{% code-tabs %}
{% code-tabs-item title="main.rs" %}
```rust
#![feature(asm)]
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Lets set a small stack size here, only 48 bytes so we can print the stack and look at it before we switch contexts:

{% code-tabs %}
{% code-tabs-item title="main.rs" %}
```rust
const SSIZE: isize = 48;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Then lets add a struct that represents our CPU state. We only focus on the register that stores the "stack pointer" for now since that is all we need:

{% code-tabs %}
{% code-tabs-item title="main.rs" %}
```rust
#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    rsp: u64,
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

In later examples we will use all the registers marked as "callee saved" in the specification document i linked to. These are the registers described in the x86-64 ABI that we'll need to save our context, but right now we only need one register to make the CPU jump over to our stack.

Note that this needs to be `#[repr(C)]` because we access the data the way we do in our assembly. Rust doesn't have a stable ABI so there is no way for us to be sure that this will be represented in memory with `rsp` as the first 8 bytes. C has a stable ABI and that's exactly what this attribute tells the compiler to use. Granted, our struct only has one field right now but we will add more later.

{% code-tabs %}
{% code-tabs-item title="main.rs" %}
```rust
fn hello() -> ! {
    println!("I LOVE WAKING UP ON A NEW STACK!");

    loop {}
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

For this very simple example we will define a function that just prints out a message and then loops forever:

Next up is our inline assembly where we switch ower to our own stack.

```rust
unsafe fn gt_switch(new: *const ThreadContext) {
    asm!("
        mov     0x00($0), %rsp
        ret
       "
    : 
    : "r"(new)
    :
    : "alignstack" // it will work without this now, will need it later
    );
}
```

We use a trick here. We write the address of the function we want to run on our new stack. Then we pass the address of the first byte where we stored this address to the `rsp` register \(the address we set to `new.rsp` will point to _an address located on our own stack that leads to the function above_\). Got it? 

The `ret` keyword transfers program control to the return address located on top of the stack. Since we pushed our address to the `%rsp`register, the CPU will think that is the return address of the function it's currently running so when we pass the `ret`instruction it returns directly into our own stack.

The first thing the CPU does is read the address of our function and runs it.

### Quick introduction to Rusts inline assembly macro

If you haven't used inline assembly before this might look foreign, but we'll use an extended version of this later to switch contexts so I'll explain what we're doing line by line:

`unsafe` is a keyword that indicates that Rust cannot enforce the safety guarantees in the function we write. Since we are manipulating the CPU directly this is most definitely unsafe

```rust
gt_switch(new: *const ThreadContext)
```

Here we take a pointer to an instance of our `ThreadContext` from which we will only read one field.

```rust
asm!("
```

This is the `asm!`macro in the Rust standard library. It will check our syntax and provide an error message if it encounters something that doesn't look like valid AT&T \(by default\) Assembly syntax.

The first thing the macro takes as input is the assembly template:

```rust
mov     0x00($0), %rsp
```

This is a simple instruction that moves the value stored at `0x00` offset \(that means no offset at all in hex\) from the memory location at `$0` to the `rsp`register. Since the `rsp`register stores a pointer to the next value on the stack, we effectively push the address we provide it on top of the current stack overwriting whats already there.

You will not see `$0` used like this in normal assembly code. This is part of the assembly template and is a placeholder for the first parameter. The parameters are numbered from 0, 1, 2... starting with the `output`parameters and then moving on to the `input`parameters. We only have one input parameter here which corresponds to `$0`. 

If you encounter $ in assembly it most likely means an immediate value \(an interger constant\) but that depends \(yeah, the $ can mean different things between dialects and between x86 assembly and x86-64 assembly\).

```rust
ret
```

The `ret`keyword instructs the CPU to pop a memory location off the top of the stack and then makes an unconditional jump to that location. In effect we have hijacked our CPU and made it return to our stack.

{% code-tabs %}
{% code-tabs-item title="output" %}
```rust
: 
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Inline ASM is a bit different from plain ASM. There are four additional parameters we pass in after the assembly template. This is the first one called `output` and is where we pass in our output parameters and are parameters we want to use as return values in our Rust function.

{% code-tabs %}
{% code-tabs-item title="input" %}
```rust
: "r"(new)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The second is our `input` parameter. the `"r"` literal is what is called a `constraint` when writing inline assembly. You can use these constraints to effectively direct where the compiler can decide to put your input \(in one of the registers as a value or use it as a "memory" location for example\). `"r"`simply means that this is put in a general purpose register that the compiler chooses. Constraints in inline assembly is a pretty big subject themselves, fortunately we have pretty simple needs.

{% code-tabs %}
{% code-tabs-item title="clobber list" %}
```rust
:
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The next option is the `clobber` list where you specify what registers the compiler should not touch and let it know we want to manage these in our assembly code. If we pop any values of the stack we need to specify what registers here and let the compiler know so it know it can't use these registers freely. We don't need that here since we return to a brand new stack.

{% code-tabs %}
{% code-tabs-item title="options" %}
```rust
: "alignstack"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The last one is our `options`. These are unique for Rust and there are three options we can set: `alignstack`, `volatile` and `intel`. I'll refer you [to the documentation to read about them](https://doc.rust-lang.org/unstable-book/language-features/asm.html#options) since they're explained there. Worth noting is that we need to specify the "alingstack" for the code to work on Windows.

### Running our example

```rust
fn main() {
    let mut ctx = ThreadContext::default();

    let mut stack = vec![0_u8; SSIZE as usize];
    let stack_ptr = stack.as_mut_ptr();

    unsafe {
        std::ptr::write(stack_ptr.offset(SSIZE - 16) as *mut u64, hello as u64);
        ctx.rsp = stack_ptr.offset(SSIZE - 16) as u64;
        gt_switch(&mut ctx);
    }
}
```

So this is actually designing our new stack. `hello` is a pointer already \(a function pointer\) so we can cast it directly as an `u64` since all pointers on 64 bits systems will be, well, 64 bit, and then we write this pointer to our new stack. 

{% hint style="info" %}
We'll talk more about the stack in the next chapter but one thing we need to know already now is that the stack grows downwards. If our 48 byte stack starts at index _0_, and ends on index _47,_ index _32_ will be the first index of a 16 byte offset from the end of our stack.
{% endhint %}

Make note that we write the pointer to an the offset of 16 bytes from the base of our stack \(remember what I wrote about 16 byte alignment?\). 

We cast it as a pointer to an `u64` instead of a pointer to a `u8`. We want to write to position 32, 33, 34, 35, 36, 37, 38, 39 which is the 8 byte space we need to store our `u64`. If we don't do this cast we try to write an u64 only to position 32 which is not what we want.

We set the `rsp` \(Stack Pointer\) to _the memory address of index 32 in our stack_, we don't pass the value of the `u64`storead at that location but an address to the first byte.

When we `cargo run` this code we get:

```text
Finished dev [unoptimized + debuginfo] target(s) in 0.58s
Running `target\debug\green_thread_start.exe`
I LOVE WAKING UP ON A NEW STACK!
```

OK, so what happened? We didn’t call the function `hello` at any point but it still was run. What happened is that we actually made the CPU jump over to our own stack and execute code there. We have taken the first step towards implementing a context switch.

In the next chapters we will talk about the stack a bit before we implement our green threads, it will be easier now that we have covered so much of the basics.

[You can have a look at the full code here if you want to run it.](https://gist.github.com/cfsamson/4b10bd4e1e828f71405418513cc5b880)



