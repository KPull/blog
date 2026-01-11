---
layout: post
title:  "What Rust Taught me about Invariants"
date:   2026-01-05 21:03:46 +0100
categories: software_development
tags: [rust, java, programming, software development]
---

Over the past year and a half, I dove deep into learning [Rust](http://rust-lang.org), the systems programming language that is consistently the most loved language in [StackOverflow's Developer Survey](https://survey.stackoverflow.co/2025/technology#2-programming-scripting-and-markup-languages). Learning Rust has helped me elevate my programming skills to the next level, even in other languages.

To begin with, let me point out that my main focus has always been Java and Python. Until now, I always treated classes in Java as a way to combine related data and behaviour into a single convenient container. This is, after all, how object-oriented programming concepts are introduced to budding programmers. But in Rust, developers are encouraged to use types to **prove** program invariants. This knowledge has changed how I treat types in Java.

An invariant is a condition that must hold true at a certain point in the program. If we can prove that an invariant holds true, managing the different states our program could be in becomes more managable. You no longer need to consider any states for which the invariant does not apply. The states that remain should be easier to test and reason about, reducing the chance that we inadvertently introduce nasty bugs.

For example, let us say we are developing an API endpoint that collects the details for a new user. The user must input a username that is between three and twenty characters long and must include only lowercase characters. To represent this invariant, in Rust, we could write something like the following:

```rust
pub struct Username(String);

impl Username {
    pub fn validate(username: &str) -> Result<Username, &str> {
        if username.len() < 3 || username.len() > 20 {
            return Result::Err("Username length must be between 3 and 20")
        }
        if username.chars().any(|c| !c.is_alphabetic() || !c.is_lowercase()) {
            return Result::Err("Username must only contain lowercase alphabetic characters")
        }
        Ok(Username(username.to_string()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}
```

The type `Username` now represents a program invariant. Anywhere we have an instance of a `Username`, we _know_ that the string follows the constraints set out earlier and we don't need to think about the possibility of getting a string outside those boundaries.

Previously, in Java, I would not have thought twice about writing something like the following for implementing the API's business logic.

```java
public void register(String username, String fullName, String country) {
    // Write the user into the database
}
```

Notice how the `username` parameter is defined as a `String`. When this function is invoked, I have no guarantee that the input string has been validated by the system. Typically anyone writing Spring applications would have validated the input using a [JSR-303 Bean Validator](https://beanvalidation.org/1.0/spec/) like the `@Size` ([JavaDocs](https://jakarta.ee/specifications/bean-validation/3.0/apidocs/jakarta/validation/constraints/size)) and `@Pattern` ([JavaDocs](https://jakarta.ee/specifications/bean-validation/3.0/apidocs/jakarta/validation/constraints/pattern)) annotations but anyone, in the future, could easily call the `register(..)` function without passing through the JSR-303 validators. A possible solution would be to validate the input within the `register(..)` function itself but if I pass the data downstream to a different module, that module has the same problem that I have here (that module has no guarantee I validated the data).

We can take the ideas we learned from Rust and apply them here by implementing the `Username` class as follows:

```java
public final class Username {
    private final String value;

    public Username(String value) {
        Objects.requireNonNull(value);

        if (value.length() < 3 || value.length() > 20) {
            throw new IllegalArgumentException("Usernames must contain between 3 and 20 characters");
        }

        for (int i = 0; i < value.length(); i++) {
            char c = value.charAt(i);
            if (!Character.isAlphabetic(c) || !Character.isLowerCase(c)) {
                throw new IllegalArgumentException("Usernames must contain only alphabetic and lowercase characters");
            }
        }

        this.value = value;
    }

    public String value() {
        return value;
    }
}
``` 

As we can see, there is no way to ever construct an instance of `Username` and obtain a reference to it without having a username string that matches the constraints we had earlier.

Then, we can modify the `register(..)` function's signature like so:

```java
public void register(Username username, String fullName, String country) {
    // Write the user into the database
}
```

With this simple change we have the **guarantee** that, when this method is invoked, the `username` passed in will _always_ have a string value that conforms to the requirements. To fully test this function, we can achieve full coverage by considering only[^1] correctly constructed instances of `Username`. For cases that break the invariant, the program would not compile.

[^1]: Unfortunately, in Java, we also have to consider the case where `username` is `null`. Other languages, like Rust and Kotlin, have eliminated this possibility.

With the above, working with the `register(..)` function should be easier. Beyond this, in my opinion, the biggest advantage is that we made our intentions _explicit_. It is now very clear what our intentions for the function's inputs were. If we had kept the parameter type as a `String`, we would have condemned anyone reading the code, one or two years down the line, to hunt for this information, either in the business requirements documentation or some other place. And that by itself, I think, is worth the extra effort that goes into creating these semantically-rich types.

# Serialization

A small note for anyone wanting to take this further if they use Jackson or Hibernate:

We can use the same `Username` type within the API message classes and entities. For Jackson, we can use the `@JsonValue` ([JavaDocs](https://fasterxml.github.io/jackson-annotations/javadoc/2.20/com/fasterxml/jackson/annotation/JsonValue.html)) and `@JsonCreator` ([JavaDocs](https://javadoc.io/doc/com.fasterxml.jackson.core/jackson-annotations/2.20/com/fasterxml/jackson/annotation/JsonCreator.html)) annotations within the `Username` class. For Hibernate, we can add a JPA `Converter` ([JavaDocs](https://jakarta.ee/specifications/persistence/2.2/apidocs/javax/persistence/converter)) to directly read and write `Username` types from and to the database.
