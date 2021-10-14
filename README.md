## Why Reliascript

As software engineers, we have come to believe flaws are a fact of life. Humans make mistakes. Third parties violate their contracts or make breaking changes without notice. The network fails. The hardware fails. An enormous amount of software engineering effort is dedicated to making software systems work reliably. In general, it’s a very hard thing to accomplish.

Reliascript was made to make reliable code easier to write. What if you could write code without bugs? Sounds impossible right? Well ordinarily, it probably is. But now enter Reliascript. Reliascript has been designed from the ground up to encourage you to write bug free code. It has been engineered with an entirely new set of constructs built into the language, all of which are designed to make it as difficult as possible to write broken code.

In the following overview, we will dive into some of the unique features of Reliascript that allow you to write bug free code.

A Next Generation Type System

Reliascript has a lot of unique constructs that can take some getting used to. The first and most prominent of these constructs is the type checking system. Like many programming languages, Reliascript is statically typed. However, Reliascript type system goes way beyond what you may find in other programming languages. Reliascripts type system is based around the concept of Constraints. Reliascript takes concepts from data validation features that are present in many web and database frameworks, and bakes it directly into the language. Data validation thus is a first class citizen.

## What is a constraint?

A constraint is some sort of limit on the values that a particular variable can take. Lets look at the simplest possible constraint - the type check.

```
constraint MyString {type is string};
```

With this statement, we have defined a new constraint. The constraint checks only a single thing - that the value of a particular variable is a string. Nothing all that shocking here. But now lets look at something more interesting.

```
constraint ShortString {
    type is string;
    length < 10;
}
```

Now what do we have here? Now we’ve defined a constraint that a particular value is not just a string, but its a string with a length less than 10! This constraint will be enforced both with static type checks and runtime type checks, to ensure there is no possible way ShortString could ever take on a value longer than 10 characters. Lets try to use this variable in action:

```
const value: ShortString;
value = “imshort”; // This is fine, because imshort is only 7 characters long.

value = “im really long!”; // This will give you a compiler error
```

But wait, I hear you say! This will never work! What about user input? How can we possibly enforce string length at the compiler level, when users can type in whatever they want? Good question.

When a user provides input, it is usually in the form of an unconstrained or partially constrained value. The act of validating the user's input is called constraining the value, and must be done explicitly (TODO: should we allow automatic inference of what constraints to apply?). Lets see this in action:
```
import io;
const value = io.readUserInput(); // Value is now constrained only by {type: string}

const shortValue: ShortString;
shortValue = value; // This will lead to a compiler error, because value does not have the length constraint imposed on it. Instead do the following:

shortValue = value constrained by ShortString; // This will trigger runtime data validation. If value does not meet the constraints imposed upon it by ShortString, then an exception is raised.
```

You can also constrain values in place, without having to declare a whole new value.

```
import io;
const value = io.readUserInput(); // Value is now constrained only by {type: string}

constrain value with ShortString;

const shortValue: ShortString;
shortValue = value; // This works fine now, because value has been previously constrained, altering its type.
```

It’s important to note that the constraints themselves live independently of the name they are given. E.g., the following code is also acceptable, even though we don’t explicitly constrain value with ShortString:
```
import io;

const value = io.readUserInput(); // Value is now constrained only by {type: string}

constrain value with {length < 10};

const shortValue: ShortString;
shortValue = value;
```

As you can see, Reliascript type system introduces a whole new level of static type checking that has never been done before in other languages.

## Varieties of Constraints
Reliascript has a huge number of built in constraints meant to enable static type checking in an extremely wide variety of scenarios. Here is a non exhaustive list:

1. What type a particular value is, e.g. date, string,
2. String length minimums and maximums
3. Whether a value is required or optional (e.g. its value can be null)
4. Whether a string matches a particular pattern, such an email, phone number or ip address
5. Whether a number is between a given minimum or maximum value
6. Whether a particular value is supposed to change or be updated at the same time as another value (more on this later)

### Pattern Matching Constraint

Let's go into a bit more detail on the pattern matching constraint. Imagine we wanted to have a string to represent an email address, and we want to ensure that it can never contain anything that isn’t a valid email address. Reliascript allows you to do the following:

```
const value: {
    type is string;
    matches pattern email;
}

value = “cheese!”; // This will give you a compiler error, because the given string does not match the email pattern

value = “cheese@example.com”; // This is allowed, because the given value matches the email pattern

```

Under the hood, patterns take one of two forms:
1. A regular expression which must match the string
2. A function which returns a boolean value on whether a given value meets the desired criteria

The pattern matching constraint provides you with a powerful way to extend your type system. In other programming languages, this might have been accomplished by creating a subclass of some built-in string class with your added validation. Then you could rely on the existing type enforcement semantics of your compiler.

In Reliascript, data validation has become a first class citizen within the language, enforced with a combination of static compile time and runtime checks. As a consequence, it becomes downright impossible to assign a bad value into a variable. At the same time, you can avoid having data validation getting duplicated in hundreds of places throughout your code, as some extra cautious teams sometimes do. When the type system has seen a particular value has already been tested against a constraint, it doesn’t need to repeat it.

## Tagging
Tags are sort of like additional pieces of type information that can be attached to a field.

A field can either be constrained to a tag, e.g. all values it takes on must be given that tag. Imagine, for example, you have some email marketing software. You may have a common Email constraint, which you reuse between a bunch of different models within your application. However, you really don’t want to confuse the email address of the user, e.g. their login email address, with the email addresses of the contacts they are sending emails to. For this, you can use tags!

```
// We define a common email address constraint, used for all email addresses in the system
constraint CommonEmail {
    type is string;
    matches pattern email;
    length <= 10;
    length > 0;
}

// Now, we define two different tags. One is for the users email, the other is for a contacts email
tag UserEmail;
tag ContactEmail;

constraint UserEmail {
    constrained by CommonEmail;
    tagged by UserEmail;
}

constraint ContactEmail {
    constrained by CommonEmail;
    tagged by ContactEmail;
}

const value: UserEmail;
value = “genixpro@gmail.com”;

Const secondValue: ContactEmail;
secondValue = value; // Yields a compiler error, because value is tagged as UserEmail, and secondValue is tagged as ContactEmail;
// Although they are technically identical in the constraints they match, because they have different tags, so the compiler will print an error
```

### Dynamic Tags
Tags do not have to be always statically provided. It's possible to create tags which depend upon the exact values that an object takes in production - so some values will have a given tag, and some will not, even though they have the same underlying type.

This is a lot like having a constraint which is not always enforced. When the constraint matches, the value takes on the given tag. When the constraint fails, the value does not take on the given tag.

TODO: describe pristine / invalid tags here


## Sagas, Transactions, Retries, 2 phase commits

One of the most difficult things to get right when your engineering a distributed, cloud based system is how the heck do you ensure reliable execution of your code across a multitude of different networked services, which may include different machines in different data centers, third party vendors, making multiple concurrent writes to a database, etc… when all of these things can potentially fail.

Over the years, software engineers have thought up a wide variety of different mechanisms to try and ensure a system’s reliable execution. First starters, we have the automatic retry - if we make a network request and it fails, just try it again. Then we have transactions in databases - a set of changes to the database that must all succeed or all fail, without any partial changes. Then we have the even more modern version, the Saga - a set of concrete steps to be executed, each with forward code and associated rollback / reversal code that should get executed in case a subsequent step in the saga fails. Then we have two phased commits, atomic and idempotent operations, and a whole host of other intellectual contraptions.

Like with data validation, Reliascript has been designed to take these contraptions from being implemented in libraries on top of the language, to being features of the language itself. With this comes a whole host of safety benefits that can be enforced and automated with the Reliascript compiler.


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

Note - ideally the rollback for a reversible can be automatically created, so the person doesn’t need to explicitly write their own reversible.

An idempotent unit - this is a function that comes in one or two parts - the guard and the body.

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
### Saga - automatically ties together multiple different reversible steps in an automated manner.

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
        value matches pattern data.email;
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

## Desirable Properties (in progress)


1. Data validation is a first-class citizens
   1. Goes beyond just type checking although that’s part of it 
   2. As many constraints as possible can be statically checked at compile time 
   3. Can infer constraints automatically based on actual usage of variables 
2. Data serialization is a first class citizen
   1. All data is by default persisted unless otherwise specified [Maybe?]
   2. Data is meant to be stored - no distinction between classes/objects for in-memory usage and classes/objects to be stored to a database - this is given as a parameter of the class / object. Should be no need for an ORM
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
   1. Ways to make operations idempotent
   1. Automatic retries from failures
   1. Timeouts and deadline propagation
   1. Automatic restarts on crash
   1. Message passing, queuing, etc..
   1. Asynchronous programming, promises, async/await etc..
   1. Auditability / history of what happened
   1. Mocking?
   1. Auto test data generation?
   1. DB Transactions as first class citizens?
   1. Request ID?
   1. Automatic logging of the entire call path?
   1. Forward and backwards data compatibility?
   1. Status codes? Error messages?
   1. Environment Configuration? (ideally dynamic configuration values can be treated as static and can then have constraints enforced)

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




