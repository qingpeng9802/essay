# JavaScript *constructor* and *prototype chain*: From ECMAScript Specification's Perspective  

## Preface

> Please note that this article emphasizes serious discussions. It may be too boring for beginners, and beginners may learn other interesting and excellent tutorials (such as [class-inheritance](https://javascript.info/class-inheritance)) for better experience :)

! After you master *prototype chain*, it is better not to mutate any [[Prototype]] in any industrial program (unless it is for language feature compatibility or other solutions are exhausted). See [The performance hazards of [[Prototype]] mutation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/The_performance_hazards_of_prototype_mutation) and [JavaScript engine fundamentals: optimizing prototypes](https://mathiasbynens.be/notes/prototypes).

We will cover the *constructor* and the *prototype chain* of the objects in ECMAScript. We will start our discussion with [ECMA-262 13th (ES13 2022)](https://262.ecma-international.org/13.0/).  
// TODO crossRef

There are many code examples in this article. It is highly recommended to paste and try these code examples in the [Chrome DevTools Console](https://developer.chrome.com/docs/devtools/console/javascript/).  

## Notation

Please note that in the following parts, the terms *JavaScript* and *ECMAScript* will be used as needed in specific context, and their meanings are almost the same, see [ECMAScript, ECMA-262 and JavaScript](https://en.wikipedia.org/wiki/ECMAScript#ECMAScript,_ECMA-262_and_JavaScript).  

### Double Square Brackets [[ ]]

[6.1.7.2 Object Internal Methods and Internal Slots](https://262.ecma-international.org/13.0/#sec-object-internal-methods-and-internal-slots)

> Internal methods and internal slots are identified within this specification using names enclosed in double square brackets [[ ]].

## *constructor*, [[Construct]] and `prototype` property

### Definition of *constructor* 

Basically, when you are writing something like `function foo() {}` in JavaScript, you are creating a *function object* (see [20.2 Function Objects](https://262.ecma-international.org/13.0/#sec-function-objects)). Then, let's see the definition of *constructor*.  

[4.4.7 constructor](https://262.ecma-international.org/13.0/#sec-constructor)
> **function object** that creates and initializes objects
  
[Table 5: Additional Essential Internal Methods of Function Objects](https://262.ecma-international.org/13.0/#table-additional-essential-internal-methods-of-function-objects)
> A *function object* is an object that supports the [[Call]] internal method. A *constructor* is an object that supports the [[Construct]] internal method.  
Every object that supports [[[Construct]]](https://262.ecma-international.org/13.0/#sec-ecmascript-function-objects-construct-argumentslist-newtarget) must support [[[Call]]](https://262.ecma-international.org/13.0/#sec-ecmascript-function-objects-call-thisargument-argumentslist); that is, every constructor must be a function object. Therefore, a constructor may also be referred to as a constructor function or constructor function object.  

By definition, the set of constructors is a subset of the set of function objects:
```
_______________________________
|  ________________           |
|  | constructors |  function |
|  |______________|  objects  |
|_____________________________|
```
|             |constructors               |function objects|
|-|-|-|
|must support |[[Call]] and [[Construct]] | [[Call]]       |

<br />

### [[Construct]] and `prototype` Property of Function Objects

This table summarises the [[Construct]] and the `prototype` property of different `kind` of functions which are created by [20.2.1.1.1 CreateDynamicFunction (*constructor, newTarget, kind, args*)](https://262.ecma-international.org/13.0/#sec-createdynamicfunction):

|`kind`|normal|generator|async|asyncGenerator|
|-|-|-|-|-|
|[[Construct]]|yes|no|no|no|
|`.prototype` property|Function.prototype|GeneratorFunction.prototype.prototype|no|AsyncGeneratorFunction.prototype.prototype|

Note that if a function is `normal`, the function will have a [[[Construct]]](https://262.ecma-international.org/13.0/#sec-ecmascript-function-objects-construct-argumentslist-newtarget) internal method by default, that is, it is a *constructor* by default. See Step 35. Else if `kind` is `normal`, perform `MakeConstructor(F)`. in [20.2.1.1.1 CreateDynamicFunction (*constructor, newTarget, kind, args*)](https://262.ecma-international.org/13.0/#sec-createdynamicfunction).

Also, there are some function objects do not have a **"prototype"** property.  
See NOTE in [20.2.4.3 prototype](https://262.ecma-international.org/13.0/#sec-function-instances-prototype),
> Function objects created using `Function.prototype.bind`, or by evaluating a *MethodDefinition* (that is not a *GeneratorMethod* or *AsyncGeneratorMethod*) or an *ArrowFunction* do not have a **"prototype"** property.  
  
(Digression: Unlike `normal` function, for `generator` function and `asyncGenerator` function, `theFunctionYouDefined.prototype.constructor` doesn't point back the function itself, it points to `GeneratorFunction.prototype` and `AsyncGeneratorFunction.prototype` respectively.  
If you are curious, `GeneratorFunction.prototype.constructor === GeneratorFunction` and `AsyncGeneratorFunction.prototype.constructor === AsyncGeneratorFunction`. See [Figure 6 (Informative): Generator Objects Relationships](https://262.ecma-international.org/13.0/#figure-2).)  

<br />

### The Ability of Function Objects to Construct Objects

It is a bit confusing about how [[Construct]] and `prototype` property is involved in the construction of objects.  

First, whether it has [[Construct]] is the criterion for the *constructor*. A more straightforward criterion is whether it can be called with `new` ([13.3.5.1.1 EvaluateNew](https://262.ecma-international.org/13.0/#sec-evaluatenew) > [7.3.15 Construct (F [,argumentsList [, newTarget]])](https://262.ecma-international.org/13.0/#sec-construct)). Thus, **"prototype"** property CANNOT determine whether a function object is a *constructor*.  

However, non-constructors can also construct objects. Let's summarises the ways of constructing objects in this table:
|                      |has a `.prototype` |has no `.prototype`|
|-|-|-|
|has a [[Construct]]   |`new`              |`new`              |
|has no [[Construct]]  |`Object.create()`  |no way             |

Here is an example that a function object, which is created by `Function.prototype.bind`, has a [[Construct]] but has no `.prototype` property,
```js
const obj = {
    x: 42,
    getX: function() { return this.x; }
};
// note that we need unboundGetX has [[Construct]]
const unboundGetX = obj.getX;
const boundGetX = unboundGetX.bind(obj);

unboundGetX.prototype !== undefined;
boundGetX.prototype === undefined;

// (new boundGetX()).[[Prototype]], we can still `new` it
// In fact, we `new` on unboundGetX's [[Construct]]
Object.getPrototypeOf(new boundGetX()) === unboundGetX.prototype;
Object.getPrototypeOf(new boundGetX()) !== boundGetX.prototype;
```

Also, we can create a object only through `.prototype` property without having [[Construct]] by using `Object.create()`, see [Object.create()](#what-is-objectcreate). Here is an example for a `generator` function object,  
```js
function* g(i, a) {
    this.a = a;
    yield i + 10;
}

// const instg = new g(4, 6);
// Uncaught TypeError: g is not a constructor

const instg = Object.create(g.prototype, {a: { value: 99 }});
// Generator {a: 99}
instg.a === 99
```

<br />

### Function.prototype

There is a very special object that is a *function object* but not a *constructor*.  
[20.2.3 Properties of the Function Prototype Object](https://262.ecma-international.org/13.0/#sec-properties-of-the-function-prototype-object)  
> accepts any arguments and returns undefined when invoked.  
does not have a [[Construct]] internal method  

> NOTE
The Function prototype object is specified to be a function object to ensure compatibility with ECMAScript code that was created prior to the ECMAScript 2015 specification.

[20.2.2.2 Function.prototype](https://262.ecma-international.org/13.0/#sec-function.prototype)
> The value of **Function.prototype** is the Function prototype object.

*Function prototype object* is a *function object*, but it does not have a [[Construct]], that is, it is not a *constructor*. We will discuss **Function.prototype** in [Inheritance](#Inheritance) later.  

<br />

## *prototype* and *prototype chain*   

### Definition of *prototype*

[4.3.1 Objects](https://262.ecma-international.org/13.0/#sec-objects) discussed about the *prototype* and the *prototype chain*, which are the key concepts for understding the objects in JavaScript.  

Unlike the class-based objects in C++ or Java, the objects in JavaScript can be created via a *literal notation* or via *constructors* which create objects and then execute code that initializes all or part of them by assigning initial values to their properties. Each constructor is a function that has a property named **"prototype"** that is used to implement *prototype-based inheritance* ([Inheritance](#Inheritance)) and *shared properties* ([Property Visibility](#property-visibility)).

> Every object created by a constructor has an **implicit reference** (called the object's *prototype*) to the value of its constructor's **"prototype"** property. Furthermore, a prototype *may have a non-null implicit reference* to its prototype, and so on; this is called the *prototype chain*.  

Note that we are introducing a new concept object's *prototype*, which is an **implicit reference** and is different from constructor's `prototype` property we discussed before.

Let's see an example to understand object's *prototype* ( `obj.[[Prototype]]` ),
```js
function Cat(color) {
    this.color = color;
}
const cat0 = new Cat('black');
cat0.constructor === Cat;

// Legacy method: Object.prototype.__proto__
// cat0.__proto__ is equal to 
Object.getPrototypeOf(cat0);

// cat0.[[Prototype]] points to Cat's prototype
Object.getPrototypeOf(cat0) === Cat.prototype;
// cat0.[[Prototype]].[[Prototype]] points to Object's prototype
Object.getPrototypeOf(Cat.prototype) === Object.prototype;
// cat0.[[Prototype]].[[Prototype]].[[Prototype]] points to null
Object.getPrototypeOf(Object.prototype) === null;

// each step is a .[[Prototype]]
// prototype chain: cat0 -> Cat.prototype -> Object.prototype -> null
```
! Please pay attention to our wording in the comment here. `cat0.[[Prototype]]` means the **reference** (called the *`cat0`'s prototype*). The value of `cat0.[[Prototype]]` is *the property* `Cat.prototype` of the constructor `Cat`. A simple way to distinguish is that we can use the word "*object's prototype*" and "constructor's `prototype` property".  
```js
// since cat0 is not a constructor
cat0.prototype === undefined;
```
We can say:  
"*`cat0`'s prototype* is the property `Cat.prototype` of `Cat`."  
"*`Cat.prototype`'s prototype* is the property `Object.prototype` of `Object`."  
(Wait, what is *`Cat`'s prototype*? We will discuss it in [Inheritance](#Inheritance) later.)  
"`cat0` has NO property `prototype`."  
"`Cat` has a property `prototype`."

In general, if an object is a *constructor*, we can say that "the object has a property `prototype`."  
Also, we can say that "an *object's prototype* is its constructor's `prototype` property."  

<br />

### Property Visibility
Note that the differences of carrying information in different languages:
|                   |class-based object-oriented (C++/Java)|ECMAScript (JavaScript)|
|-|-|-|
|state is carried   |by instances                          |by objects             |
|methods is carried |by classes                            |by objects             |
|inheritance is     |structure and behaviour       |structure, behaviour, and state|

In Fig 1, 
> All objects that do not directly contain a particular property *that their **prototype** contains* share that property and its value.  

<img src="https://262.ecma-international.org/13.0/img/figure-1.png" alt="Fig 1: Object/Prototype Relationships"/>  

`CF` is a constructor (and also an object). `cf`s are objects created by using `new CF()`.
`CFp` is `CF`'s `prototype` property by explicit reference.  
Note that, since `CFp` is `CF`'s `prototype`, `CFP1` is shared by `cf`s, but `P1` and `P2` are NOT visible to `cf`s and `CFp`.

<br />

### *constructor*, `prototype` property and *prototype* Relationship

See [4.4.8 prototype](https://262.ecma-international.org/11.0/#sec-terms-and-definitions-prototype),
> When a constructor creates an object, that object **implicitly references** the **constructor's "prototype"** property for the purpose of resolving property references. The **constructor's "prototype"** property can be referenced by the program expression `constructor.prototype`, and properties added to an object's prototype are shared, through inheritance, by all objects sharing the prototype. Alternatively, a new object may be created with an explicitly specified prototype by using the `Object.create` built-in function.

This example shows the relationship between constructor*, `prototype` property and *prototype*,  
```js
function Cat(color) {
    this.color = color;
}
const cat0 = new Cat('black');

// cat0.[[Prototype]] points to cat0's constructor's prototype
Object.getPrototypeOf(cat0) === Cat.prototype;
cat0.constructor === Cat;
Object.getPrototypeOf(cat0) === cat0.constructor.prototype;

// just curious, what is Cat's prototype's constructor?
Cat.prototype.constructor === Cat;
```

```
cat0.[[Prototype]] === cat0.constructor.prototype === Cat.prototype
Cat.prototype.constructor === Cat;

   .constructor         .[[Prototype]]
 --------------- cat0 -----------------
 |                                    |
 |                                    |
\|/                                  \|/
             .prototype
Cat  --------------------------> Cat.prototype
    <--------------------------
             .constructor
```

<br />

## `new` keyword and `Object.create()`

### What happened after `new`  

From [13.3.5.1.1 EvaluateNew](https://262.ecma-international.org/13.0/#sec-evaluatenew) > [7.3.15 Construct (F [,argumentsList [, newTarget]])](https://262.ecma-international.org/13.0/#sec-construct), [[Construct]] will create the object. Also, [[Construct]] can be invoked via the `new` operator or a `super` call.
```js
function CatPrint(color) {
    this.color = color;
    console.log('got a cat');
}
const cat0 = new CatPrint('black');
// print: got a cat
// cat0: Cat {color: 'black'}
```

In [10.2.2 [[Construct]] (*argumentsList, newTarget*)](https://262.ecma-international.org/13.0/#sec-ecmascript-function-objects-construct-argumentslist-newtarget) steps, 

- 3\. If `kind` is base, then   
    &nbsp;&nbsp;&nbsp;&nbsp;a. Let `thisArgument` be ? `OrdinaryCreateFromConstructor(newTarget, "%Object.prototype%")`.  
    > 1\. Creates a blank, plain JavaScript object. For convenience, let's call it `newInstance`.   
    2\. Points `newInstance`'s [[Prototype]] to the constructor function's `prototype` property.

- 4\. Let `calleeContext` be `PrepareForOrdinaryCall(F, newTarget)`.  
    > start preparing `this`-related information from this step

- 6\. If `kind` is base, then  
    &nbsp;&nbsp;&nbsp;&nbsp;a. Perform `OrdinaryCallBindThis(F, calleeContext, thisArgument)`.  
    > 3\. Executes the constructor function with the given arguments, binding `newInstance` as the `this` context (i.e. all references to `this` in the constructor function now refer to `newInstance`).

- 7\. Let `constructorEnv` be the LexicalEnvironment of `calleeContext`.  
    12\. Let `thisBinding` be ? `constructorEnv.GetThisBinding()`.  
    14\. Return `thisBinding`.  
    > 4\. If the constructor function returns a non-primitive, this return value becomes the result of the whole `new` expression. Otherwise, if the constructor function doesn't return anything or returns a primitive, `newInstance` is returned instead. (Normally constructors don't return a value, but they can choose to do so to override the normal object creation process.)

The comment parts above are from [MDN new Description](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new#description).

With the steps, let's implement a `dummyNew`.
```js
// dummyNew can be implemented like this
function dummyNew(Foo, ...args){
    // directly call ES13 10.1.12 OrdinaryObjectCreate()
    const obj = {};
    Object.setPrototypeOf(obj, Foo.prototype);
    const resFoo = Foo.call(obj, ...args);

    // primitive value will be wrapped into Object
    // only return resFoo when it is a Object
    return resFoo !== Object(resFoo) ? obj : resFoo;
}

function CatPrint(color) {
    this.color = color;
    console.log('got a cat');
}
const cat0 = dummyNew(CatPrint, 'black');
// print: got a cat
// cat0: CatPrint {color: 'black'}
```

<br />

### What is `Object.create()`
In [20.1.2.2 Object.create (*O, Properties*)](https://262.ecma-international.org/13.0/#sec-object.create),  

1. If `Type(O)` is neither Object nor Null, throw a `TypeError` exception.
2. Let `obj` be `OrdinaryObjectCreate(O)`.
3. If `Properties` is not undefined, then  
    a. Return ? `ObjectDefineProperties(obj, Properties)`.
4. Return `obj`.  

```js
function CatPrint(color) {
    this.color = color;
    console.log('got a cat');
}
const cat0 = Object.create(CatPrint.prototype, {color: { value: 'black' }});
// didn't print
// cat0: Cat {color: 'black'}
```

The biggest difference is that the constructor (the function) is not called here. The object is created directly from the provided prototype. Also, this means that we can create objects only through prototypes without having [[Construct]].

With the steps, let's implement a `dummyCreate`.  
Note that the properties (such as `{color: { value: 'black' }}`) are defined by `Object.defineProperties()`.  
```js
// dummyCreate can be implemented like this
function dummyCreate(Foo, propertiesObject){
    // directly call ES13 10.1.12 OrdinaryObjectCreate()
    const obj = {};
    Object.setPrototypeOf(obj, Foo.prototype);
    Object.defineProperties(obj, propertiesObject);
    return obj;
}

function CatPrint(color) {
    this.color = color;
    console.log('got a cat');
}
const cat0 = dummyCreate(CatPrint, {color: { value: 'black' }});
// didn't print
// cat0: CatPrint {color: 'black'}
```

## Reference
[ECMA-262 13th (ES13 2022)](https://262.ecma-international.org/13.0/)  
