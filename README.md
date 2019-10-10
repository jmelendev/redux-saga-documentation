# Redux Saga and Generator Functions

Redux Saga is a Redux middleware that keeps dispatched actions pure and utilizes generator functions to neatly and efficiently handle asynchronous code.

## Generators and Iterators?

Iterators are objects that define a sequence of events that typically return a value when completed. An iterator will follow the [iterator protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterator_protocol). They are objects that use a `next()` method. This method doesn't take any arguments and returns at least a "done" property which will be a boolean and a value. 

* Note: Values can be omitted when done = `true`

When a sequence has been completed, it will return the done property along with any associated value. Once an iterator is created, the iteration will be triggered by calling the `next()` method. This will run until a value is yielded. Any subsequent `next()` calls will continue to return `{ done: true }`.

### Example Iterator

Below is an example of an range iterator. It takes a start, end, and step argument to be used when itertating through. 

```
function makeRangeIterator(start = 0, end = Infinity, step = 1) {
    let nextIndex = start;
    let iterationCount = 0;

    const rangeIterator = {
       next() {
           let result;
           if (nextIndex < end) {
               result = { value: nextIndex, done: false }
               nextIndex += step;
               iterationCount++;
               return result;
           }
           return { value: iterationCount, done: true }
       }
    };
    return rangeIterator;
}

```

This function has an iterator object with a `next()` property that returns an object in this syntax `{ value: ..., done: ... }`. This will continue to return a value with a falsey done property until `nextIndex` > `end`. At that point the iterator is terminated and returned object has would look like this `{value: ..., done: true }`. Once `done` equals true then you know the iterator has completed. All subsequent calls on to `next()` will return with `done: true`.

Here is what is would look like to use `makeRangeIterator()`.

```
let it = makeRangeIterator(1, 10, 2);

let result = it.next();
while (!result.done) {
  console.log(result.value); // 1 3 5 7 9
  result = it.next();
}
```

Above you will see that by calling `it.next()` the function will continue to run in the `while()` statement until the iterator returns `{..., done: true }`.

* Note: There isn't a way to tell if a particular function is an iterator without using [Iterables](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators#Iterables).

### How Do Generators Work?

Generator functions, written as `function* ()` are functions that make writing iterative functions in a single function. Generators don't execute the code when initially called. They will execute an iterator that is called a "generator". The the `next()` is called, the function will run until it reaches the `yield` keyword. Generators can be called repeatedly, however, they will only iterate once.

When using a generator function, the above iterator example can be simplied like so:

```
function* makeRangeIterator(start = 0, end = 100, step = 1) {
  for (let i = start; i < end; i += step) {
    yield i;
  }
}
```

In this case you will see that instead of running through the entire for loop, the generator is suspended until the next method is called upon it. For example:

```
let it = makeRangeIterator(1, 10, 2);

console.log(it.next())

```

Will return `{ done: false, value: 1 }`. The yield keyword keeps the loop from continuing until the `next()` method is called.

### Features of Generators

Since generator functions run until a `yield` is called, there is more control over how far along to allow a loop to run. There are a multitude of benefits that redux saga utilizes given this aspect of generator functions. You can use the `next()` method to step through loops as long as a yield statement is used within the loop. Evaluating in this manner allows for exiting the loop when certain conditions are met or exiting the loop if another instance is called before the previous was finished. This is a huge benefit in sagas.

### Example Usage

Given the following generator function that return an id in increments of 1, you can see the test case below.

```
function* idMaker() {
  var index = 0;
  while (index < index+1)
    yield index++;
}

var gen = idMaker();

console.log(gen.next().value); // 0
console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
console.log(gen.next().value); // 3
```

### Example Test

Since generator functions run until each yield, you can test each step of the function by called the `next()` and asserting on that value object returned in the following syntax `{ done: false, value: 1 }`.

```
describe('idMaker test', () => {
  it('should return 0', () => {
      const generator = idMaker();

      expect(generator.next().value).toBe(0));
  });
  
    it('should return 2', () => {
      const generator = idMaker();
      generator.next();
      generator.next();
      expect(generator.next().value).toBe(2));
  });
});
```

## What is Redux Saga?

Redux Saga is a redux middle-ware that utilizes generator functions to act as a side effect library similar to thunks. Generators allow for an asynchronous syntax similar to async await.

### How Does Redux Saga Work?

By utilizing generator functions, the Redux Saga api is able to create a multitude of methods to hanlde side effects in your react application. The api consists of effect creators, effect combinators, interfaces, and external api and other utility functions.

Effect creators are the bulk of the api and there are a wide range of them to help accomplish the task that your app needs to achieve. The api is extensive but here are some of the main ones that can be used.

* `take` - waits for a `REDUX_ACTION` to be dispatched. All code below this will be suspended until the pattern has been met. If another action is dispatched, the previous generator will run until completion and then the second one will run.

* `takeEvery` - waits for a `REDUX_ACTION` to be dispatched and each dispatch will have a separate saga called. This method takes the `PATTERN`, another saga function to run, and any arguments for the mentioned saga to be called with. This allows for concurrent actions to be handled at the same time. These calls are not gauranteed to terminate at the same rate that they were called in.

* `put` - can dispatch an action to the store directly or call an action creator function. This would look like `put({ type: 'SOME_ACTION', payload: true)`

Checkout the full api list [Here](https://redux-saga.js.org/docs/api) 

### Sagas vs Thunks?

There are two major benefits, aside from the complex scenarios that the sagas api allow you to accomplish, of using sagas over thunks. Thunks dispatch a function unlike Sagas that dispatch actions, or objects. With this being said, sagas allow you to:

1. Keep actions as pure objects
2. Subscribe directly to the store.

This means that sagas can run as a separate thread in your application by taking and dispatching actions. Thunks have to be called from within the component in order to dispatched to interact with the store.

### Example Usage

This is a very basic example.

```
import { take, put } from `redux-saga/effects`
import getUser from 'api'

function* fetchUser(action) {
  yield take('FETCH_USER');
  const data = yield getUser(); 
  
  if (data) {
    put({type: 'SAVE_USER', payload: data})
  }
}

export default fetchUser
```

### Example Test

```
import fetchUser from 'sagas'

describe('fetchUser saga', () => {
    it('fetches user', () => {
        const gen = fetchUser;
        
        gen.next()
        
        //assert on gen.next() until complete
    });
});
```

For more information on Redux Sagas check the full documentation here. [Redux Saga Documentation](https://redux-saga.js.org/)
