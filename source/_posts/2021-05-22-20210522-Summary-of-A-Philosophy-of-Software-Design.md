---
title: 20210522 Summary of A Philosophy of Software Design
date: 2021-05-22 11:04:03
tags: Book
---

This is my summary after reading the book: A Philosophy of Software Design.

Write for humans, not machines. You should treat yourself as a reader, no less than a writer. The more you think about as a reader, the easier it’ll be when you need to modify the code in future. Keep an investment mindset.

# Complexity

Complexity is anything related to the structure of a software system that makes it hard to understand and modify the system.

Symptoms of complexity:

- Change amplification: a seemingly simple change requires code modifications in many different places
- Cognitive load: how much a developer need to know in order to complete a task
- Unknown unknowns: it is not obvious which pieces of code must be modified to complete a task (Worst)

Causes of complexity:

- Dependencies: lead to change amplification and cognitive load
- Obscurities: when important information is not obvious; lead to cognitive load and unknown unknowns

Two ways to fight complexity:

1. Make code simpler and more obvious
2. Encapsulate it (modular design)

# Modular design

Module = interface + implementation

Deep module = simple interface + powerful functionality(implementation)

Shallow module: whose interface is complicated relative to the functionality. It doesn’t help much against complexity.

How to create deep modules:

- Information hiding
  - Information hiding can often be improved by making a class slightly larger. (Large class or more lines of code doesn’t always mean more complex, long methods aren’t always bad)
- Pull complexity downwards
  - You should strive to make life as easy as possible for the users of your module, even if that means extra work for you



Question: write general-purpose module or special-purpose module?

Answer: “some what generous-purpose”, the module’s functionality should reflect current needs, but the interface should be general enough to support multiple uses.

# Exception handling

- Define errors (and special cases) out of existence
- Mask exception: detect and handle exception at a low level
- Exception aggregation: handle all exceptions in one higher place with a single handler

# Design it twice

# All about comments

Comments can reduce cognitive load and unknown unknowns

- Comments should describe things that aren’t obvious from the code
  - Don’t repeat the code: use different words in the comment from those in the name of the entity being described
  - Lower-level comments add precision
  - Higher-level comments enhance intuition
  - Interface documentation: provide information that someone needs to know to use a class or method, not information about implementation
  - Implementation comments: inside methods, explain what, why, not how
- Write the comments first, use it as a design tool
- Comments belong in the code, not the commit log
