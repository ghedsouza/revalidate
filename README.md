# revalidate

[![Travis branch](https://img.shields.io/travis/jfairbank/revalidate/master.svg?style=flat-square)](https://travis-ci.org/jfairbank/revalidate)
[![npm](https://img.shields.io/npm/v/revalidate.svg?style=flat-square)](https://www.npmjs.com/package/revalidate)

Composable form value validations for JavaScript.

Revalidate was originally created as a helper library for composing and reusing
common validations to generate validate functions for
[redux-form](https://github.com/erikras/redux-form). It became evident that the
validators that revalidate can generate are pretty agnostic about how they are
used. They are just functions that take a value and return an error message if
the value is invalid.

## Install

    $ npm install revalidate

## Usage

Revalidate provides functions for creating validation functions as well as
composing and combining them. Think [redux](https://github.com/reactjs/redux)
for validation functions.

### `createValidator`

The simplest function is `createValidator` which creates a value validation
function. `createValidator` takes two arguments. The first argument is a curried
function that takes an error message and the value. The curried function must
return the message if the value is invalid. If the field value is valid, it's
recommended that you return nothing, so a return value of `undefined` implies
the field value was valid.

The second argument is a function that takes a field name and must return the
error message. Optionally, you can just pass in a string as the second argument
if you don't want to depend on the field name.

The returned validation function is also a curried function. The first argument
is a field name string or a configuration object where you can specify the field
or a custom error message. The second argument is the value. You can pass in
both arguments at the same time too. We'll see why currying the function can be
useful when we want to compose validators.

Here is an implementation of an `isRequired` validator with `createValidator`:

```js
import { createValidator } from 'revalidate';

const isRequired = createValidator(
  message => value => {
    if (value == null || value === '') {
      return message;
    }
  },

  field => `${field} is required`
);

isRequired('My Field')();     // 'My Field is required'
isRequired('My Field')('');   // 'My Field is required'
isRequired('My Field')('42'); // undefined, therefore assume valid

// With a custom message
isRequired({ message: 'Error' })(); // 'Error'
```

### `composeValidators`

`composeValidators` is the function that makes revalidate start to be really
useful. As the name suggests, it allows you to compose validators into one. The
composed validator will check each validator and return the first error message
it encounters. Validators are checked in a left-to-right fashion to make them
more readable. (**Note:** this is opposite most functional implementations of the
compose function.)

The composed validator is also curried and takes the same arguments as an
individual validator made with `createValidator`.

```js
import {
  createValidator,
  composeValidators,
  isRequired
} from 'revalidate';

const isAlphabetic = createValidator(
  message => value => {
    if (value && !/^[A-Za-z]+$/.test(value)) {
      return message;
    }
  },

  field => `${field} must be alphabetic`
);

const validator = composeValidators(
  isRequired,

  // You can still customize individual validators
  // because they're curried!
  isAlphabetic({
    message: 'Can only contain letters'
  })
)('My Field');

validator();      // 'My Field is required'
validator('123'); // 'Can only contain letters'
validator('abc'); // undefined
```

You can supply an additional `multiple: true` option to return all potential
errors from your composed validators.

```js
import { createValidator, composeValidators } from 'revalidate';

const startsWithA = createValidator(
  message => value => {
    if (value && !/^A/.test(value)) {
      return message;
    }
  },
  field => `${field} must start with A`
);

const endsWithC = createValidator(
  message => value => {
    if (value && !/C$/.test(value)) {
      return message;
    }
  },
  field => `${field} must end with C`
);

const validator = composeValidators(
  startsWithA,
  endsWithC
)({ field: 'My Field', multiple: true });

validator('BBB');
// [
//   'My Field must start with A',
//   'My Field must end with C'
// ]
```

### `combineValidators`

`combineValidators` is analogous to a function like `combineReducers` from
redux. It allows you to validate multiple field values at once. It returns a
function that takes an object with field names mapped to their values.
`combineValidators` will run named validators you supplied it with their
respective field values and return an object literal containing any error
messages for each field value. An empty object return value implies no field
values were invalid.

```js
import {
  createValidator,
  composeValidators,
  combineValidators,
  isRequired,
  isAlphabetic,
  isNumeric
} from 'revalidate';

const dogValidator = combineValidators({
  // Use composeValidators too!
  name: composeValidators(
    isRequired,
    isAlphabetic
  )('Name'),

  // Don't forget to supply a field name if you
  // don't compose other validators
  age: isNumeric('Age')
});

dogValidator({}); // { name: 'Name is required' }

dogValidator({ name: '123', age: 'abc' });
// { name: 'Name must be alphabetic', age: 'Age must be numeric' }

dogValidator({ name: 'Tucker', age: '10' }); // {}
```

## redux-form

As mentioned, even though revalidate is pretty agnostic about how you use it, it
does work out of the box for redux-form. The `validate` function you might write
for a redux-form example like
[here](http://erikras.github.io/redux-form/#/examples/synchronous-validation?_k=mncrmp)
can also be automatically generated with `combineValidators`. The function it
returns will work perfectly for the `validate` option for your form components
for React and redux-form.

Here is that example from redux-form rewritten to generate a `validate` function
with revalidate.

```js
import React, {Component, PropTypes} from 'react';
import {reduxForm} from 'redux-form';
import {
  createValidator,
  composeValidators,
  combineValidators,
  isRequired,
  hasLengthLessThan,
  isNumeric
} from 'revalidate';

export const fields = ['username', 'email', 'age'];

const isValidEmail = createValidator(
  message => value => {
    if (value && !/^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,4}$/i.test(values.email)) {
      return message;
    }
  },
  'Invalid email address'
);

const isGreaterThan = (n) => createValidator(
  message => value => {
    if (value && Number(value) <= n) {
      return message;
    }
  },
  field => `${field} must be greater than ${n}`
);

const customIsRequired = isRequired({ message: 'Required' });

const validate = combineValidators({
  username: composeValidators(
    customIsRequired,

    hasLengthLessThan(16)({
      message: 'Must be 15 characters or less'
    })
  )(),

  email: composeValidators(
    customIsRequired,
    isValidEmail
  )(),

  age: composeValidators(
    customIsRequired,

    isNumeric({
      message: 'Must be a number'
    }),
    
    isGreaterThan(17)({
      message: 'Sorry, you must be at least 18 years old'
    })
  )()
});

class SynchronousValidationForm extends Component {
  static propTypes = {
    fields: PropTypes.object.isRequired,
    handleSubmit: PropTypes.func.isRequired,
    resetForm: PropTypes.func.isRequired,
    submitting: PropTypes.bool.isRequired
  };

  render() {
    const {fields: {username, email, age}, resetForm, handleSubmit, submitting} = this.props;
    return (<form onSubmit={handleSubmit}>
        <div>
          <label>Username</label>
          <div>
            <input type="text" placeholder="Username" {...username}/>
          </div>
          {username.touched && username.error && <div>{username.error}</div>}
        </div>
        <div>
          <label>Email</label>
          <div>
            <input type="text" placeholder="Email" {...email}/>
          </div>
          {email.touched && email.error && <div>{email.error}</div>}
        </div>
        <div>
          <label>Age</label>
          <div>
            <input type="text" placeholder="Age" {...age}/>
          </div>
          {age.touched && age.error && <div>{age.error}</div>}
        </div>
        <div>
          <button type="submit" disabled={submitting}>
            {submitting ? <i/> : <i/>} Submit
          </button>
          <button type="button" disabled={submitting} onClick={resetForm}>
            Clear Values
          </button>
        </div>
      </form>
    );
  }
}

export default reduxForm({
  form: 'synchronousValidation',
  fields,
  validate
})(SynchronousValidationForm);
```

## Common Validators

Revalidate exports some common validations for your convenience. If you need
something more complex, then you'll need to create your own validators with
`createValidator`.

### `isRequired`

`isRequired` is pretty self explanatory. It determines that a value isn't valid
if it's `null`, `undefined` or the empty string `''`.

```js
isRequired('My Field')();     // 'My Field is required'
isRequired('My Field')(null);     // 'My Field is required'
isRequired('My Field')('');   // 'My Field is required'
isRequired('My Field')('42'); // undefined, therefore assume valid
```

### `hasLengthBetween`

`hasLengthBetween` tests that the value falls between a min and max inclusively.
It wraps a call to `createValidator`, so you must first call it with the min and
max arguments.

```js
hasLengthBetween(1, 3)('My Field')('hello');
// 'My Field must be between 1 and 3 characters long'
```

### `hasLengthGreaterThan`

`hasLengthGreaterThan` tests that the value is greater than a predefined length.
It wraps a call to `createValidator`, so you must first call it with the
min length.

```js
hasLengthGreaterThan(3)('My Field')('foo');
// 'My Field must be longer than 3 characters'
```

### `hasLengthLessThan`

`hasLengthLessThan` tests that the value is less than a predefined length. It
wraps a call to `createValidator`, so you must first call it with the max
length.

```js
hasLengthLessThan(4)('My Field')('hello');
// 'My Field cannot be longer than 4 characters'
```

### `isAlphabetic`

`isAlphabetic` simply tests that the value only contains any of the 26 letters
in the English alphabet.

```js
isAlphabetic('My Field')('1');
// 'My Field must be alphabetic'
```

### `isAlphaNumeric`

`isAlphaNumeric` simply tests that the value only contains any of the 26 letters
in the English alphabet or any numeric digit (i.e. 0-9).

```js
isAlphaNumeric('My Field')('!@#$');
// 'My Field must be alphanumeric'
```

### `isNumeric`

`isNumeric` simply tests that the **string** is comprised of only digits (i.e.
0-9).

```js
isNumeric('My Field')('a');
// 'My Field must be numeric'
```

### `isOneOf`

`isOneOf` tests that the value is contained a predefined array of values. It
wraps a call to `createValidator`, so you must first call it with the array of
allowed values.

```js
isOneOf(['foo', 'bar'])('My Field')('baz');
// 'My Field must be one of ["foo","bar"]'

isOneOf(['foo', 'bar'])('My Field')('FOO');
// 'My Field must be one of ["foo","bar"]'
```

By default it does a sameness equality (i.e. `===`) **with** case sensitivity
for determining if a value is valid. You can supply an optional second argument
function to define how values should be compared. The comparer function takes
the field value as the first argument and each valid value as the second
argument. You could use this to make values case insensitive. Returning a truthy
value in a comparison means that the field value is valid.

```js
const validator = isOneOf(
  ['foo', 'bar'],

  (value, validValue) => (
    value && value.toLowerCase() === validValue.toLowerCase()
  )
);

validator('My Field')('FOO'); // undefined, so valid
```
