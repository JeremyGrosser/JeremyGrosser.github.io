---
layout: post
title: "Should I learn Ada for embedded development?"
date: 2021-03-17 18:21:00 -0700
categories: ada
---
Originally posted in response to [this thread](https://www.reddit.com/r/embedded/comments/m74b4d/what_is_your_opinion_on_ada_have_you_used_it_for/) on Reddit.

I started learning Ada a few years ago and have been using it for the type of hobby projects where most people would reach for an Arduino or Teensy compatible thing. Like you, I was also seeking a safer alternative to C.

Learning Ada is not easy. It's a big language and the reference manual uses quite a bit of vocabulary that's specific to Ada, making it even harder for a beginner to comprehend. There are some excellent textbooks, but it's 2021 and most people learn by Googling things, not thumbing paper. Ada resources tend to be heavily weighted towards explaining with words rather than examples, which is again complicated by the vocabulary. Personally, I feel like I learned the most when I did the Advent of Code 2019 and 2020 puzzles in Ada. There's just no substitute for bashing your head against lots of different problems until it makes sense.

If you've ever skimmed any of the popular C coding standards, you'll find a lot of rules that boil down to "allocate on the stack and avoid pointers". When you do need a pointer, Ada has access types, which are fat pointers that associate a type and size with the underlying pointer and make it much harder to do unsafe things by accident. One of my favorite things is the concept of a `not null access` type, which is a pointer that must be assigned a value at instantiation and can never be assigned null. This effectively makes null pointer exceptions impossible.

"Safe" code is an overloaded term that means different things depending on what industry you work in. There is a subset of the Ada language called SPARK that allows you to specify enough constraints to be able to generate formal proofs of certain properties of your implementation. If you're building a machine that can kill people unintentionally, your code is probably subject to a standard like DO-178C or ISO 26262 that requires this level of verification. These days, most of those systems get modeled and verified in a program like LabVIEW which generates C or C++ code that conforms to the model. Having a toolchain that can verify that the model's constraints still hold for the actual implementation code is where Ada/SPARK shines.

If you're just trying to avoid buffer overflows, Ada makes it difficult to do dumb things. In most cases, your code won't compile if you write past the bounds of an array. Worst case, it'll raise an exception at runtime. If you really believe you know better than the compiler, you can disable the runtime checks and use an `Unchecked_Conversion` to change the size of the array. But if you're reaching for these switches, you probably need to rethink your decisions.

Record types are like C structs, with a very rich syntax for defining the memory layout of the struct members. These "record representation clauses" are probably the most useful construct for embedded programming as you can define bit fields in registers and buffers without ever needing to shift and mask bits, eliminating a large class of potential bugs.

Ada has namespaces, generics, objects, interfaces, polymorphism, collections, and iterators, just like you'd find in C++ or Java, but with more verbose syntax. "Annex A" of the language spec is effectively the standard library and is quite comprehensive. The language predates unicode, so it has the same issues as C conflating character encoding with size. There are several libraries that add new unicode string types, but the community hasn't centered around a single implementation yet.

There are many arguments against Ada that have been discussed ad nauseum for the last 20 years. One of the most persistent is that Ada compilers are expensive. GCC has had a GPL licensed Ada frontend called GNAT since the mid-90s. A company called AdaCore contributes the majority of patches to GNAT and sells a commercial version that provides better tooling for people that need to certify their code. Most applications will be just fine using the open source GPL licensed GNAT. It's already in most Linux distros, on Debian you can `apt-get install gnat`.

"It's hard to hire Ada programmers." This is true, but in my opinion points to a larger issue with organizations wanting to hire a candidate with every conceivable skill they will ever need, rather than investing time into training and education. Plenty of people would be happy to learn a new programming language given the opportunity. That said, there are several Ada developers lurking on reddit, freenode, and github that would be delighted to take a job writing Ada full time.

I know I didn't directly answer your question about Ada in embedded contexts specifically, but that's because I believe that you can write good embedded software in any language. The development process is more important than the language.

The Department of Defense put a bunch of experts in a room together to come up with requirements for a systems language. They iterated on these requirements a few times, then called for proposals for a language to fulfil those requirements. Three languages were proposed, discussed, and voted upon. The winning proposal was renamed to Ada. This same requirements -> proposals -> deliberation -> implementation flow is more familiar today as the waterfall process or V model. Process drives good software, the language is just a tool.
