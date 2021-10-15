# Why Reliascript

As software engineers, we have come to believe bugs are a fact of life.
Humans make mistakes. Third parties violate their contracts. Library
maintainers make breaking changes without notice. The network fails.
The hardware fails. The software fails. Systems behave in complex
and often unpredictable ways, and then fail.

An enormous amount of software engineering effort is dedicated to 
making software systems work reliably. In general, it’s a very hard
thing to accomplish. You already knew that.

Reliascript was made to make reliable code easier to write. What if
you could write code without bugs? Sounds impossible right? Well 
ordinarily, it probably is. But now enter Reliascript. Reliascript 
has been designed from the ground up to encourage you to write bug
free code. It has been engineered with a new set of constructs built
into the language, all of which are designed to make it as difficult
as possible to write broken code.

In the following overview, we will dive into some of the unique
features of Reliascript that allow you to write bug free code.
Reliascript has been engineered with the following high level design
principles:

1. Build best practices into the language, and make it hard not to follow them
2. Catch more errors during compilation and fewer at runtime
3. Declaring properties you want is better than coding them imperitively
4. Focus on web and distributed systems programming
5. Devs cost more then servers. Sacrifice performance for reliability

# A Next Generation Type System

Type systems are a core part of many programming languages. Over
the years, type systems have gone in and out of fashion many times.
The developer community has oscillated back and forth between
statically typed and dynamically typed languages about every 10 years.

A goal of Reliascript is to catch as many bugs as possible at compile
time. Therefore, we naturally mesh with the statically typed camp. A
wrong type is a wrong type. But when designing Reliascript, we naturally
asked ourselves - could we go even further? A second design goal of
Reliascript is to focus on declaration over writing code. 

With these things in mind, Reliascripts type system goes way beyond
what you may find in other programming languages. Reliascript takes
concepts from data validation that are present in many web
and database frameworks, and bakes it directly into the language.
Data validation is no longer just an extra thing - its an intrinsic
part of writing Reliascript.

Additionally, Reliascript introduces concepts like Constraints, Kinds,
Tags, and Traits, to help you catch even more bugs and make it harder
still to make mistakes. In the following sections, we will discuss
further what these things are. But here is the basic summary.

1. Patterns are a set of properties that a value must always take on.
   Patterns are enforced using both compile time checks and data
   validation at runtime.
2. Constraints are functions or Regexs that can be used as part of a pattern
   to ensure that a value is valid. Constraints are enforced with data validation
   at runtime.
3. Kinds are like types in conventional statically typed languages.
   They are enforced entirely at compile time. Unlike traditional languages
   though, you can apply a Kind to even simple values like integers or strings
   which might otherwise have the exact same set of patterns and constraints.
4. Tags are like data validations in traditional languages. They are computed
   strictly at runtime using lazy semantics, but may be set as required in 
   certain sections of code, triggering them to be 
5. Traits are sort of like additional metadata that may be associated with
   the fields in your schema (or your schema as a whole). Traits are used
   for all sorts of useful things - like triggering a particular field to
   be sanitized if it is ever logged, defining what fields are publicly visible
   through your API and which ones are private, or enforcing role based access
   schemes to modifications on certain fields.

These 5 systems collectively define the Reliascript type system. Now lets go
into more details on each of these concepts.

## What is a pattern?

A pattern is some sort of limit on the values that a particular variable
can take. Let's look at the simplest possible pattern - the type check.

```
pattern MyString {type is string};
```

With this statement, we have defined a new pattern. The pattern
checks only a single thing - that the value of a particular variable is
a string. Nothing all that shocking here. But now lets look at something
more interesting.

```
pattern ShortString {
    type is string;
    length < 10;
}
```

Now what do we have here? Now we’ve defined a pattern that a particular
value is not just a string, but it's a string with a length less than 10!
This pattern will be enforced both with static type checks and runtime
type checks, to ensure there is no possible way ShortString could ever
take on a value longer than 10 characters. Let's try to use this variable
in action:

```
const value: ShortString;
value = “imshort”; // This is fine, because imshort is only 7 characters long.

value = “im really long!”; // This will give you a compiler error
```

But wait, I hear you say! This will never work! What about user input?
How can we possibly enforce string length at the compiler level, when 
users can type in whatever they want? Good question.

When a user provides input, it is usually in the form of an unconstrained
or partially constrained value. The act of validating the user's input
is called constraining the value, and must be done explicitly (TODO: 
should we allow automatic inference of what patterns to apply?). 
Let's see this in action:

```
import io;
const value = io.readUserInput(); // Value is now constrained only by {type: string}

const shortValue: ShortString;
shortValue = value; // This will lead to a compiler error, because value does not have the length pattern imposed on it. Instead do the following:

shortValue = value constrained by ShortString; // This will trigger runtime data validation. If value does not meet the patterns imposed upon it by ShortString, then an exception is raised.
```

You can also constrain values in place, without having to declare a whole new value.

```
import io;
const value = io.readUserInput(); // Value is now constrained only by {type: string}

constrain value with ShortString;

const shortValue: ShortString;
shortValue = value; // This works fine now, because value has been previously constrained, altering its type.
```

It’s important to note that the patterns themselves live independently
of the name they are given (this contrasts with Kinds described next, which
are statically enforced) E.g. the following code is also acceptable, even
though we don’t explicitly constrain the value with ShortString.

```
import io;

const value = io.readUserInput(); // Value is now constrained only by {type: string}

constrain value with {length < 10};

const shortValue: ShortString;
shortValue = value;
```

As you can see, Reliascript's type system introduces a completely in-built way
of performing data validation, by bringing it into the type system.

### Varieties of Patterns

Reliascript has a huge number of built-in patterns meant to enable
static type checking in an extremely wide variety of scenarios.
Here is a non-exhaustive list:

1. What type a particular value is, e.g. date, string,
2. String length minimums and maximums
3. Whether a value is required or optional (e.g. its value can be null)
4. Whether a string matches a particular constraint, such an email, phone number or ip address
5. Whether a number is between a given minimum or maximum value
6. Whether a particular value is supposed to change or be updated at the same time as another value (more on this later)

## Constraints

[TODO] Should constraints be used to describe all possible validations that can be applied? including minimums and maximums?

Constraints are used to describe in more detail what values are valid for 
a particular variable. Imagine we wanted to have a string to represent
an email address, and we want to ensure that it can never contain
anything that isn’t a valid email address. Reliascript allows you
to do the following:

```
const value: {
    type is string;
    matches constraint email;
}

value = “cheese!”; // This will give you a compiler error, because the given string does not match the email constraint

value = “cheese@example.com”; // This is allowed, because the given value matches the email constraint

```

Under the hood, constraints take one of two forms:
1. A regular expression which must match the string
2. A function which returns a boolean value on whether a given value meets the desired criteria

The constraint matching pattern provides you with a powerful way to extend your type system. 
In other programming languages, this might have been accomplished by creating a subclass of
some built-in string class with your added validation. Then you could rely on the existing
type enforcement semantics of your compiler.

In Reliascript, data validation has become a first class citizen within the language,
enforced with a combination of static compile time and runtime checks. 
As a consequence, it becomes downright impossible to assign a bad value into a variable.
At the same time, you can avoid having data validation getting duplicated in hundreds
of places throughout your code, as some extra cautious teams sometimes do. 
When the type system has seen a particular value has already been tested against
a pattern, it doesn’t need to repeat it.

## Kinds

Kinds are a little bit closer to what you might see in a traditional type system.
Where as Constraints are meant to define the shape that your data takes at a low-level,
a Kind is meant to imply more of a high level interpretation of the data. A Kind
can be used to distinguish between two different groups of variables which can 
otherwise take on similar values. For example, you don't want to confuse an 
integer describing a process PID with an integer describing a timeout in milliseconds.
They are both integers, but they aren't the same.

When you constrain a variable to a Kind, all values assigned to it must be explicitly
given that Kind. Imagine, for example, you have some email marketing software.
You may have a common Email pattern, which you reuse between a bunch of
different models within your application. However, you really don’t want to
confuse the email address of the user, e.g. their login email address, with
the email addresses of the contacts they are sending emails to. For this, you
can use kinds!

```
// We define a common email address pattern, used for all email addresses in the system
pattern CommonEmail {
    type is string;
    matches constraint email;
    length <= 10;
    length > 0;
}

// Now, we define two different kinds. One is for the users email, the other is for a contacts email
kind UserEmail;
kind ContactEmail;

pattern UserEmail {
    constrained by CommonEmail;
    kind of UserEmail;
}

pattern ContactEmail {
    constrained by CommonEmail;
    kind of ContactEmail;
}

const value: UserEmail;
value = “genixpro@gmail.com”;

Const secondValue: ContactEmail;
secondValue = value; // Yields a compiler error, because value has a kind of UserEmail, 
                     // and secondValue has kidn of ContactEmail;
                     // Although they are technically identical in the patterns they match, 
                     // because they have different Kinds, so the compiler will print an error
```

## Tags

A tag in some sense is the opposite of a Kind. Tags are focused mostly on runtime enforcement. 
A value may or may not have a specific tag associated with it, depending on the value it 
takes on at runtime. 

This is a lot like having a pattern which is not always enforced. When the pattern
matches, the value takes on the given tag. When the pattern fails, the value does
not take on the given tag.

If you define a function that requires a variable to have a specific tag, the compiler
will automatically introduce the associated runtime data validation to ensure the
value meets all of the required patterns to be given that tag. 

TODO: describe pristine / invalid tags here

## Traits

A trait is like a piece of metadata associated with a particular variable, pattern, 
or kind. Traits can be used for a variety of different forms of metaprogramming, 
by giving you a declarative way to describe the behaviours, rules, and properties
you want your data to have.

Traits in Reliascript have nothing to do with "Traits" used in some object-oriented
languages like Scala, which are sometimes also called "Mixin classes". A trait in
Reliascript is just a property that is statically declared and associated with a
type.

Traits are used for all sorts of goodies, and are a key feature that make Reliascript
so easy to write. Examples of what you can use traits for:

1. Defining whether a particular value contains PII or personally identifiable 
   information, and therefore should be sanitized if that value is ever printed
   to logs.
2. Giving extra information that allows your classes values to be serialized using
   tools like Google Protocol Buffers or Apache Thrift
3. Declaring whether a variable should be visible on public API endpoints,
   or should just be kept private.
4. Defining the RBAC permissions that should be enforced for a particular field -
   who is allowed to read it? Who is allowed to modify it?
5. Declaring whether an index should be created in the database for a field
   or set of fields.
6. Declaring whether a value is read-only
7. Declaring whether a value can be automatically computed based on other values
   (but should be stored in the database anyhow for various reasons like querying)

# Execution Environment

## Contexts & Stack-local variables

## Asynchronous Programming

### Timeouts

### Deadline Propagation

### Automatic Retry

# Reliable Distributed and Server Programming

One of the most difficult things to get right when your engineering a distributed,
cloud based system is how the heck do you ensure reliable execution of your code
across a multitude of different networked services, which may include different
machines in different data centers, third party vendors, making multiple
concurrent writes to a database, etc… when all of these things can potentially
fail.

Over the years, software engineers have thought up a wide variety of different
mechanisms to try and ensure a system’s reliable execution. First starters, we
have the automatic retry - if we make a network request and it fails, just try
it again. Then we have transactions in databases - a set of changes to the
database that must all succeed or all fail, without any partial changes. Then
we have the even more modern version, the Saga - a set of concrete steps to be
executed, each with forward code and associated rollback / reversal code that
should get executed in case a subsequent step in the saga fails. Then we have
two phased commits, atomic and idempotent operations, and a whole host of
other intellectual contraptions.

Like with data validation, Reliascript has been designed to take these
contraptions from being implemented in libraries on top of the language, to
being features of the language itself. With this comes a whole host of
safety benefits that can be enforced and automated with the
Reliascript compiler.


A reversible step - a change that has a forward step and a backward rollback

```
reversible DoTransaction(user, account, amount) {
    forward {
        const transaction = new Transaction({
            user,
            account,
            amount
        });

        save transaction;
    }
    rollback {
        const transaction = new Transaction({
            user,
            account,
            -amount,
        });

        save transaction;
    }
}

reversible SubtractBalance(account, amount) {
    forward {
        const account = Account.findOne({account});
        account.balance -= amount;
        save account;
    }
    rollback {
        const account = Account.findOne({account});
        account.balance += amount;
        save account;
    }
}
```

Note - ideally the rollback for a reversible can be automatically created,
so the person doesn’t need to explicitly write their own reversible.

An idempotent unit - this is a function that comes in one or two parts -
the guard and the body.

```
idempotent RecordTransaction(user, account, amount) {
    guard {
        if(transactions.count({user, account, amount}) > 0) {
            return false;
        }
        return true;
    }
    body {
        const transaction = new Transaction({
            user,
            account,
            amount
        });

        save transaction;
    }
}
```
## Saga - automatically ties together multiple different reversible steps in an automated manner.

```
saga MakeTransaction(user, account, amount) {
    DoTransaction(user, account, amount);
    SubtractBalance(account, amount);
}

import data;

type Person in memory {
    firstName: {
        type is string;
        length < 100;
        length > 0;
        always exists;
        changes with lastName;
    }
    lastName: {
        type is string;
        value.length < 100;
        Value.length > 0;
        always exists;
        changes with firstName;
    }
    middleName: {
        type is string or null;
        value.length < 100;
        sometimes exists;
        changes with firstName, lastName;
    }
    email: {
        type is string;
        value.length < 250;
        always exists;
        value matches constraint data.email;
    }
}


function main() {
    const person = new Person({
        firstName: “Brad”,
        lastName: “Arsenault”,
        middleName: “”,
        email: “test@example.com”
    });

    save person;
}
```

# Desirable Properties (in progress)


1. Data validation is a first-class citizens
   1. Goes beyond just type checking although that’s part of it 
   2. As many patterns as possible can be statically checked at compile time 
   3. Can infer patterns automatically based on actual usage of variables 
2. Data serialization is a first class citizen
   1. All data is by default persisted unless otherwise specified [Maybe?]
   2. Data is meant to be stored - no distinction between classes/objects for
      in-memory usage and classes/objects to be stored to a database - this
      is given as a parameter of the class / object. Should be no need for an ORM
3. Properties associated with data are more declarative and less imperative
   1. Whether or not a variable is meant to be serialized and returned via a public API
   1. Whether or not a variable needs to be sanitized in log messages
   1. Whether a variable is private to a class or public
   1. Whether a variable is indexed, is unique, immutable, read only
   1. Whether a variable can only be changed or read by functions that have certain declarations
   1. Whether a variable is just a cached copy of data that could be derived from other variables
   1. Whether a variable should always be updated at the same time as another variable
   1. Whether arrays are supposed to be sorted in a particular order, or are completely unsorted
   1. Whether a variable or function is deprecated
4. Idioms that are common in live systems to enhance reliability need to be first-class citizens
   1. Ways to make operations atomic
   2. Ways to make operations idempotent
   3. Automatic retries from failures
   4. Timeouts and deadline propagation
   5. Automatic restarts on crash
   6. Message passing, queuing, etc..
   7. Asynchronous programming, promises, async/await etc..
   8. Auditability / history of what happened
   9. Mocking?
   10. Auto test data generation?
   11. DB Transactions as first class citizens?
   12. Request ID?
   13. Automatic logging of the entire call path?
   14. Forward and backwards data compatibility?
   15. Status codes? Error messages?
   16. Environment Configuration? (ideally dynamic configuration values can be
       treated as static and can then have patterns enforced)

Other notes:
1. Like how java forces you to catch all possible exceptions a method can produce or bubble it up
2. No global variables or fake global variables that are just embedded into singletons that are passed everywhere in the application
3. Really like immutability, but wonder how to make it as convenient for programming as mutable programming
4. Needs mechanisms to be reliable even when working with APIs, services, code, programmed in other languages and provided by third parties
5. Testing needs to be first class citizen and helped by the language as much as possible
6. Like null checking and null type enforcement
7. Automatic time complexity calculation? (is this even possible?)
8. Aspect programming?
9. Java annotation?




