---
layout: page
title: "TypeScript Introduction"
permalink: /typescript1/
---


## Learning Outcomes

- Create programs in TypeScript using types to ensure correctness at compile time.
- Explain how TypeScript features like Interfaces, Union Types and Optional Properties allow us to model types in common JavaScript coding patterns with precision.
- Explain how Generics allow us to create type safe but general and reusable code.
- Compare and contrast strongly, dynamically and gradually typed languages.
- Describe how compilers that support type inference assist us in writing type safe code.

## Introduction

As the Web 2.0 revolution hit in 2000s web apps built on JavaScript grew increasingly complex and today, applications like GoogleDocs are as intricate as anything built over the decades in C++.  In the 90s I for one (though I don’t think I was alone) thought that this would be impossible in a dynamically typed language.  It is just too easy to make simple mistakes (as simple as typos) that won’t be caught until run time.  It’s likely only been made possible due to increased rigour in testing.  That is, instead of relying on a compiler to catch mistakes, you rely on a comprehensive suite of tests that evaluate running code before it goes live.

Part of the appeal of JavaScript is that the source code being the artefact that runs directly in a production environment gives an immediacy and comprehensibility to software deployment.  However, in recent years more and more tools have been developed that introduce a build-chain into the web development stack.  Examples include: minifiers, which compact and obfuscate JavaScript code before it is deployed; bundlers, which merge different JavaScript files and libraries into a single file to (again) simplify deployment; and also, new languages that compile to JavaScript, which seek to fix older versions of the JavaScript language’s shortcomings and compatibility issues in different browsers.  Examples of the latter include CoffeeScript, Clojure and (more recently) PureScript (which we will visit later in this unit).  Right now, however, we will take a closer look at another language in this family called TypeScript.  See here for some tutorials and deeper documentation.

TypeScript is interesting because it forms a relatively minimal augmentation, or superset, of EcmaScript syntax that simply adds type annotations.  For the most part, the compilation process simply performs validation on the declared types and strips away the type annotations rendering just the legal JavaScript ready for deployment.  This lightweight compilation into a language with a similar level of abstraction to the source is known also known as transpiling (as opposed to C++ or Java where the object code is much closer to the machine execution model).

The following is intended as a minimally sufficient intro to TypeScript features such that we can type some fairly rich data structures and higher-order functions.
An excellent free resource for learning the TypeScript language in depth is the [TypeScript Deep-dive book](https://basarat.gitbooks.io/typescript/content/docs/getting-started.html).  

## Type annotations

Type annotations in TypeScript come after the variable name’s declaration, like so:

```javascript
let i: number = 123;
```

Actually, in this case the type annotation is completely redundant.  The TypeScript compiler features sophisticated type inference.  In this case it can trivially infer the type from the type of the literal.

Previously, we showed how rebinding such a variable to a string in JavaScript is perfectly fine by the JavaScript interpreter.  However, such a change of type in a variable a dangerous pattern that is likely an error on the programmer’s part.  The TypeScript compiler will generate an error:

```javascript
let i = 123;
i = 'hello!';
```

> [TS compiler says] Type '"hello!"' is not assignable to type 'number'.

## Why should we declare types?

Declaring types for variables, functions and their parameters, and so on, provides more information to the person reading and using the code, but also to the compiler, which will check that you are using them consistently and correctly.  This prevents a lot of errors.

Consider that JavaScript is commonly used to manipulate HTML pages.  For example, I can get the top level headings in the current page, like so:

```javascript
const headings = document.getElementsByTagName("h1")
```

In the browser, the global ```document``` variable always points to the currently loaded HTML document.  Its method ```getElementsByTagName``` returns a collection of elements with the specified tag, in this case ```<h1>```.

Let's say I want to indent the first heading by 100 pixels, I could do this by manipulating the "style" attribute of the element:

```javascript
headings[0].setAttribute("style","padding-left:100px")
```

Now let's say I do a lot of indenting and I want to build a library function to simplify manipulating padding of HTML elements.

```javascript
function setLeftPadding(elem, value) {
    elem.setAttribute("style", `padding-left:${value}`)
}
```

```javascript
setLeftPadding(headings[0], "100px")
```

But how will a user of this function (other than myself, or myself in three weeks when I can't remember how I wrote this code) know what to pass in as parameters?  ```elem``` is a pretty good clue that it needs to be an instance of an HTML element, but is value a number or a string?

In TypeScript we can make the expectation explicit:

```typescript
function setLeftPadding(elem: Element, value: string) {
    elem.setAttribute("style", `padding-left:${value}`)
}
```

If I try to pass something else in, the TypeScript compiler will complain:

```javascript
setLeftPadding(headings[0],100)
```

> Argument of type '100' is not assignable to parameter of type 'string'.ts(2345)

In JavaScript, the interpreter would silently convert the number to a string and set ```padding-left:100``` - which wouldn't actually cause the element to be indented because CSS expects ```px``` (short for pixel) at the end of the value.

Potentially worse, I might forget to add the index after headings:

```javascript
setLeftPadding(headings,100)
```

This would cause a run-time error in the browser:

> VM360:1 Uncaught TypeError: headings.setAttribute is not a function

This is because headings is not an HTML Element but an HTMLCollection, with no method ```setAttribute```.  Note that if I try to debug it, the error will be reported from a line of code inside the definition of ```setLeftPadding```.  I don't know if the problem is in the function itself or in my call.

The same call inside a typescript program would trigger a compile-time error.  In a fancy editor with compiler services like VSCode I'd know about it immediately because the call would get a red squiggly underline immediately after I type it, I can hover over the squiggly to get a detailed error message, and certainly, the generated broken JavaScript would never make it into production.
![Compile Error Screenshot](setLeftPaddingTypeScript.png)

## Union Types

Above we see how typescript prevents a variable from being assigned values of different types.
However, it is a fairly common practice in JavaScript to implicitly create overloaded functions by accepting arguments of different types and resolving them at run-time.  

The following will append the "px" after the value if a number is passed in, or simply use the given string (assuming the user added their own "px") otherwise.

```javascript
function setLeftPadding(elem, value) {
    if (typeof value === "number")
        elem.setAttribute("style", `padding-left:${value}px`)
    else
        elem.setAttribute("style", `padding-left:${value}`)
}
```

So this function accepts either a string or a number for the ```value``` parameter - but to find that out we need to dig into the code.  The "Union Type" facility in typescript allows us to specify the multiple options directly in the function definition, with a list of types separated by "```|```":

```typescript
function setLeftPadding(elem: Element, value: string | number) {...
```

Going further, the following allows either a number, or a string, or a function that needs to be called to retrieve the ```string```:

```typescript
function setLeftPadding(elem: Element, value: number | string | (()=>string)) {
    if (typeof value === "number")
        elem.setAttribute("style", `padding-left:${value}px`)
    else if (typeof value === "function")
        elem.setAttribute("style", `padding-left:${value()}`)
    else
        elem.setAttribute("style", `padding-left:${value}`)
}
```

Note that typescript's notation for function types uses the fat-arrow syntax.

The TypeScript typechecker also knows about typeof expressions (as used above) and will also typecheck the different clauses of if statements that use them for consistency with the expected types.

## Interfaces

In typescript I can declare an ```interface``` which defines the set of properties and their types, that I expect to be available for certain objects.

For example, when tallying scores at the end of semester, I will need to work with collections of students that have a name, assignment and exam marks.  There might even be some special cases which require mark adjustments, the details of which I don't particularly care about but that I will need to be able to access, e.g. through a function particular to that student.  The student objects would need to provide an interface that looks like this:

```typescript
interface Student {
  name: string
  AssignmentMark: number
  ExamMark: number
  markAdjustment(): number
}
```

Note that this interface guarantees that there is a function returning a number called ```markAdjustment``` available on any object implementing the ```Student``` interface, but it says nothing about how the adjustment is calculated.  Software engineers like to talk about [Separation of Concerns (SoC)](https://en.wikipedia.org/wiki/Separation_of_concerns).  To me, the implementation of markAdjustment is Someone Else's Problem (SEP).

Now I can define functions which work with students, for example to calculate the average score for the class:

```typescript
function averageScore(students: Student[]): number {
  return students.reduce((total, s) =>
    total + s.assignmentMark + s.examMark + s.markAdjustment(), 0)
    / students.length
}
```

Other parts of my program might work with richer interfaces for the student database---with all sorts of other properties and functions available---but for the purposes of computing the average class score, the above is sufficient.

## Generic Types

What if sometimes students have other information that I need to know exists, for example I might need to pass it on to someone else, but I don't really care about what type of information it is myself?  This might matter if I am preparing class lists for exams.  Certain students might need special support, such as a scribe or reader.  The type of support might come in many forms - perhaps that I can't even predict in advance.  However, if there is information about special support for a student, I will have to make sure that it is passed along to the invigilators who will know what to do with it.  The type particulars don't matter to me, but it's important that I pass it along at the right time, and that I don't mix it up with something else.  In this case I can add a *type parameter* to my definition of Student, and a property that has that type:

```typescript
interface Student<T> {
  name: string
  specialConsideration: T
  ...
}
```

Here ```T``` is the type parameter, and the compiler will infer its type when it is used.  Type parameters can have more descriptive names if you like, but they must start with a capital.  The convention though is to use rather terse single letter parameter names in the same vicinity of the alphabet as T.  This habit comes from C++, where T used to stand for "Template", and the terseness stems from the fact that we don't really care about the details of what it is in places where we used such type parameters.  As in functions parameter lists, you can also have more than one type parameter:

```typescript
interface Student<T,U> {
  name: string
  specialConsideration: T
  someOtherThingThatWeDontCareMuchAbout: U
  ...
}
```

Formally, this is a kind of "parametric polymorphism".  T may be referred to as a *type parameter*, *type variable* or *generic type*.  

You see generic types definitions used a lot in algorithm and data structure libraries, to give a type---to be specified by the calling code---for the data stored in the data structures.  For example, the following interface might be the basis of a linked list element:

```typescript
interface IListNode<T> {
    data: T;
    next?: IListNode<T>;
}
```

The specific type of ```T``` will be resolved when ```data``` is assigned a specific type value.

## Optional Properties
Look again at the ```next``` property of ```IListNode```:

```typescript
    next?: IListNode<T>;
```

The ```?``` after the ```next``` property means that this property is optional.  Thus, if ```node``` is an instance of ```IListNode<T>```, we can test if it has any following nodes:
```javascript
typeof node.next === 'undefined'
```
or simply:
```javascript
!node.next 
```
In either case, if these expressions are ```true```, it would indicate the end of the list.

A concrete implementation of a class for ```IListNode<T>``` can provide a constructor:

```javascript
class ListNode<T> implements IListNode<T> {
   constructor(public data: T, public next?: IListNode<T>) {}
}
```

Then we can construct a list like so:

```javascript
const list = new ListNode(1, new ListNode(2, new ListNode(3)));
```
------
## Exercises
- Implement a class ```List<T>``` whose constructor takes an array parameter and creates a linked list of ListNode<T>.  
- Add methods to your ```List<T>``` class for:

```typescript
forEach(f: (_:T)=> void): List<T>
filter(f: (_:T)=> boolean): List<T>
map<V>(f: (_:T)=> V): List<V>
reduce(f: (accumulator:V, t:T)=> V, initialValue: V): V
```

Note that each of these functions returns a list, so that we can chain the operations, e.g.:

```typescript
list.filter(x=>x%2===0).reduce((x,y)=>x+y,0)
```
-------------

## Using the compiler to ensure immutability

We saw [earlier](../functionaljavascript), that while an object reference can be declared const:

```javascript
const studentVersion1 = {
  name: "Tim",
  assignmentMark: 20,
  examMark: 15
}
```

Which prevents reassigning ```studentVersion1``` to any other object, the ```const``` declaration does not prevent properties of the object from being changed:

```javascript
studentVersion1.name = "Tom"
```

Typescript will not complain about the above assignment at all.

However, we can declare immutable types using the ```Readonly``` construct:

```javascript
type ImmutableStudent = Readonly<{
name: string;
assignmentMark: number;
examMark: number;
}>

const studentVersion1:ImmutableStudent = {
name: "Tim",
assignmentMark: 20,
examMark: 15
}

studentVersion1.name = "Tom"
```

Now we get the squiggly:
![Compile Error Screenshot](readonly.png)

## Typing systems with different ‘degrees’ of strictness

C++ is considered a strongly typed language in the sense that all types of values and variables must match up on assignment or comparison.  Further, it is “statically” typed in that the compiler requires complete knowledge (at compile-time) of the type of every variable.  This can be overridden (type can be cast away and void pointers passed around) but the programmer has to go out of their way to do it (i.e. opt-out).

JavaScript, by contrast, as we have already mentioned, is dynamically typed in that types are only checked at run time. Run-time type errors can occur and be caught by the interpreter on primitive types, for example the user tried to invoke an ordinary object like a function, or refer to an attribute that doesn’t exist, or to treat an array like a number.

TypeScript represents a relatively new trend in being a gradually typed language.  Another way to think about this is that, by default, the type system is opt-in.  Unless declared otherwise, all variables have type any.  The any type is like a wild card that always matches, whether the any type is the target of the assignment:

```javascript
let value; // has implicit <any> type
value = "hello";
value = 123;
// no error.

Or the source (r-value):
let value: number;
value = "hello";
//[ts] Type '"hello"' is not assignable to type 'number'.
value = <any>"hello"
// no error.
```

While leaving off type annotations and forcing types with any may be convenient, for example, to quickly port legacy JavaScript into a TypeScript program, generally speaking it is good practice to use types wherever possible, and can actually be enforced with the ```--noImplicitAny``` compiler flag.  The compiler’s type checker is a sophisticated constraint satisfaction system and the correctness checks it applies are usually worth the extra effort - especially in modern compilers like TypeScript where type inference does most of the work for you.