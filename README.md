---
description: >-
  This book aims to explain green threads by using a small example where we
  implement a simple but working program where we use our own green threads to
  execute code.
---

# Introduction

{% hint style="info" %}
All the code we go through here is [located in a github repository](https://github.com/cfsamson/example-greenthreads). There are two branches, the `main` branch that only contains the code and the `commented` branch that contains the code with comments explaining what we do.
{% endhint %}

Green threads, userland threads, coroutines, goroutines or fibers, they have many names but for simplicity’s sake I’ll refer to them all as green threads from now on.

In this article I want to explore how they work by implementing a very simple example where we create our own green threads in 200 lines of Rust code. We'll be explaining everything along the way so our main focus here is to understand them and learn how they work by using simple, but working example.

{% hint style="info" %}
We will not use any external libraries or helpers and will do everything from scratch so we make sure we really understand what's going on.
{% endhint %}

### Who is this article for?

We are peeking down the rabbit hole in this article so if that sounds scary, this article probably isn’t for you. Just go back and live happily ever after.

If you are the curious kind and want to understand how things work, then read on. Maybe you’ve heard of Go and it’s goroutines, or the equivalent in Ruby or Julia and you know how to use them but want to know how they work - well then read on.

In addition, this should be interesting if:

* You’re new to Rust and want to learn more about its features.
* You have followed the discussions in the Rust community about async/await, the Pin-API and why we need generators. In this case I try to put all the pieces together in this article.
* If you want to learn the basics of inline assembly in Rust.
* If you’re just curious. 

Well, join me as we try to figure out everything we need to understand them.

You don’t have to be a Rust programmer to understand this article but it is highly recommended to read some of the basic syntax first. If you want to follow a long or clone the repo and play around with the code you should probably get Rust and learn the basics.

{% hint style="info" %}
 [You will find everything you need to set up Rust here.](https://www.rust-lang.org/tools/install)
{% endhint %}

### Following along

All the code I provide here is in a single file and has no dependencies which means that you can easily start your own project and follow along if you want to \(i suggest you do\). You can even run most of the code in the [Rust playground](https://play.rust-lang.org). Just remember to use the `nightly`version of the compiler.

### Portability and issues

Currently there is an issue I have with the `asm!`macro that doesn't compile in release mode. It seems to be related to the `"=m"`constraint I use in the inline macro. Even though we could work around this, I don't consider that a big problem since this is only an example.

 I've filed an issue about it in the Rust repo, so we'll wait and see if we get a fix for it.

I've tested the code on both OSX and Windows, I have not tested it under Linux. Please file an issue [in the repository](https://github.com/cfsamson/example-greenthreads) if you find any issues on Linux.

### Disclaimer <a id="docs-internal-guid-12e6c217-7fff-3de7-4bee-4532b47ef574"></a>

I’m not trying to make a perfect implementation here. I’m cutting corners to get down to the essence and fit it into what was originally intended to be an article but expanded into a small book. This is not the best way of displaying Rusts greatest strengths, its safety guarantees, but it does show an interesting use of Rust and the code is mostly pretty clean and easy to follow.

However, if you spot places where I can make the code safer without making it significantly more complex, I welcome you to create an issue in [the repo](https://github.com/cfsamson/example-greenthreads) or even better, a pull request.

### Credits

[Quentin Carbonneaux](https://github.com/mpu) wrote an [nice article](https://c9x.me/articles/gthreads/intro.html) back in 2013 which I used as inspiration for the main code example.

