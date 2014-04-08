# express-form

Express Form provides data filtering and validation as route middleware to your Express applications.

[![Build Status](https://travis-ci.org/freewil/express-form.png?branch=master)](https://travis-ci.org/freewil/express-form)

## Install

* **Express 2.x** `npm install express-form@0.9.x`

* **Express 3.x** `npm install express-form@0.10.x`

## Usage

```js
var express = require('express'),
    form = require('express-form'),
    field = form.field;

var app = express();

app.configure(function() {
  app.use(express.bodyDecoder());
  app.use(app.router);
});

app.post(

  // Route
  '/user',

  // Form filter and validation middleware
  form(
    field("username").trim().required().is(/^[a-z]+$/),
    field("password").trim().required().is(/^[0-9]+$/),
    field("email").trim().isEmail()
   ),

   // Express request-handler now receives filtered and validated data
   function(req, res){
     if (!req.form.isValid) {
       // Handle errors
       console.log(req.form.errors);

     } else {
       // Or, use filtered form data from the form object:
       console.log("Username:", req.form.username);
       console.log("Password:", req.form.password);
       console.log("Email:", req.form.email);
     }
  }
);

app.listen(3000);
```

## Documentation

### Module

`express-form` returns an `express` [Route Middleware](http://expressjs.com/guide.html#Route-Middleware) function.
You specify filtering and validation by passing filters and validators as
arguments to the main module function. For example:

```js
var form = require("express-form");

app.post('/user',

  // Express Form Route Middleware: trims whitespace off of
  // the `username` field.
  form(form.field("username").trim()),

  // standard Express handler
  function(req, res) {
    // ...
  }
);
```

### Fields

The `field` property of the module creates a filter/validator object tied to a specific field.

```
field(fieldname[, label]);
```

You can access nested properties with either dot or square-bracket notation.

```js
field("post.content").minLength(50),
field("post[user][id]").isInt(),
field("post.super.nested.property").required()
```

Simply specifying a property like this, makes sure it exists. So, even if `req.body.post` was undefined,
`req.form.post.content` would be defined. This helps avoid any unwanted errors in your code.

The API is chainable, so you can keep calling filter/validator methods one after the other:

```js
field("username")
  .required()
  .trim()
  .toLower()
  .truncate(5)
  .isAlphanumeric()
```

### Filter API

####Type coercion
```js
.toFloat()                 // -> Number

.toInt()                   // -> Number, rounded down

.toBoolean()               // -> Boolean from truthy and falsy values

.toBooleanStrict()         // -> Only true, "true", 1 and "1" are `true`

.ifNull(replacement)       // -> "", undefined and null get replaced by `replacement`

.ifNullOrNaN(replacement)  // -> "", undefined, null, NaN get replaced by `replacement`
```

If you intend to both validate and coerce numeric fields using `toInt()` or `toFloat()`, then you have two distinct approaches:
* `.ifNull(0).toInt()` will ensure empty fields are zero, but non-numeric input will still return null, allowing you the option to validate with `.isInt()`.
* `.toInt().ifNullOrNaN(0)` will ensure the field has a valid integral value whatever type is entered, defaulting to 0.

For array coercion, see [Array coercion/validation](#array-coercionvalidation) below.

####HTML encoding for `& " < >`
```js
.entityEncode()            // -> encodes HTML entities

.entityDecode()            // -> decodes HTML entities
```

####String transformations
```js
.trim(chars)               // -> `chars` defaults to whitespace

.ltrim(chars)

.rtrim(chars)

.toLower()                 // alias: toLowerCase()

.toUpper()                 // alias: toUpperCase()

.truncate(length)          // -> Chops value at (length - 3), appends `...`
```

### Validator API

####Custom error messages

Each validator has its own default validation message.
These can easily be overridden at runtime by passing a custom validation message
to the validator. The custom message is always the **last** argument passed to
the validator. `required()` allows you to set a placeholder (or default value)
that your form contains when originally presented to the user. This prevents the
placeholder value from passing the `required()` check.

Use "%s" in the message to have the field name or label printed in the message:
```js
.required()
// -> "username is required"

.required("Type your desired username", "What is your %s?")
// -> "What is your username?"

field("uid", "Username").required("", "What is your %s?")
// -> "What is your Username?"
```

####Required field validation

Check that the field is present in form data, and has a value:
```js
.required([message])
```

####Type validation
```js
.isNumeric([message])

.isInt([message])

.isDecimal([message])

.isFloat([message])
```

####Format validation
```js
.isDate([message])

.isEmail([message])

.isUrl([message])

.isIP([message])

.isAlpha([message])

.isAlphanumeric([message])

.isLowercase([message])

.isUppercase([message])
```

####Content validation
```js
.notEmpty([message])

    Checks if the value is not just whitespace.

.equals( value [, message] )
- value (String): A value that should match the field value OR a fieldname
                  token to match another field, ie, `field::password`.

    Compares the field to `value`.

    Example:
    field("username").equals("admin")

    field("password").is(/^\w{6,20}$/)
    field("password_confirmation").equals("field::password")


.contains(value[, message])
- value (String): The value to test for.

    Checks if the field contains `value`.


.notContains(string[, message])
- value (String): A value that should not exist in the field.

    Checks if the field does NOT contain `value`.
```

####Regular expression validation
```js
.regex(pattern[, modifiers[, message]])
- pattern (RegExp|String): RegExp (with flags) or String pattern.
- modifiers (String): Optional, and only if `pattern` is a String.
- message (String): Optional validation message.

    alias: is

    Checks that the value matches the given regular expression.

    Example:

    field("username").is("[a-z]", "i", "Only letters are valid in %s")
    field("username").is(/[a-z]/i, "Only letters are valid in %s")


.notRegex(pattern[, modifiers[, message]])
- pattern (RegExp|String): RegExp (with flags) or String pattern.
- modifiers (String): Optional, and only if `pattern` is a String.
- message (String): Optional validation message.

    alias: not

    Checks that the value does NOT match the given regular expression.

    Example:

    field("username").not("[a-z]", "i", "Letters are not valid in %s")
    field("username").not(/[a-z]/i, "Letters are not valid in %s")
```

### Array coercion/validation

```js
.array([min, max [, message])
```

Using the `array()` method means that field *always* returns an array. If the field value is an array, but you don't use this method, then the first value in that array is used instead for any validation/filtering.

Coercing with `array()` means that you can safely use array methods on post data that might otherwise break your code, such as when the form actually returns a string.

*Examples*
```js
field("project.users").array(),
// undefined => [], "" => [], "q" => ["q"], ["a", "b"] => ["a", "b"]

field("project.block"),
// project.block: ["a", "b"] => "a". No "array()", so only first value used.
```

In addition, any other methods called with the array method, are applied to every value within the array:

```js
field("post.users").array().toUpper()
// post.users: ["one", "two", "three"] => ["ONE", "TWO", "THREE"]
```

You can optionally pass a pair of numeric arguments to array() to specify a minimum and maximum valid array length, and a further optional argument for a custom message to display if the length is out of range:

```js
field("post.compulsoryCourses").array(1,30)
// undefined => error, [] => error, ["Javascript 101"] => ["Javascript 101"]
```

### Custom Methods

```js
.custom(function(value[, source]) {
  // ...
})
```
The `custom()` method lets you create custom filters or validation by passing a function. It has one required argument, which is substituted with the field value on submission. You can optionally reference a second argument in your function, whose value is the contents of the `dataSources` defined in the [configuration](#configuration). This allows your code to look up values of other form fields, for example when you need to validate combinations of fields.


* If the function throws an error, then an error is added to the form. (If `message` is not provided, the thrown error message is used.)
* If the function returns a value, then it is considered a filter method, with the field then becoming the returned value.
* If the function returns undefined, then the method has no effect on the field.

*Examples*

If the `name` field has a value of "hello there", this would transform it to "hello-there":
```js
field("name").custom(function(value) {
  return value.replace(/\s+/g, "-");
});
```
Throw an error if `username` field does not have value "admin":
```js
field("username").custom(function(value) {
    if (value !== "admin") {
        throw new Error("%s must be 'admin'.");
    }
});
```
Validate `sport` field depending on the value on another field:
```js
field("sport", "favorite sport").custom(function(value, source) {
  if (!source.country) {
    throw new Error('unable to validate %s');
  }

  switch (source.country) {
    case 'US':
      if (value !=== 'baseball') {
        throw new Error('America likes baseball');
      }
      break;

    case 'UK':
      if (value !=== 'football') {
        throw new Error('UK likes football');
      }
      break;
  }
});
```
Asynchronous custom validator (3 argument function signature):
```js
form.field('username').custom(function(value, source, callback) {
  username.check(value, function(err) {
    if (err) return callback(new Error('Invalid %s'));
    callback(null);
  });
});
```

### http.ServerRequest.prototype.form

Express Form adds a `form` object with various properties to the request.
```js
isValid -> Boolean

errors  -> Array

flashErrors(function) -> undefined
// Method used to flash error messages. Defaults to use req.flash()

getErrors(fieldname) -> Array (or Object if no name given)
// Gets all errors for the field with the given name.
// You can also call this method with no parameters to get a map of errors for all of the fields.
```
Example request handler:
```js
function(req, res) {
  if (!req.form.isValid) {
    console.log(req.errors);
    console.log(req.getErrors("username"));
    console.log(req.getErrors());
  }
}
```
### Configuration

Express Form has various configuration options, but aims for sensible defaults for a typical Express application.
```js
form.configure(options) -> self

options (Object) {
  flashErrors,
  // (Boolean) If validation errors should be automatically passed to Express’ flash() method.
  // For Express 3.x, this requires the `connect-flash` module, or an overridden flashErrors method on the form.
  // Default: true
  autoLocals,
  // (Boolean) If field values from Express’ request.body should be passed into Express’ response.locals object.
  // This is helpful when a form is invalid an you want to repopulate the form elements with their submitted values.
  // If a field name is dash-separated, the name used for the locals object will be in camelCase.
  // Default: true
  dataSources,
  // (Array) An array of Express request properties to use as data sources when filtering and validating data.
  // Default: ["body", "query", "params"]
  autoTrim,
  // (Boolean) If true, all fields will be automatically trimmed.
  // Default: false
  passThrough
  // (Boolean) If true, all data sources will be merged with `req.form`.
  // Default: false
}
```
### Credits

Currently, Express Form uses many of the validation and filtering functions provided by Chris O'Hara's [node-validator](https://github.com/chriso/node-validator).
