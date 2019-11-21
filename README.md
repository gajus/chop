# Surgeon

[![GitSpo Mentions](https://gitspo.com/badges/mentions/gajus/surgeon?style=flat-square)](https://gitspo.com/mentions/gajus/surgeon)
[![Travis build status](http://img.shields.io/travis/gajus/surgeon/master.svg?style=flat-square)](https://travis-ci.org/gajus/surgeon)
[![Coveralls](https://img.shields.io/coveralls/gajus/surgeon.svg?style=flat-square)](https://coveralls.io/github/gajus/surgeon)
[![NPM version](http://img.shields.io/npm/v/surgeon.svg?style=flat-square)](https://www.npmjs.org/package/surgeon)
[![Canonical Code Style](https://img.shields.io/badge/code%20style-canonical-blue.svg?style=flat-square)](https://github.com/gajus/canonical)
[![Twitter Follow](https://img.shields.io/twitter/follow/kuizinas.svg?style=social&label=Follow)](https://twitter.com/kuizinas)

Declarative DOM extraction expression evaluator.

Powerful, succinct, composable, extendable, declarative API.

```yaml
articles:
- select article {0,}
- body:
  - select .body
  - read property innerHTML
  imageUrl:
  - select img
  - read attribute src
  summary:
  - select ".body p:first-child"
  - read property innerHTML
  - format text
  title:
  - select .title
  - read property textContent
pageName:
- select .body
- read property innerHTML

```

> Not succinct enough for you? Use [aliases](#declare-subroutine-aliases) and the [pipe operator (`|`)](#the-pipe-operator-) to shorten and concatenate the commands:
>
> ```
> articles:
> - sm article
> - body: s .body | rp innerHTML
>   imageUrl: s img | ra src
>   summary: s .body p:first-child | rp innerHTML | f text
>   title: s .title | rp textContent
> pageName: s .body | rp innerHTML
>
> ```

Have you got suggestions for improvement? [I am all ears](https://github.com/gajus/surgeon/issues).

---

* [Configuration](#configuration)
* [Evaluators](#evaluators)
  * [`browser` evaluator](#browser-evaluator)
  * [`cheerio` evaluator](#cheerio-evaluator)
* [Subroutines](#subroutines)
  * [Built-in subroutines](#built-in-subroutines)
    * [`append` subroutine](#append-subroutine)
    * [`closest` subroutine](#closest-subroutine)
    * [`constant` subroutine](#constant-subroutine)
    * [`format` subroutine](#format-subroutine)
    * [`match` subroutine](#match-subroutine)
    * [`nextUntil` subroutine](#nextuntil-subroutine)
    * [`prepend` subroutine](#prepend-subroutine)
    * [`previous` subroutine](#previous-subroutine)
    * [`read` subroutine](#read-subroutine)
    * [`remove` subroutine](#remove-subroutine)
    * [`select` subroutine](#select-subroutine)
      * [Quantifier expression](#quantifier-expression)
    * [`test` subroutine](#test-subroutine)
  * [User-defined subroutines](#user-defined-subroutines)
  * [Inline subroutines](#inline-subroutines)
  * [Built-in subroutine aliases](#built-in-subroutine-aliases)
* [Expression reference](#expression-reference)
  * [The pipe operator (`|`)](#the-pipe-operator-)
* [Cookbook](#cookbook)
  * [Extract a single node](#extract-a-single-node)
  * [Extract multiple nodes](#extract-multiple-nodes)
  * [Name results](#name-results)
  * [Validate the results using RegExp](#validate-the-results-using-regexp)
  * [Validate the results using a user-defined test function](#validate-the-results-using-a-user-defined-test-function)
  * [Declare subroutine aliases](#declare-subroutine-aliases)
* [Error handling](#error-handling)
* [Debugging](#debugging)

## Configuration

|Name|Type|Description|Default value|
|---|---|---|---|
|`evaluator`|[`EvaluatorType`](./src/types.js)|HTML parser and selector engine. See [evaluators](#evaluators).|[`browser` evaluator](#browser-evaluator) if `window` and `document` variables are present, [`cheerio`](#cheerio-evaluator) otherwise.|
|`subroutines`|[`$PropertyType<UserConfigurationType, 'subroutines'>`](./src/types.js)|User defined subroutines. See [subroutines](#subroutines).|N/A|

## Evaluators

[Subroutines](#subroutines) use an evaluator to parse input (i.e. convert a string into an object) and to select nodes in the resulting document.

The default evaluator is configured based on the user environment:
  * [`browser` evaluator](#browser-evaluator) is used if `window` and `document` variables are defined; otherwise
  * [`cheerio`](#cheerio-evaluator)

> Have a use case for another evaluator? [Raise an issue](https://github.com/gajus/surgeon/issues).
>
> For an example implementation of an evaluator, refer to:
>
> * [`./src/evaluators/browserEvaluator.js`](./src/evaluators/browserEvaluator.js)
> * [`./src/evaluators/cheerioEvaluator.js`](./src/evaluators/cheerioEvaluator.js)

### `browser` evaluator

Uses native browser methods to parse the document and to evaluate CSS selector queries.

Use `browser` evaluator if you are running Surgeon in a browser or a headless browser (e.g. PhantomJS).

```js
import {
  browserEvaluator
} from './evaluators';

surgeon({
  evaluator: browserEvaluator()
});

```

### `cheerio` evaluator

Uses [cheerio](https://github.com/cheeriojs/cheerio) to parse the document and to evaluate CSS selector queries.

Use `cheerio` evaluator if you are running Surgeon in Node.js.

```js
import {
  cheerioEvaluator
} from './evaluators';

surgeon({
  evaluator: cheerioEvaluator()
});

```

## Subroutines

A subroutine is a function used to advance the DOM extraction expression evaluator, e.g.

```js
await x('foo | bar baz', 'qux');

```

In the above example, Surgeon expression uses two subroutines: `foo` and `bar`.

`foo` subroutine is invoked without additional values. `bar` subroutine is executed with 1 value ("baz").

Subroutines are executed in the order in which they are defined – the result of the last subroutine is passed on to the next one. The first subroutine receives the document input (in this case: "qux" string).

Multiple subroutines can be written as an array. The following example is equivalent to the earlier example.

```js
await x([
  'foo',
  'bar baz'
], 'qux');

```

There are two types of subroutines:

* [Built-in subroutines](#built-in-subroutines)
* [User-defined subroutines](#user-defined-subroutines)

> Note:
>
> These functions are called subroutines to emphasise the cross-platform nature of the declarative API.

### Built-in subroutines

The following subroutines are available out of the box.

#### `append` subroutine

`append` appends a string to the input string.

|Parameter name|Description|Default|
|---|---|---|
|tail|Appends a string to the end of the input string.|N/A|

Examples:

```js
// Assuming an element <a href='http://foo' />,
// then the result is 'http://foo/bar'.
await x(`select a | read attribute href | append '/bar'`);

```

#### `closest` subroutine

`closest` subroutine iterates through all the preceding nodes (including parent nodes) searching for either a preceding node matching the selector expression or a descendant of the preceding node matching the selector.

Note: This is different from the jQuery [`.closest()`](https://api.jquery.com/closest/) in that the latter method does not search for parent descendants matching the selector.

|Parameter name|Description|Default|
|---|---|---|
|CSS selector|CSS selector used to select an element.|N/A|

#### `constant` subroutine

`constant` returns the parameter value regardless of the input.

|Parameter name|Description|Default|
|---|---|---|
|`constant`|Constant value that will be returned as the result.|N/A|

#### `format` subroutine

`format` is used to format input using [printf](https://en.wikipedia.org/wiki/Printf_format_string).

|Parameter name|Description|Default|
|---|---|---|
|format|[sprintf format](https://www.npmjs.com/package/sprintf-js) used to format the input string. The subroutine input is the first argument, i.e. `%1$s`.|`%1$s`|

Examples:

```js
// Extracts 1 matching capturing group from the input string.
// Prefixes the match with 'http://foo.com'.
await x(`select a | read attribute href | format 'http://foo.com%1$s'`);

```

#### `match` subroutine

`match` is used to extract matching [capturing groups](https://www.regular-expressions.info/refcapture.html) from the subject input.

|Parameter name|Description|Default|
|---|---|---|
|Regular expression|Regular expression used to match capturing groups in the string.|N/A|
|Sprintf format|[sprintf format](https://www.npmjs.com/package/sprintf-js) used to construct a string using the matching capturing groups.|`%s`|

Examples:

```js
// Extracts 1 matching capturing group from the input string.
// Throws `InvalidDataError` if the value does not pass the test.
await x('select .foo | read property textContent | match "/input: (\d+)/"');

// Extracts 2 matching capturing groups from the input string and formats the output using sprintf.
// Throws `InvalidDataError` if the value does not pass the test.
await x('select .foo | read property textContent | match "/input: (\d+)-(\d+)/" %2$s-%1$s');

```

#### `nextUntil` subroutine

`nextUntil` subroutine is used to select all following siblings of each element up to but not including the element matched by the selector.

|Parameter name|Description|Default|
|---|---|---|
|selector expression|A string containing a selector expression to indicate where to stop matching following sibling elements.|N/A|
|filter expression|A string containing a selector expression to match elements against.|

#### `prepend` subroutine

`prepend` prepends a string to the input string.

|Parameter name|Description|Default|
|---|---|---|
|head|Prepends a string to the start of the input string.|N/A|

Examples:

```js
// Assuming an element <a href='//foo' />,
// then the result is 'http://foo/bar'.
await x(`select a | read attribute href | prepend 'http:'`);

```

#### `previous` subroutine

`previous` subroutine selects the preceding sibling.

|Parameter name|Description|Default|
|---|---|---|
|CSS selector|CSS selector used to select an element.|N/A|

Example:

```html
<ul>
  <li>foo</li>
  <li class='bar'></li>
<ul>
```

```js
await x('select .bar | previous | read property textContent');
// 'foo'

```

#### `read` subroutine

`read` is used to extract value from the matching element using an [evaluator](#evaluators).

|Parameter name|Description|Default|
|---|---|---|
|Target type|Possible values: "attribute" or "property"|N/A|
|Target name|Depending on the target type, name of an attribute or a property.|N/A|

Examples:

```js
// Returns .foo element "href" attribute value.
// Throws error if attribute does not exist.
await x('select .foo | read attribute href');

// Returns an array of "href" attribute values of the matching elements.
// Throws error if attribute does not exist on either of the matching elements.
await x('select .foo {0,} | read attribute href');

// Returns .foo element "textContent" property value.
// Throws error if property does not exist.
await x('select .foo | read property textContent');

```

#### `remove` subroutine

`remove` subroutine is used to remove elements from the document using an [evaluator](#evaluators).

`remove` subroutine accepts the same parameters as the `select` subroutine.

The result of `remove` subroutine is the input of the subroutine, i.e. previous `select` subroutine result.

|Parameter name|Description|Default|
|---|---|---|
|CSS selector|CSS selector used to select an element.|N/A|
|Quantifier expression|A [quantifier expression](#quantifier-expression) is used to control the expected result length.|See [quantifier expression](#quantifier-expression).|

Examples:

```js
// Returns 'bar'.
await x('select .foo | remove span | read property textContent', `<div class='foo'>bar<span>baz</span></div>`);

```

#### `select` subroutine

`select` subroutine is used to select the elements in the document using an [evaluator](#evaluators).

|Parameter name|Description|Default|
|---|---|---|
|CSS selector|CSS selector used to select an element.|N/A|
|Quantifier expression|A [quantifier expression](#quantifier-expression) is used to control the shape of the results (direct result or array of results) and the expected result length.|See [quantifier expression](#quantifier-expression).|

##### Quantifier expression

A *quantifier expression* is used to assert that the query matches a set number of nodes. A quantifier expression is a modifier of the [`select` subroutine](#select-subroutine).

A *quantifier expression* is defined using the following syntax.

|Name|Syntax|
|---|---|
|Fixed quantifier|`{n}` where `n` is an integer `>= 1`|
|Greedy quantifier|`{n,m}` where `n >= 0` and `m >= n`|
|Greedy quantifier|`{n,}` where `n >= 0`|
|Greedy quantifier|`{,m}` where `m >= 1`|

A *quantifier expression* can be appended a node selector `[i]`, e.g. `{0,}[1]`. This allows to return the first node from the result set.

> If this looks familiar, its because I have adopted the syntax from regular expression language. However, unlike in regular expression, a quantifier in the context of Surgeon selector will produce an error (`SelectSubroutineUnexpectedResultCountError`) if selector result length is out of the quantifier range.

Examples:

```js
// Selects 0 or more nodes.
// Result is an array.
await x('select .foo {0,}');

// Selects 1 or more nodes.
// Throws an error if 0 matches found.
// Result is an array.
await x('select .foo {1,}');

// Selects between 0 and 5 nodes.
// Throws an error if more than 5 matches found.
// Result is an array.
await x('select .foo {0,5}');

// Selects 1 node.
// Result is the first match in the result set (or `null`).
await x('select .foo {0,}[0]');

```

#### `test` subroutine

`test` is used to validate the current value using a regular expression.

|Parameter name|Description|Default|
|---|---|---|
|Regular expression|Regular expression used to test the value.|N/A|

Examples:

```js
// Validates that .foo element textContent property value matches /bar/ regular expression.
// Throws `InvalidDataError` if the value does not pass the test.
await x('select .foo | read property textContent | test /bar/');

```

See [error handling](#error-handling) for more information and usage examples of the `test` subroutine.

### User-defined subroutines

Custom subroutines can be defined using [`subroutines` configuration](#configuration).

A subroutine is a function. A subroutine function is invoked with the following parameters:

|Parameter name|
|---|
|An instance of [Evaluator].|
|Current value, i.e. value used to query Surgeon or value returned from the previous (or ancestor) subroutine.|
|An array of values used when referencing the subroutine in an expression.|

Example:

```js
const x = surgeon({
  subroutines: {
    mySubroutine: (currentValue, [firstParameterValue, secondParameterValue]) => {
      console.log(currentValue, firstParameterValue, secondParameterValue);

      return parseInt(currentValue, 10) + 1;
    }
  }
});

await x('mySubroutine foo bar | mySubroutine baz qux', 0);

```

The above example prints:

```
0 "foo" "bar"
1 "baz" "qux"

```

For more examples of defining subroutines, refer to:

* [Validate the results using a user-defined test function](#validate-the-results-using-a-user-defined-test-function).
* [Source code](./src/subroutines) of the the built-in subroutines.

## Inline subroutines

Custom subroutines can be inlined into [pianola](https://github.com/gajus/pianola) instructions, e.g.

```js
await x(
  [
    'foo',
    (subject) => {
      // `subject` is the return value of `foo` subroutine.

      return 'bar';
    },
    'baz',
  ],
  'qux'
);

```

## Built-in subroutine aliases

Surgeon exports an alias preset is used to reduce verbosity of the queries.

|Name|Description|
|---|---|
|`ra ...`|Reads Element attribute value. Equivalent to `read attribute ...`|
|`rdtc ...`|Removes any descending elements and reads the resulting `textContent` property of an element. Equivalent to `remove * {0,} | read property ... textContent`|
|`rih ...`|Reads `innerHTML` property of an element. Equivalent to `read property ... innerHTML`|
|`roh ...`|Reads `outerHTML` property of an element. Equivalent to `read property ... outerHTML`|
|`rp ...`|Reads Element property value. Equivalent to `read property ...`|
|`rtc ...`|Reads `textContent` property of an element. Equivalent to `read property ... textContent`|
|`sa ...`|Select any (sa). Selects multiple elements (0 or more). Returns array. Equivalent to `select "..." {0,}`|
|`saf ...`|Select any first (saf). Selects multiple elements (0 or more). Returns single result or `null`. Equivalent to `select "..." {0,}[0]`|
|`sm ...`|Select many (sm). Selects multiple elements (1 or more). Returns array. Equivalent to `select "..." {1,}`|
|`smo ...`|Select maybe one (smo). Selects one element. Returns single result or `null`. Equivalent to `select "..." {0,1}[0]`|
|`so ...`|Select one (so). Selects a single element. Returns single result. Equivalent to `select "..." {1}[0]`.|
|`t {name}`|Tests value. Equivalent to `test ...`|

> Note regarding `s ...` alias. The CSS selector value is quoted. Therefore, you can write a CSS selector that includes spaces without putting the value in the quotes, e.g. `s .foo .bar` is equivalent to `select ".foo .bar" {1}`.
>
> Other alias values are not quoted. Therefore, if value includes a space it must be quoted, e.g. `t "/foo bar/"`.

Usage:

```js
import surgeon, {
  subroutineAliasPreset
} from 'surgeon';

const x = surgeon({
  subroutines: {
    ...subroutineAliasPreset
  }
});

await x('s .foo .bar | t "/foo bar/"');

```

In addition to the built-in aliases, user can [declare subroutine aliases](#declare-subroutine-aliases).

## Expression reference

Surgeon subroutines are referenced using expressions.

An expression is defined using the following pseudo-grammar:

```
subroutines ->
    subroutines _ "|" _ subroutine
  | subroutine

subroutine ->
    subroutineName " " parameters
  | subroutineName

subroutineName ->
  [a-zA-Z0-9\-_]:+

parameters ->
    parameters " " parameter
  | parameter

```

Example:

```js
await x('foo bar baz', 'qux');

```

In this example, Surgeon query executor (`x`) is invoked with `foo bar baz` expression and `qux` starting value. The expression tells the query executor to run `foo` subroutine with parameter values "bar" and "baz". The expression executor runs `foo` subroutine with parameter values "bar" and "baz" and subject value "qux".

Multiple subroutines can be combined using an array:

```js
await x([
  'foo bar baz',
  'corge grault garply'
], 'qux');

```

In this example, Surgeon query executor (`x`) is invoked with two expressions (`foo bar baz` and `corge grault garply`). The first subroutine is executed with the subject value "qux". The second subroutine is executed with a value that is the result of the parent subroutine.

The result of the query is the result of the last subroutine.

Read [user-defined subroutines](#user-defined-subroutines) documentation for broader explanation of the role of the parameter values and the subject value.

### The pipe operator (`|`)

Multiple subroutines can be combined using the pipe operator.

The following examples are equivalent:

```js
await x([
  'foo bar baz',
  'qux quux quuz'
]);

await x([
  'foo bar baz | foo bar baz'
]);

await x('foo bar baz | foo bar baz');

```

## Cookbook

Unless redefined, all examples assume the following initialisation:

```js
import surgeon from 'surgeon';

/**
 * @param configuration {@see https://github.com/gajus/surgeon#configuration}
 */
const x = surgeon();

```

### Extract a single node

Use [`select` subroutine](#select-subroutine) and [`read` subroutine](#read-subroutine) to extract a single value.

```js
const subject = `
  <div class="title">foo</div>
`;

await x('select .title | read property textContent', subject);

// 'foo'

```

### Extract multiple nodes

Specify [`select` subroutine](#select-subroutine) `quantifier` to match multiple results.

```js
const subject = `
  <div class="foo">bar</div>
  <div class="foo">baz</div>
  <div class="foo">qux</div>
`;

await x('select .title {0,} | read property textContent', subject);

// [
//   'bar',
//   'baz',
//   'qux'
// ]

```

### Name results

Use a [`QueryChildrenType`](./src/types.js) object to name the results of the descending expressions.

```js
const subject = `
  <article>
    <div class='title'>foo title</div>
    <div class='body'>foo body</div>
  </article>
  <article>
    <div class='title'>bar title</div>
    <div class='body'>bar body</div>
  </article>
`;

await x([
  'select article',
  {
    body: 'select .body | read property textContent'
    title: 'select .title | read property textContent'
  }
]);

// [
//   {
//     body: 'foo body',
//     title: 'foo title'
//   },
//   {
//     body: 'bar body',
//     title: 'bar title'
//   }
// ]

```

### Validate the results using RegExp

Use [`test` subroutine](#test-subroutine) to validate the results.

```js
const subject = `
  <div class="foo">bar</div>
  <div class="foo">baz</div>
  <div class="foo">qux</div>
`;

await x('select .foo {0,} | test /^[a-z]{3}$/');

```

See [error handling](#error-handling) for information how to handle `test` subroutine errors.

### Validate the results using a user-defined test function

Define a [custom subroutine](#user-defiend-subroutines) to validate results using arbitrary logic.

Use `InvalidValueSentinel` to leverage standardised Surgeon error handler (see [error handling](#error-handling)). Otherwise, simply throw an error.

```js
import surgeon, {
  InvalidValueSentinel
} from 'surgeon';

const x = surgeon({
  subroutines: {
    isRed: (value) => {
      if (value === 'red') {
        return value;
      };

      return new InvalidValueSentinel('Unexpected color.');
    }
  }
});

```

### Declare subroutine aliases

As you become familiar with the query execution mechanism, typing long expressions (such as `select`, `read attribute` and `read property`) becomes a mundane task.

Remember that subroutines are regular functions: you can partially apply and use the partially applied functions to create new subroutines.

Example:

```js
import surgeon, {
  readSubroutine,
  selectSubroutine,
  testSubroutine
} from 'surgeon';

const x = surgeon({
  subroutines: {
    ra: (subject, values, bindle) => {
      return readSubroutine(subject, ['attribute'].concat(values), bindle);
    },
    rp: (subject, values, bindle) => {
      return readSubroutine(subject, ['property'].concat(values), bindle);
    },
    s: (subject, values, bindle) => {
      return selectSubroutine(subject, [values.join(' '), '{1}'], bindle);
    },
    sm: (subject, values, bindle) => {
      return selectSubroutine(subject, [values.join(' '), '{0,}'], bindle);
    },
    t: testSubroutine
  }
});

```

Now, instead of writing:

```yaml
articles:
- select article
- body:
  - select .body
  - read property innerHTML

```

You can write:

```yaml
articles:
- sm article
- body:
  - s .body
  - rp innerHTML
```

The aliases used in this example are available in the aliases preset (read [built-in subroutine aliases](#built-in-subroutine-aliases)).

## Error handling

Surgeon throws the following errors to indicate a predictable error state. All Surgeon errors can be imported. Use `instanceof` operator to determine the error type.

> Note:
>
> Surgeon errors are non-recoverable, i.e. a selector cannot proceed if it encounters an error.
> This design ensures that your selectors are capturing the expected data.

|Name|Description|
|---|---|
|`ReadSubroutineNotFoundError`|Thrown when an attempt is made to retrieve a non-existent attribute or property.|
|`SelectSubroutineUnexpectedResultCountError`|Thrown when a [`select` subroutine](#select-subroutine) result length does not match the quantifier expression.|
|`InvalidDataError`|Thrown when a subroutine returns an instance of `InvalidValueSentinel`.|
|`SurgeonError`|A generic error. All other Surgeon errors extend from `SurgeonError`.|

Example:

```js
import {
  InvalidDataError
} from 'surgeon';

const subject = `
  <div class="foo">bar</div>
`;

try {
  await x('select .foo | test /bar/', subject);
} catch (error) {
  if (error instanceof InvalidDataError) {
    // Handle data validation error.
  } else {
    throw error;
  }
}

```

Return `InvalidValueSentinel` from a subroutine to force Surgeon throw `InvalidDataError` error.

## Debugging

Surgeon is using [`roarr`](https://www.npmjs.com/package/roarr) to log debugging information.

Export `ROARR_LOG=TRUE` environment variable to enable Surgeon debug log.
