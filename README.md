# RefluxJS

A simple library for unidirectional dataflow architecture inspired by ReactJS [Flux](http://facebook.github.io/react/blog/2014/05/06/flux.html).

[![NPM Version][npm-image]][npm-url]
[![NPM Downloads][downloads-image]][npm-url]
[![Bower Version][bower-image]][bower-url]
[![Dependencies][dependencies-image]][npm-url]
[![Build Status][travis-image]][travis-url]
[![Gratipay][gratipay-image]][gratipay-url]

You can read an overview of Flux [here](http://facebook.github.io/react/docs/flux-overview.html), however the gist of it is to introduce a more functional programming style architecture by eschewing MVC like pattern and adopting a single data flow pattern.

```
╔═════════╗       ╔════════╗       ╔═════════════════╗
║ Actions ║──────>║ Stores ║──────>║ View Components ║
╚═════════╝       ╚════════╝       ╚═════════════════╝
     ^                                      │
     └──────────────────────────────────────┘

```

The pattern is composed of actions and data stores, where actions initiate new data to pass through data stores before coming back to the view components again. If a view component has an event that needs to make a change in the application's data stores, they need to do so by signalling to the stores through the actions available.

## Comparing RefluxJS with Facebook Flux

The goal of the refluxjs project is to get this architecture easily up and running in your web application, both client-side or server-side. There are some differences between how this project works and how Facebook's proposed Flux architecture works:

You can read more in this [blog post about React Flux vs Reflux](http://spoike.ghost.io/deconstructing-reactjss-flux/).

### Similarities with Flux

Some concepts are still in Reflux in comparison with Flux:

* There are actions
* There are data stores
* The data flow is unidirectional

### Differences with Flux

Reflux has refactored Flux to be a bit more dynamic and be more FRP friendly:

* The singleton dispatcher is removed in favor for letting every action act as dispatcher instead.
* Because actions are listenable, the stores may listen to them. Stores don't need to have a big switch statements that does static type checking (of action types) with strings
* Stores may listen to other stores, i.e. it is possible to create stores that can *aggregate data further*, similar to a map/reduce.
* `waitFor` is replaced in favor to handle *serial* and *parallel* data flows:
 * **Aggregate data stores** (mentioned above) may listen to other stores in *serial*
 * **Joins** for joining listeners in *parallel*
* *Action creators* are not needed because RefluxJS actions are functions that will pass on the payload they receive to anyone listening to them

## Examples

You can find some example projects at these locations:

* [Todo Example Project](https://github.com/spoike/refluxjs-todo) - [http://spoike.github.io/refluxjs-todo/](http://spoike.github.io/refluxjs-todo/)

## Installation

You can currently install the package as a npm package or bower.

### NPM

The following command installs reflux as an npm package:

    npm install reflux

### Bower

The following command installs reflux as a bower component that can be used in the browser:

    bower install reflux

## Usage

For a full example check the [`test/index.js`](test/index.js) file.

### Creating actions

Create an action by calling `Reflux.createAction` with an optional options object.

```javascript
var statusUpdate = Reflux.createAction(options);
```

An action is a functor that can be invoked like any function.

```javascript
statusUpdate(data); // Invokes the action statusUpdate
statusUpdate.triggerAsync(data); // same effect as above
```

If `options.sync` is true, the functor will instead call `action.trigger` which is a synchronous operation. You can change `action.sync` during the lifetime of the action, and the following calls will honour that change.

There is also a convenience function for creating multiple actions.

```javascript
var Actions = Reflux.createActions([
    "statusUpdate",
    "statusEdited",
    "statusAdded"
  ]);

// Actions object now contains the actions
// with the names given in the array above
// that may be invoked as usual

Actions.statusUpdate();
```

#### Action hooks

There are a couple of hooks avaiable for each action.

* `preEmit` - Is called before the action emits an event. It receives the arguments from the action invocation. If it returns something other than undefined, that will be used as arguments for `shouldEmit` and subsequent emission.

* `shouldEmit` - Is called after `preEmit` and before the action emits an event. By default it returns `true` which will let the action emit the event. You may override this if you need to check the arguments that the action receives and see if it needs to emit the event.

Example usage:

```javascript
Actions.statusUpdate.preEmit = function() { console.log(arguments); };
Actions.statusUpdate.shouldEmit = function(value) {
    return value > 0;
};

Actions.statusUpdate(0);
Actions.statusUpdate(1);
// Should output: 1
```

You can also set the hooks by sending them in a definition object as you create the action:

```javascript
var action = Reflux.createAction({
    preEmit: function(){...},
    shouldEmit: function(){...}
});
```

### Creating data stores

Create a data store much like ReactJS's own `React.createClass` by passing a definition object to `Reflux.createStore`. You may set up all action listeners in the `init` function and register them by calling the store's own `listenTo` function.

```javascript
// Creates a DataStore
var statusStore = Reflux.createStore({

    // Initial setup
    init: function() {

        // Register statusUpdate action
        this.listenTo(statusUpdate, this.output);
    },

    // Callback
    output: function(flag) {
        var status = flag ? 'ONLINE' : 'OFFLINE';

        // Pass on to listeners
        this.trigger(status);
    }

});
```

In the above example, whenever the action is called, the store's `output` callback will be called with whatever parameters was sent in the action. E.g. if the action is called as `statusUpdate(true)` then the flag argument in `output` function is `true`.

A data store is a publisher much like the actions, so they too have the `preEmit` and `shouldEmit` hooks.

#### Listening to many actions at once

Since it is a very common pattern to listen to all actions from a `createActions` call in a store `init` call, the store has a `listenToMany` function that takes an object of listenables. Instead of doing this:

```javascript
var actions = Reflux.createActions(["fireBall","magicMissile"]);

var Store = Reflux.createStore({
    init: function() {
        this.listenTo(actions.fireBall,this.onFireBall);
        this.listenTo(actions.magicMissile,this.onMagicMissile);
    },
    onFireBall: function(){
        // whoooosh!
    },
    onMagicMissile: function(){
        // bzzzzapp!
    }
});
```

...you can do this:

```javascript
var actions = Reflux.createActions(["fireBall","magicMissile"]);

var Store = Reflux.createStore({
    init: function() {
        this.listenToMany(actions);
    },
    onFireBall: function(){
        // whoooosh!
    },
    onMagicMissile: function(){
        // bzzzzapp!
    }
});
```

This will add listeners to all actions `actionName` who have a corresponding `onActionName` (or `actionName` if you prefer) method in the store. Thus if the `actions` object should also have included an `iceShard` spell, that would simply be ignored.

#### The listenables shorthand

To make things more convenient still, if you give an object of actions to the `listenables` property of the store definition, that will be automatically passed to `listenToMany`. So the above example can be simplified even further:

```javascript
var actions = Reflux.createActions(["fireBall","magicMissile"]);

var Store = Reflux.createStore({
    listenables: actions,
    onFireBall: function(){
        // whoooosh!
    },
    onMagicMissile: function(){
        // bzzzzapp!
    }
});
```

The `listenables` property can also be an array of such objects, in which case all of them will be sent to `listenToMany`. This allows you to do convenient things like this:

```javascript
var Store = Reflux.createStore({
    listenables: [require('./darkspells'),require('./lightspells'),{healthChange:require('./healthstore')}],
    // rest redacted
});
```

### Listening to changes in data store

In your component, register to listen to changes in your data store like this:

```javascript
// Fairly simple view component that outputs to console
function ConsoleComponent() {

    // Registers a console logging callback to the statusStore updates
    statusStore.listen(function(status) {
        console.log('status: ', status);
    });
};

var consoleComponent = new ConsoleComponent();
```

Invoke actions as if they were functions:

```javascript
statusUpdate(true);
statusUpdate(false);
```

With the setup above this will output the following in the console:

```
status:  ONLINE
status:  OFFLINE
```
### React component example

Register your component to listen for changes in your data stores, preferably in the `componentDidMount` [lifecycle method](http://facebook.github.io/react/docs/component-specs.html) and unregister in the `componentWillUnmount`, like this:

```javascript
var Status = React.createClass({
    initialize: function() { },
    onStatusChange: function(status) {
        this.setState({
            currentStatus: status
        });
    },
    componentDidMount: function() {
        this.unsubscribe = statusStore.listen(this.onStatusChange);
    },
    componentWillUnmount: function() {
        this.unsubscribe();
    },
    render: function() {
        // render specifics
    }
});
```

#### Convenience mixin for React

You always need to unsubscribe components from observed actions and stores upon
unmounting. To simplify this process you can use [mixins in React](http://facebook.github.io/react/docs/reusable-components.html#mixins). There is a convenience mixin available at `Reflux.ListenerMixin`. Using that, the above example can be written like thus:

```javascript
var Status = React.createClass({
    mixins: [Reflux.ListenerMixin],
    onStatusChange: function(status) {
        this.setState({
            currentStatus: status
        });
    },
    componentDidMount: function() {
        this.listenTo(statusStore, this.onStatusChange);
    },
    render: function() {
        // render specifics
    }
});
```

The mixin provides the `listenTo` method for the React component, that works much like the one found in the Reflux's stores, and handles the listeners during mount and unmount for you. You also get the same `listenToMany` method as the store has.


#### Using Reflux.listenTo

If you're not reliant on any special logic for the `this.listenTo` calls inside `componentDidMount`, you can instead use a call to `Reflux.listenTo` as a mixin. That will automatically set up the `componentDidMount` and the rest for you, as well as add the `ListenerMixin` functionality. With this our example above can be reduced even further:

```javascript
var Status = React.createClass({
    mixins: [Reflux.listenTo(statusStore,"onStatusChange")],
    onStatusChange: function(status) {
        this.setState({
            currentStatus: status
        });
    },
    render: function() {
        // render using `this.state.currentStatus`
    }
});
```

You can have multiple calls to `Reflux.listenTo` in the same `mixins` array.

There is also `Reflux.listenToMany` which works in exactly the same way, exposing `listener.listenToMany`.

#### Using Reflux.connect

If all you want to do is update the state of your component to whatever the data store transmits, you can use `Reflux.connect(listener,[stateKey])` as a mixin. If you supply a `stateKey` the state will be updated through `this.setState({<stateKey>:data})`, otherwise `this.setState(data)`. Here's the example above changed to use this syntax:

```javascript
var Status = React.createClass({
    mixins: [Reflux.connect(statusStore,"currentStatus")],
    render: function() {
        // render using `this.state.currentStatus`
    }
});
```

### Listening to changes in other data stores (aggregate data stores)

A store may listen to another store's change, making it possible to safely chain stores for aggregated data without affecting other parts of the application. A store may listen to other stores using the same `listenTo` function as with actions:

```javascript
// Creates a DataStore that listens to statusStore
var statusHistoryStore = Reflux.createStore({
    init: function() {

        // Register statusStore's changes
        this.listenTo(statusStore, this.output);

        this.history = [];
    },

    // Callback
    output: function(statusString) {
        this.history.push({
            date: new Date(),
            status: statusString
        });
        // Pass the data on to listeners
        this.trigger(this.history);
    }

});
```
## Advanced usage

### Switching EventEmitter

Don't like to use the EventEmitter provided? You can switch to another one, such as NodeJS's own like this:

```javascript
// Do this before creating actions or stores

Reflux.setEventEmitter(require('events').EventEmitter);
```

### Switching nextTick

Whenever action functors are called, they return immediately through the use of `setTimeout` (`nextTick` function) internally.

You may switch out for your favorite `setTimeout`, `nextTick`, `setImmediate`, et al implementation:

```javascript

// node.js env
Reflux.nextTick(process.nextTick);
```

For better alternative to `setTimeout`, you may opt to use the [`setImmediate` polyfill](https://github.com/YuzuJS/setImmediate).


### Joining parallel listeners with composed listenables

The Reflux API contains `join` methods that makes it easy to aggregate publishers that emit events in parallel. This corresponds to the `waitFor` method in Flux.

#### Argument tracking

A join is triggered once all participating publishers have emitted at least once. The callback will be called with the data from the various emissions, in the same order as the publishers were listed when the join was created.

There are four join methods, each representing a different strategy to track the emission data:

*    `joinLeading`: Only the first emission from each publisher is saved. Subsequent emissions by the same publisher before all others are finished are ignored.
*    `joinTrailing`: If a publisher triggers twice, the second emission overwrites the first.
*    `joinConcat`: An array of emission arguments are stored for each publisher.
*    `joinStrict`: An error is thrown if a publisher emits twice before the join is completed.

The method signatures all look like this:

```javascript
joinXyz(...publisher,callback)
```

Once a join is triggered it will reset, and thus it can trigger again when all publishers have emitted anew.

#### Using the listener instance methods

All objects using the listener API (stores, React components using `ListenerMixin`, or other components using the `ListenerMethods`) gain access to the four join instance methods, named after the argument strategy. Here's an example saving the last emission from each publisher:

```javascript
var gainHeroBadgeStore = Reflux.createStore({
    init: function() {
        this.joinTrailing(actions.disarmBomb, actions.saveHostage, actions.recoverData, this.triggerAsync);
    }
});

actions.disarmBomb("warehouse");
actions.recoverData("seedyletter");
actions.disarmBomb("docks");
actions.saveHostage("offices",3);
// `gainHeroBadgeStore` will now asyncronously trigger `[["docks"],["offices",3],["seedyletter"]]`.
```

#### Using the static methods

Since it is rather common to have a store where the only purpose is to listen to a join and trigger when the join is completed, the join methods have static counterparts on the `Reflux` object which return stores listening to the requested join. Using them, the store in the example above could instead be created like this:

```javascript
var gainHeroBadgeStore = Reflux.joinTrailing(actions.disarmBomb, actions.saveHostage, actions.recoverData);
```

### Sending default data with the listenTo function

The `listenTo` function provided by the `Store` and the `ListenerMixin` has a third parameter that accepts a callback. This callback will be invoked when the listener is registered with whatever the `getInitialState` is returning.

```javascript
var exampleStore = Reflux.createStore({
    init: function() {},
    getInitialState: function() {
        return "the initial data";
    }
});

// Anything that will listen to the example store
this.listenTo(exampleStore, onChangeCallback, initialCallback)

// initialCallback will be invoked immediately with "the initial data" as first argument
```

Remember the `listenToMany` method? In case you use that with other stores, it supports `getInitialState`. That data is sent to the normal listening callback, or a `this.on<Listenablename>Default` method if that exists.

## Colophon

[List of contributors](https://github.com/spoike/reflux/graphs/contributors) is available on Github.

This project is licensed under [BSD 3-Clause License](http://opensource.org/licenses/BSD-3-Clause). Copyright (c) 2014, Mikael Brassman.

For more information about the license for this particular project [read the LICENSE.md file](LICENSE.md).

This project uses [eventemitter3](https://github.com/3rd-Eden/EventEmitter3), is currently MIT licensed and [has it's license information here](https://github.com/3rd-Eden/EventEmitter3/blob/master/LICENSE).

[npm-image]: http://img.shields.io/npm/v/reflux.svg
[downloads-image]: http://img.shields.io/npm/dm/reflux.svg
[dependencies-image]: http://img.shields.io/david/spoike/refluxjs.svg
[npm-url]: https://www.npmjs.org/package/reflux
[bower-image]: http://img.shields.io/bower/v/reflux.svg
[bower-url]: http://bower.io/search/?q=reflux
[travis-image]: http://img.shields.io/travis/spoike/refluxjs/master.svg
[travis-url]: https://travis-ci.org/spoike/refluxjs
[gratipay-image]: http://img.shields.io/gratipay/spoike.svg
[gratipay-url]: https://gratipay.com/spoike/
