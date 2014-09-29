js-schema
=========

js-schema is a new way of describing object schemas in JavaScript. It has a clean and simple syntax,
and it is capable of serializing to/from the popular JSON Schema format. The typical use case is
declarative object validation.

**Latest release**: 0.7.1 (2014/09/29)

Features
========

Defining a schema:

```javascript
var Duck = schema({              // A duck
  swim : Function,               //  - can swim
  quack : Function,              //  - can quack
  age : Number.min(0).max(5),    //  - is 0 to 5 years old
  color : ['yellow', 'brown']    //  - has either yellow or brown color
});
```

The resulting function (`Duck`) can be used to check objects against the declared schema:

```javascript
// Some animals
var myDuck = { swim : function() {}, quack : function() {}, age : 2, color : 'yellow' },
    myCat =  { walk : function() {}, purr  : function() {}, age : 3, color : 'black'  },
    animals = [ myDuck, myCat, {}, /*...*/ ];

// Simple checks
console.log( Duck(myDuck) ); // true
console.log( Duck(myCat)  ); // false

// Using the schema function with filter
var ducks   = animals.filter( Duck );                        // every Duck-like animal
var walking = animals.filter( schema({ walk : Function }) ); // every animal that can walk
```

It is also possible to define self-referencing data structures:

```javascript
var Tree = schema({ left : [ Number, Tree ], right : [ Number, Tree ] });

console.log( Tree({ left : 3, right : 3 })                        ); // true
console.log( Tree({ left : 3, right : { left: 5, right: 5 } })    ); // true
console.log( Tree({ left : 3, right : { left: 5, right: 's' } })  ); // false
```

Error reporting:

```javascript
Duck.errors({
  swim: function() {},
  quack: function() {},
  age: 6,
  color: 'green'
});

// {
//   age: 'number = 6 is bigger than required maximum = 5',
//   color: 'string = green is not reference to string = yellow AND
//           string = green is not reference to string = brown'
// }
```

Usage
=====

Include js-schema in your project with `var schema = require('js-schema');` in node.js or with
`<script src="js-schema.min.js"></script>` in the browser.

The first parameter passed to the `schema` function describes the schema, and the return value
is a new function called validator. Then the validator can be used to check any object against
the described schema as in the example above.

There are various patterns that can be used to describe a schema. For example,
`schema({n : Number})` returns a validation function which returns true when called
with an object that has a number type property called `n`. This is a combination of the
object pattern and the instanceof pattern. Most of the patterns are pretty intuitive, so
reading a schema description is quite easy even if you are not familiar with js-schema.
Most patterns accept other patterns as parameters, so composition of patterns is very easy.

Extensions are functions that return validator by themselves without using the `schema` function
as wrapper. These extensions are usually tied to native object constructors, like `Array`,
`Number`, or `String`, and can be used everywhere where a pattern is expected. Examples
include `Array.of(X)`, `Number.min(X)`.

For serialization to JSON Schema use the `toJSON()` method of any schema (it returns an object)
or call `JSON.stringify(x)` on the schema (to get a string). For deserialization use
`schema.fromJSON(json)`. JSON Schema support is still incomplete, but it can reliably deserialize
JSON Schemas generated by js-schema itself.

Patterns
========

### Basic rules ###

There are 10 basic rules used for describing schemas:

1. `Class` (where `Class` is a function, and has a function type property called `schema`)
   matches `x` if `Class.schema(x) === true`.
2. `Class` (where `Class` is a function) matches `x` if `x instanceof Class`.
3. `/regexp/` matches `x` if `/regexp/.test(x) === true`.
4. `[object]` matches `x` if `x` is deep equal to `object`
5. `[pattern1, pattern2, ...]` matches `x` if _any_ of the given patterns match `x`.
6. `{ 'a' : pattern1, 'b' : pattern2, ... }` matches `x` if `pattern1` matches `x.a`,
   `pattern2` matches `x.b`, etc. For details see the object pattern subsection.
7. `primitive` (where `primitive` is boolean, number, or string) matches `x` if `primitive === x`.
8. `null` matches `x` if `x` _is_ `null` or `undefined`.
9. `undefined` matches anything.
10. `schema.self` references the schema returned by the last use of the `schema` function.
    For details see the self-referencing subsection.

The order is important. When calling `schema(pattern)`, the rules are examined one by one,
starting with the first. If there's a match, js-schema first resolves the sub-patterns, and then
generates the appropriate validator function and returns it.

### Example ###

The following example contains patterns for all of the rules. The comments
denote the number of the rules used and the nesting level of the subpatterns (indentation).

```javascript
var Color = function() {}, x = { /* ... */ };

var validate = schema({                    // (6) 'object' pattern
  a : [ Color, 'red', 'blue', [[0,0,0]] ], //     (5) 'or' pattern
                                           //         (2) 'instanceof' pattern
                                           //         (7) 'primitive' pattern
                                           //         (4) 'deep equality' pattern
  b : Number,                              //     (1) 'class schema' pattern
  c : /The meaning of life is \d+/,        //     (3) 'regexp' pattern
  d : undefined,                           //     (9) 'anything' pattern
  e : [null, schema.self]                  //     (5) 'or' pattern
                                           //         (8) 'nothing' pattern
                                           //         (10) 'self' pattern
});

console.log( validate(x) );
```

`validate(x)` returns true if all of these are true:
* `x.a` is either 'red', 'blue', an instance of the Color class,
  or an array that is exactly like `[0,0,0]`
* `x.b` conforms to Number.schema (it return true if `x.b instanceof Number`)
* `x.c` is a string that matches the /The meaning of life is \d+/ regexp
* `x` doesn't have a property called `e`, or it does but it is `null` or `undefined`,
  or it is an object that matches this schema

### The object pattern ###

The object pattern is more complex than the others. Using the object pattern it is possible to
define optional properties, regexp properties, etc. This extra information can be encoded in
the property names.

The property names in an object pattern are always regular expressions, and the given schema
applies to instance properties whose name match this regexp. The number of expected matches can
also be specified with `?`, `+` or `*` as the first character of the property name. `?` means
0 or 1, `*` means 0 or more, and `+` means 1 or more. A single `*` as a property name
matches any instance property that is not matched by other regexps.

An example of using these:
```javascript
var x = { /* ... */ };

var validate = schema({
  'name'             : String,  // x.name must be string
  'colou?r'          : String   // x must have a string type property called either
                                // 'color' or 'colour' but not both
  '?location'        : String,  // if x has a property called 'location' then it must be string
  '*identifier-.*'   : Number,  // if the name of a property of x matches /identifier-.*/ then
                                // it must be a number
  '+serialnumber-.*' : Number,  // if the name of a property of x matches /serialnumber-.*/ then
                                // it must be a number and there should be at least one such property
  '*'                : Boolean  // any other property that doesn't match any of these rules
                                // must be Boolean
});

assert( validate(x) === true );
```

### Self-referencing ###

The easiest way to do self-referencing is using `schema.self`. However, to support a more
intuitive notation (as seen in the Tree example above) there is an other way to reference
the schema that is being described. When executing this:

```javascript
var Tree = schema({ left : [ Number, Tree ], right : [ Number, Tree ] });
```

js-schema sees in fact `{ left : [ Number, undefined ], right : [ Number, undefined ] }` as first
parameter, since the value of the `Tree` variable is undefined when the schema function is
called. Consider the meaning of `[ Number, undefined ]` according to the rules described above:
'this property must be either Number, or anything else'. It doesn't make much sense to include
'anything else' in an 'or' relation. If js-schema sees `undefined` in an or relation, it assumes
that this is in fact a self-reference.

Use this feature carefully, because it may easily lead to bugs. Only use it when the return value
of the schema function is assigned to a newly defined variable.

Extensions
==========

### Numbers ###

There are five functions that can be used for describing number ranges: `min`, `max`, `below`,
`above` and `step`. All of these are chainable, so for example `Number.min(a).below(b)` matches `x`
if `a <= x && x < b`. The `Number.step(a)` matches `x` if `x` is a divisible by `a`.

### Strings ###

The `String.of` method has three signatures:
- `String.of(charset)` matches `x` if it is a string and contains characters that are included in `charset`
- `String.of(length, charset)` additionally checks the length of the instance and returns true only if it equals to `length`.
- `String.of(minLength, maxLength, charset)` is similar, but checks if the length is in the given interval.

`charset` must be given in a format that can be directly inserted in a regular expression when
wrapped by `[]`. For example, `'abc'` means a character set containing the first 3 lowercase letters
of the english alphabet, while `'a-zA-C'` means a character set of all english lowercase letters,
and the first 3 uppercase letters. If `charset` is `undefined` then the `a-zA-Z0-9` character set
is used.

### Arrays ###

The `Array.like(array)` matches `x` if `x instanceof Array` and it deep equals `array`.

The `Array.of` method has three signatures:
- `Array.of(pattern)` matches `x` if `x instanceof Array` and `pattern` matches every element of `x`.
- `Array.of(length, pattern)` additionally checks the length of the instance and returns true only if it equals to `length`.
- `Array.of(minLength, maxLength, pattern)` is similar, but checks if the length is in the given interval.

### Objects ###

`Object.reference(object)` matches `x` if `x === object`.

`Object.like(object)` matches `x` if `x` deep equals `object`.

### Functions ###

`Function.reference(func)` matches `x` if `x === func`.

Future plans
============

Better JSON Schema support. js-schema should be able to parse any valid JSON schema and generate
JSON Schema for most of the patterns (this is not possible in general, because of patterns that hold
external references like the 'instanceof' pattern).

Contributing
============

Feel free to open an issue or send a pull request if you would like to help improving js-schema or find a bug.

People who made significant contributions so far:

 * [Alan James](https://github.com/alanjames1987)
 * [Kuba Wyrobek](https://github.com/parhelium)

Installation
============

Using [npm](http://npmjs.org):

    npm install js-schema

Using [bower](http://bower.io):

    bower install js-schema

Build
=====

To build the browser version you will need node.js and the development dependencies that can be
installed with npm (type `npm install ./` in the project directory). `build.sh`
assembles a debug version using browserify and then minifies it using uglify.

License
=======

The MIT License

Copyright (C) 2012 Gábor Molnár <gabor@molnar.es>
