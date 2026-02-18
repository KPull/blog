---
layout: post
title: "A Springless Way of Thinking"
date: 2026-02-18 22:06:04 +0100
categories: software_development
tags: [java, spring, software development]
---

I've been programming with Java for as long as I can remember. I remember
picking up and reading through the entirety of Deitel's _Java How to Program:
2nd Edition_ in my childhood and being amazed how much more interesting it
was than Borland's _Turbo Pascal 7_ (although, I do miss that retro blue-yellow
colour scheme).

One of the things that I remember quite vividly is how "raw", for lack of a better
word, it was to program with Java in those days. One of the things I had learned
from the aforementioned book was how to use the operating system's socket
interface and create a very simple web server that responded to HTTP
requests.

Looking back, even though it was very verbose code (and I was very young), I
could understand, to a fairly high degree, what the program was doing. After
that chapter, the book went on to describe the Java Servlet API and that was
the last time I had ever used, discussed or even thought about the Socket APIs
within Java, ever again.

# Impressive Tech

When the Spring Framework came along and simplified the way we built
applications, I think everyone was impressed at how much simpler the
development process had gotten. It turned out, not too many software shops
actually required the "distributed transaction management" capabilities
offered by JBoss and other application servers; it was enough to have a
transaction manager to cover your database queries. With Spring, it was simple
to just configure your transaction manager (ie. still called Beans to this day)
within your application and get straight into implementing your business logic.
Unbeknownst to most people though, this was the birth of a dark pattern that
slowly started creeping into the Java ecosystem.

# Leaky Abstractions

One big selling point of the Spring Framework was that it provided a way to do
automatic transaction management using the `@Transactional` annotation.
Automatic transaction management using annotations was something that
was not uniquely available in Spring. However, I would argue that Spring did
popularise the pattern of annotating every single Service method with a
`@Transactional` annotation.

The problem I wanted to discuss in this blog post is not about the concept of
having automatic transaction management; rather, it's that the Java ecosystem
appears to be moving in a direction where frameworks and libraries are taking
away control, and, more importantly, **understanding** from the coder's hands.
And I could not think of a more widespread and well-known example than
Spring's `@Transactional` annotation. See if you can answer the questions
below.

At a high-level, most people can tell you that the annotation in question, in its
default configuration, will ensure that the annotated method will be executed
within some sort of transaction. But there's more to it than that.

* Will it be a new transaction or an already open one?
* If not an existing transaction, what happens to the running transaction? 
* If a new transaction is opened within an existing transaction, how does 
the application even achieve that (it acquires a new database connection, by the way)? 
* At what point is the transaction committed or rolled back?
* Does catching an exception in an outer method still
rollback the transaction?
* How does Spring even manage to do all of this
transaction management in the first place without modifying our code? 
* Why do we get database errors only after our method has returned?

Notice how this single, seemingly harmless, annotation has lowered our
understanding of our own code and how our systems behave. We cannot even
open the implementation of this annotation with our IDE to see what is going
on behind-the-scenes. These abstractions would be fine if we did not need 
to think anymore about the implementations that they hide. In practice, 
however, business requirements
constantly come up where knowing how these abstractions work, in ample detail, 
is important... because they do not behave exactly as we want them to.

These abstractions quickly become **leaky abstractions**.

# Better Alternatives

Because of what I described above, I find myself wondering whether weening
ourselves off of Spring is the right approach. With the advent of Spring Boot
the problem has become much worse. The programmer barely gets to write any
code and all of a sudden the application is doing countless different things that
were never intended.

In both Rust (specifically the _axum_ framework) and Python (specifically the
_FastAPI_ framework), abstractions are more explicit and. To me, it feels that
**understanding** happens more naturally, and is encouraged, in those languages.

For example, in Rust, I can ask for a macro (similar to Java's annotations) to be
expanded into the real generated code that will be passed to the compiler right
within the IDE's editor. Likewise, in Python, decorators are just functions that
take another function as an argument: if I wanted to know what it, all I have to 
do is open the decorator's function definition
(`CTRL`-Left Click it in PyCharm).

```python
from functools import wraps


# A decorator is simply a function that takes another function as a parameter
def transactional(f): # The function being decorated is 'f'
    @wraps(f)         # This is a useful error reporting utility for wrapped functions
    def wrapper(*args, **kwargs):
        try:
            transaction = database.start_transaction()
            result = f(*args, **kwargs) # Call the decorated function
            transaction.commit()
            return result
        except:
            transaction.rollback()
    return f

@transactional
def create_article():
    # Create an article within the transaction
    pass

# Decorating a function like above is simply syntactical sugar for the below.
# create_article() is now just the original function but with the transactional
# behavior wrapped around it
create_article = transactional(create_article)
```

I started with my story about Socket APIs because it's ultimately the most
explicit form of code you could get. Now obviously, I don't think we should be
removing all forms of abstraction; but, I do think that the Java ecosystem
should return to using more explicit forms of programming.

The antipattern of using extreme dependency injection and auto-configuration
is killing the joy of coding. In addition to this, frameworks like Spring Boot tend
to create very inefficient binaries even though the Java Virtual Machine's
performance, by itself, is close to that of natively compiled code.

There is a lot of value for companies and software teams to think about going
Springless when building and architecting their systems.

# Where to Start

If you wish to try your hand at this Springless way of thinking you could try:

* [Javalin](https://javalin.io/) - An imperative web server instead of annotation-based
  frameworks.
* Using your transaction manager imperatively by acquiring and managing the transaction explicitly.
* Creating the simplest abstraction you need within your application for metrics and logging.
