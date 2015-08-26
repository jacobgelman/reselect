# reselect
Simple "selector" library for Redux inspired by getters in [NuclearJS](https://github.com/optimizely/nuclear-js.git), [subscriptions](https://github.com/Day8/re-frame#just-a-read-only-cursor) in [re-frame](https://github.com/Day8/re-frame) and this [proposal](https://github.com/gaearon/redux/pull/169) from [speedskater](https://github.com/speedskater).

* Selectors can compute derived data, allowing Redux to store the minimal possible state.
* Selectors are efficient. A selector is not recomputed unless one of its arguments change.
* Selectors are composable. They can be used as input to other selectors. 

## Installation
    npm install reselect

## Example

### Selector Definitions
selectors/ShopSelectors.js
```js

/* 
The data in the Redux store has the following shape:

store: {
  shop: {
    items: [
      {
        name: 'Item 1',
        value: 100
      },
      {
        name: 'Item 2',
        value: 200
      },
      {
        name: 'Item 3',
        value: 300
      }
    ],
    taxPercent: 20
  }
}
*/

import { createSelector } from 'reselect';

/*
 * Definition of simple selectors. 
 * Simple selectors should be used to abstract away the structure
 * of the store in cases where no calculations are needed 
 * and memoization wouldn't provide any benefits.
 */
const shopItemsSelector = state => state.shop.items;
const taxPercentSelector = state => state.shop.taxPercent;

/* 
 * Definition of combined selectors. 
 * In the subsequent examples selectors are combined to derive new information. 
 * To prevent expensive recalculation of these selectors memoization is applied. 
 * Hence, these selectors are only recomputed whenever their input selectors change. 
 * In all other cases the precomputed values are returned.
 */
const subtotalSelector = createSelector(
  shopItemsSelector,
  items => items.reduce((acc, item) => acc + item.value, 0)
);

const taxSelector = createSelector(
  subtotalSelector,
  taxPercentSelector,
  (subtotal, taxPercent) => subtotal * (taxPercent / 100)
);

export const totalSelector = createSelector(
  subtotalSelector,
  taxSelector,
  (subtotal, tax) => { return {total: subtotal + tax}}
);
```

You can use a factory function when you need additional arguments for your selectors:

```js
const expensiveItemSelectorFactory = minValue => {
  return createSelector(
    shopItemsSelector,
    items => items.filter(item => item.value < minValue)
  );
}

const subtotalSelector = createSelector(
  expensiveItemSelectorFactory(200),
  items => items.reduce((acc, item) => acc + item.value, 0)
);
```

### Using Selectors with React Redux

```js

import React from 'react';
import { connect } from 'react-redux';

/*
 * Import the selector defined in the example above.
 * This allows your to separate your components from the structure of your stores.
 */
import { totalSelector } from 'selectors/ShopSelectors';

/*
 * Bind the totalSelector on the Total component.
 * The keys of the selector result are bound to the corresponding component props.
 * In our example there is the 'total' key which is bound to this.props.total
 */
@connect(totalSelector)
class Total extends React.Component {
  render() {
    return <div>{ this.props.total }</div>
  }
}

export default Total;
```


## API Documentation

### createSelector(...inputSelectors, resultFn)
### createSelector([inputSelectors], resultFn)

Takes a variable number or array of selectors whose values are computed and passed as arguments to `resultFn`. A selector has the signature `(state, props) => state`.
```js

const mySelector = createSelector(
  state => state.values.value1,
  state => state.values.value2,
  (value1, value2) => value1 + value2
);

// You can also pass an array of selectors
const totalSelector = createSelector(
  [ 
    state => state.values.value1,
    state => state.values.value2
  ],
  (value1, value2) => value1 + value2
);

// A selector is also passed props as the last parameter into the resultFunc
const selectorWithProps = createSelector(
  state => state.values.value1,
  state => state.values.value2,
  (value1, value2, props) => value1 + value2 + props.value3
);

// A selector created with createSelector ignores the props for memoization
// See note at bottom of `defaultMemoizeFunc`
let called = 0;
const selector = createSelector(
  state => state.a,
  (a, props) => {
    called++;
    return a + b
  }
);
assert.equal(selector({a: 1}, 100), 101);
assert.equal(selector({a: 1}, 200), 101);
assert.equal(called, 1);
assert.equal(selector({a: 2}, 200), 202);
assert.equal(called, 2);
```

### defaultMemoizeFunc(func, valueEquals = defaultValueEquals)

`defaultMemoizeFunc` has a cache size of 1. This means it always recalculates when an argument changes, as it only stores the result for immediately preceding value of the argument.

`defaultMemoizeFunc` determines if an argument has changed by calling the valueEquals function. The default `valueEquals` function checks for changes using reference equality:

```js
function defaultValueEquals(currentVal, previousVal) {
  return currentVal === previousVal;
}
```

```js
  let called = 0;
  const memoized = defaultMemoize(state => {
    called++;
    return state.a;
  });
  const o1 = {a: 1};
  const o2 = {a: 2};
  assert.equal(memoized(o1, {}), 1);
  assert.equal(memoized(o1, {}), 1);
  assert.equal(called, 1);
  assert.equal(memoized(o2, {}), 2);
  assert.equal(called, 2);
```

`defaultMemoizeFunc` does not look at the props argument when determining if the arguments have changed. This is because by default `defaultMemoize` uses `defaultValueEquals`, and as`defaultValueEquals` is a reference equality check and the reference of the props will be different each render the memoization will not work. 

(NOTE: I am not happy about this, I think this is going to surprise people. I am considering adding an option that specifies whether the props should be used for memoization as part of the configuration options for people who aren't using the reference equality of `defaultValueEquals`. To make things worse, when using a third party memoize like `lodash.memoize` with `createSelectorCreator`, the memoize function **will** take the props into account for memoization. I think it is likely that if you are using a third party memoize your hashing function will be doing some kind of deepEquals so the selector will only update when there is actually a new value somewhere in the props. However, this could still be a problem if the props are changing in a manner such that they resolve to a new never-before-seen hash on every render. Ideas are welcome here.)

### createSelectorCreator(memoizeFunc, ...memoizeOptions)

Return a selectorCreator that creates selectors with a non-default memoizeFunc. 

`...memoizeOptions` is a variadic number of configuration options that will be passsed to `memoizeFunc` inside `createSelectorSelector`:

```js

let memoizedResultFunc = memoizeFunc(resultFunc, ...memoizeOptions);

```

You can use createSelectorCreator to customize the `valueEquals` function for `defaultMemoizeFunc` like this:

```js
import { createSelectorCreator, defaultMemoizeFunc } from 'reselect';
import Immutable from 'immutable';

// create a "selector creator" that uses Immutable.is instead of ===
// Note that this is not usually necessary when using Immutable.js with reselect
const immutableCreateSelector = createSelectorCreator(
  defaultMemoizeFunc,
  Immutable.is
);

// use the new "selector creator" to create a 
// selector (state.values is an Immutable.List)
const mySelector = immutableCreateSelector(
  [state => state.values.filter(val => val < 5)],
  values => values.reduce((acc, val) => acc + val, 0)
);
```

Using the memoize function from lodash for an unbounded cache:

```js
import { createSelectorCreator } from 'reselect';
import memoize from 'lodash.memoize';


let called = 0;
const customSelectorCreator = createSelectorCreator(memoize, JSON.stringify);
const selector = customSelectorCreator(
  state => state.a,
  state => state.b,
  (a, b) => {
    called++;
    return a + b;
  }
);
assert.equal(selector({a: 1, b: 2}), 3);
assert.equal(selector({a: 1, b: 2}), 3);
assert.equal(called, 1);
assert.equal(selector({a: 2, b: 3}), 5);
assert.equal(called, 2);
```
