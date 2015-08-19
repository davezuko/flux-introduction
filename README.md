Data Flow / Architecture
========================

Table of Contents
-----------------
1. [Introduction](#introduction)
1. [Unidirectional Data Flow](#unidirectional-data-flow)
1. [Component Responsibility](#component-responsibility)
  - Dumb Components
  - Smart Components
1. [Flux](#flux)
  - Stores
  - Actions
  - Dispatcher
  - Immutability
  - Benefits
  - Async
  - Flavors
1. [Summary](#summary)

Introduction
------------
#### Recommended Viewing
- [Full-Stack Flux](https://www.youtube.com/watch?v=KtmjkCuV-EU) - Pete Hunt from Instagram/Facebook discusses what problems Flux is designed to solve, and how centralizing the source of truth in your application can naturally minimize complexity.
- [Redux](https://www.youtube.com/watch?v=xsSnOQynTHs) - Redux, a not-so-Flux-like Flux implementation. Video describes what inspired it and how it relates to Flux, and the benefits that result from its divergence. Helps solidify general Flux knowledge through comparisons and examples.
- [React.js Conf 2015](http://facebook.github.io/react/blog/2015/02/18/react-conf-roundup-2015.html) - Round-up of all the talks from this awesome conference.

#### Recommended Reading
- [Thinking in React](http://facebook.github.io/react/docs/thinking-in-react.html)
- [Flux from an Angular perspective](http://blog.celerity.com/react/flux-from-an-angularjs-perspective)
- [[Official] Thinking in React](http://facebook.github.io/react/docs/thinking-in-react.html)
- [Getting to Know Flux](https://scotch.io/tutorials/getting-to-know-flux-the-react-js-architecture)
- [Introduction to React and Flux](http://blog.mimacom.com/introduction-to-react-and-flux/)
- [Simpler UI Reasoning with Unidirectional Data Flow](http://omniscientjs.github.io/guides/01-simpler-ui-reasoning-with-unidirectional/)

One of the biggest contributors to complexity in an application is statefulness; that is, mutations accrued over time. An application may load with an initial state, but as components accumulate their own states - tracking flags, executing ajax calls, mutating data - soon that simple representation has snowballed into a tangle of changes that become difficult to understand let alone remember. And these aren't just mutations at one point in the hierarchy, they could be at any level of the codebase.

This sort of statefulness can lead to bigger issues such as race conditions or conflicting/out-of-sync data. A good example of this is the original Facebook chat application, described [here by Pete Hunt](https://www.youtube.com/watch?v=KtmjkCuV-EU#t=3m). Ultimately, complexity is exponentially proportional to the factor of time in an application - the more time, especially _changes over time_, plays a role in your application the harder it is to know what combination of actions produce a certain state.

The solution is even simpler than reducing how many mutations are made or writing better code. It's centralizing the state that represents your application (the prevailing concept in Flux) and passing that state down _into_ components. If components can simply _react_ to changes, it suddenly becomes much easier to figure out what they're supposed to do or how they should rebder, and you can begin asserting things about them based on what properties they receive. In essence, you remove time from the equation.

Unidirectional Data Flow
-----------------------

While it may be self-evident, unidirectional data flow, formally stated, describes a one-way architecture, specifically one that is top-down. In the real world, this means that data enters your application at the highest possible level and trickles down. As an example, if a component's responsibility is to display data in a certain format, it doesn't need to be concerned about where that data comes from - it simply comes "down" into the component via a property. Conversely, a view might need to make a request for a specific resource, in which case it will be the one responsible for connecting to a store and providing the necessary properties to its children.

The influences of this architecture can be seen all the way down at the component level. Because data flows from top to bottom, components become _reactive_, and therefore declarative, as they only need to describe how to render given a certain input. Approached functionally, a component can therefore be thought as:

`f(x) = ...`

where the component is a function of its input, `x` (in the case of React, `x` is the set of properties that a component receives, and the result of the function is how that component looks when it renders). This means that you can begin thinking about your components as representations of slices in time.

Think back to middle school for a minute. Remember learning that in order for `f(x)` to be a function, there can be only one `y` value for every `x`. If you pass a two values to a function, e.g.:

```js
function add (a, b) {
  return a + b;
}
```

you can reasonably expect get the same output for every input, anything else and what's the point? If you extrapolate this to the application level, you should be able to pass in some state (properties) and obtain the same result every time. This is far easier said than done, and as a result it's common to introduce side effects and impure functions, ultimately complicating applications to the point that they become non-deterministic and difficult to reason about. However, if you succeed, your application becomes a simple function composed of declarative components, allowing you to easily reproduce any state just by providing the correct input.

On the component level, by adhering to pure functions (in React all render methods should be pure) components become [referentially transparent](http://stackoverflow.com/questions/210835/what-is-referential-transparency). This makes testing incredibly easy because, just like your application, you can just pass a component a set of properties and analyze the result; no need to track what mutations occur over time (not to mention the myriad of possible combinations and timings). This is just part of why functional reactive programming is _awesome_.

Compare this to what a `directive` and its `controller` in Angular might be responsible for: influencing and enacting changes over time. This means that how a component looks when it first renders might not be how it looks in 2 minutes, even though the properties it was given haven't changed! How do you reliably test this? it's incredibly difficult because these types of components are _impure_ and _can have different outputs for the same input_. You have to remember everything that goes on inside the component, rather than having it simply react to changes that come from the outside. The imperative approach is far too common, primarily because it's intuitive, and quickly leads to unneeded complexity. Thankfully, React makes doing this very difficult.

Following the reactive paradigm encouraged by React, you end up almost completely eliminating an entire dimension (time) from the equation. This is called _reactive programming_, and it fits incredibly well with unidirectional, data flow where state changes occur at the top level and propagate down, allowing components to automatically re-render to reflect those changes. This can't be overstated enough: components are merely blueprints for describing how a given state should render.

So how do we implement this? And what about the data? Where is it stored and how does it get changed? These are great questions, but they are implementation details secondary to the overarching design. The section on [Flux](#flux) will delve further into answering these questions.

Component Responsibility
------------------------

**Recommended Reading**
  - [Smart and Dumb Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0) - Dan Abramov, the creator of Redux and React-hot-loader, discusses component design patterns that have evolved over time.

Components should touch as little surface area as possible, especially where data is concerned. Doing so eliminates complexity and makes it far easier to test and reuse components since there are few moving parts. For example, a `TodoList` component does not need to know where the list of todos comes from, or what to do when the user wants to delete one. Such a component would be classified as a `dumb component`, which is covered in more detail below.

This results in two primary types of components: `Smart` and `Dumb` components. `Dumb` components will make up the majority of your codebase, and are able to be reused pretty much anywhere, since they don't control how data is obtained or manipulated. `Smart` components, on the other hand, are the "top level" of an application. They are responsible for distributing data down into their constituent `dumb` components, who simply do the bidding of whoever implements them.

#### Dumb Components

This type of component is the building block of an application. It is what takes state (in the form of properties) and transforms it into something meaningful for the user. A TodoList takes a collection of Todos and displays it in a friendly list format, possibly as smaller Todo components.

In order for a `Dumb` component to be meaningful, it needs to expose an API (again, in the form of properties) that implementors adhere must to. This can range from simple properties that provide data, to functions that the component can invoke in response to different events. With the TodoList component, the TodoList doesn't need to know how to remove todos or flag them as complete. Maybe you have 5 different types of TodoLists in your application that all use different sets of data from totally separate sources - maybe some of them are even tied directly to the server. Because of this, it doesn't make sense for the TodoList to make assumptions about how to operate on the data its given. Instead, it might invoke handlers such as "deleteTodo" or "toggleTodoCompletion", which its parent provides and can hook into. New data for a TodoList flows _into_ the TodoList - top-down, unidirectional data flow!

This may seem like a lot of work, and it does take more effort than simply allowing components to perform direct mutations, but you are rewarded with components that are entirely isolated from the ecosystems they live in, and your single source of truth remains safely intact.

#### Smart Components


What about views? I had this exact question, which the author of Redux was [kind enough to clarify](https://github.com/rackt/redux/issues/562#event-384662248).


Flux
----

So unidirectional data flow and smart/dumb components sound like a great idea, but how do they fit together? What kind of architecture surrounds them? Enter Flux.

Flux boils down to a pretty basic concept: data flows in one direction through your application, with data and the actions that change it being centralized in stores. Components react to changes in stores, but it's important to note that they react based on the new state that's produced, not the events themselves.

If you take this description further, it can be extrapolated to state that your core application state should be able to be represented by a single "input" at any point in time. And, similar to components, an application is just one big function that encompasses a bunch of smaller functions. You take some state representation, run it through your application, and out pops a result!

### Stores

Stores are, as the name implies, where data is stored. It's also where any changes to that data take place, which keeps everything centralized. Stores were designed to solve the problem where multiple components rely on the same data, but potentially have inaccurate or out-of-date representations of that data. When data can be transformed in more than one spot, you open yourself up to race conditions or other nasty bugs, and you lose that "single source of truth". Stores keep all the business logic right next to the actual data, which means that as long as all components draw from that store (which they should), they will all remain in sync with each other.

Importantly, stores are _entirely synchronous_. They receive actions and perform some internal operation in response to those actions, that's it. This means that asynchronous events simply dispatch actions upon success/error, this way the store doesn't have to know about, or keep track of outstanding requests. Once you start storing state within a store, it becomes impure - it could look the same on the outside, but whatever's going on internally could influence its output.

### Actions

So you have a centralized store, awesome, but how do you change what's inside of it? After all, that's the whole point of an application, _doing_ something. This is what actions (and the dispatcher, as you'll see) are for. An action, in its simplest form, is an object describing the type of action (normally as a contant) and any data relevant to the action. It might look something like this:

```js
import { TODO_DELETE } from 'constants/todo';

const deleteTodoAction = {
  type : TODO_DELETE,
  payload : {
    id : 5
  }
};
```

All actions will flow through a store, whether it cares to do anything with them or not. It decides what actions to use based on their `type`, so a `Todo` store might want to know about all todo-related actions, e.g. `TODO_DELETE`, `TODO_CREATE`, `TODO_TOGGLE_COMPLETE`. These actions are a way to communicate with top-level stores from anywhere within the component hierarchy. Actions, which again are just simple objects, can be cumbersome to create, so there are normally _action creators_ to help with this. Here's one that helps with that `TODO_DELETE` action from above.

```js
import { TODO_DELETE } from 'constants/todo';

export function deleteTodo (id) {
  return {
    type : TODO_DELETE,
    payload : {
      id // es6, remember!
    }
  };
}
```

Notice how this abstracts away all of the actual creating and formatting of the action object, inserting the data that's passing in as arguments into the payload. Now that we know how to create actions, let's see how a store might handle one:

```js
Dispatcher.register(function (action) {
  const { type, payload } = action;

  switch (type) {
    case TODO_DELETE:
      this._todos = this._todos.filter(todo => todo.id !== payload.id);
      this.emitChange();
      break;
  }
}.bind(this));
```

Once the todos have been updated with the deletion, the change event will notify whichever component is listening to the store. That component will then, likely, re-render based on the new todos list, which allows all child components to automatically react to this change, without having to know what changed or how. Now, you may be wondering, how does the store know about this action? And here in lies a critical point.

In the original Flux implementation, action creators (that `deleteTodo` function from earlier), would automatically call the dispatcher. So instead of just return a plain object, it would look something like this:

```js
import { TODO_DELETE } from 'constants/todo';
import AppDispatcher from 'dispatchers/app';

export function deleteTodo (id) {
  AppDispatcher.dispatch({
    type : TODO_DELETE,
    payload : {
      id // es6, remember!
    }
  });
}
```

Many flavors of Flux have moved away from this, since it couples your action creators directly to a specific dispatcher. By just returning plain action objects, they can be dispatched however you want.

### Dispatcher

The dispatcher is essentially a central routing system that forwards actions on to stores. But wait, why don't the actions just communicate directly with the store? Wouldn't that be easier? Great question, lad (or ladette)!

  1. That would couple the action way too tightly to the store. And what happens when you want to communicate with multiple stores? By keeping them separated, you can dispatch actions freely without having to be aware of the other half of the implementation.
  2. Using a dispatcher (in certain Flux implementations) allows stores to declare dependencies on other stores, so actions can be handled in a specific order.
  3. Funneling all actions through a single point allows for central logging, debugging, and centralizes events that affect application state. Without a central system, there's no easy way to reliably track these actions.

### Immutability

Let's approach this from the perspective of the the canonical Todo application. Think of how it's been written in the past, where your list of todos might start off as an empty array that gets pushed/popped/filtered over time.

```js
const todos = [];

// let's add a new Todo item
todos.push(new Todo());
```

What could be wrong with an example so simple? Well, for one, think about what would happen if you were to provide this list of todos to other components. How would they know something changed? There'd have be a deep equality check on the object, since the object reference is the same. Additionally, how would the application know _when_ to do this check? Angular attempts to solve this exact problem with its dirty checking and digest cycle, and it's not exactly a clean solution. Data could change anywhere at any point in time. Ever wonder why Angular 2.0 differs so wildly from its predecessor?

Anyways, back to the example. The important thing to notice is that we're mutating `todos`, not producing a new brand new state. As a result, we've lost the ability to represent the old state and new state as separate entities at the same time, because the old one no longer exists. In order to traverse back through time, you'd have to remember which mutations occurred and when. You'd have to reverse actions, `popping` instead of `pushing`. Wouldn't it be easier to just represent distinct states as, well, distinct states (separate objects)? Enter immutability.

...

### Benefits

#### Hot Reloading
Pure functions are what allow react-hot-loader to work. If a component's render function was non-deterministic (impure), the entire component would need to be reloaded after a change. However, since that's not the case, the previous state and properties can continue to exist as they were while the component's methods/render function are simply be swapped out for their new versions.

#### Centralization
The funneling of actions through a central dispatcher means that it's easy to add middleware between your actions and the stores. This opens up the possibility to implement features such as centralized logging and debugging.

#### Time Travel
If you use immutable data structures, you now have the _completely free_ ability to completely step through time by replaying actions or stepping through previous states.

### Async

So we've now talked about how the dispatcher, actions, and stores all fit together. The discussion on unidirectional data flow discussed how awesome it is to be able to eliminate time from the equation, or at least limit its effect on the application. But we're well past the AJAX revolution, and webapps need to be able to work with asynchronous events. How does that fit into the Flux architecture?

### Flavors

#### Redux
**Recommended Reading**
  - [Redux Documentation](http://rackt.github.io/redux/docs/introduction/Motivation.html)

**Recommended Viewing**
  - [Dan Abramov - Live React](https://www.youtube.com/watch?v=xsSnOQynTHs#t=11m) - Links directly to the 11-minute timestamp where Dan Abramov discusses how traditional Flux stores can be simplified as reducers. Basically, he more elegantly describes everything I'm about to try to explain.

Redux, or "reducers" + "flux", takes the original principles of flux but applies them in an even more purely functional manner. Traditional flux stores are responsible for emitting changes that components can listen to, and this allows state to be mutable. It also means that there is no singular, cohesive application state, but instead multiple top-level stores. The distinction might sound trivial, but it's not and here's why.

First, let's recall the function signature for a reducer. If you're not already familiar with it, here's a refresher:

```js
// (acc, val) -> val
[1,2,3].reduce(function (acc, n) {
  return acc + n;
}); // 6
```

The function takes a collection and runs through them sequentially, "accumulating" a result as it does. The first argument to the callback function is the accumulator, the second is the next item. So what if we approached applications in the same way? The application could start with some initial state, and you get a new state as actions are accumulated.

```js
[createTodo, deleteTodo, toggleCompleteTodo].reduce(function (state, action) {
  // your application!
}, initialState);
```

So now your stores have essentially become _reducers_; they receive actions and return a new state. And since they're pure, you can compose them together into one core application reducer. Actions come in, state comes out. The important thing to note here is that reducers must remain pure - that is, without side effects - and they must return a _new_ state, they cannot mutate the old one.

By following these two simple rules, you not only reduce immediate complexity but you allow other things to occur naturally within the codebase. Redux can naturally notice state changes, not by deep equality checks but just by new references, and automatically update subscribed components. No more `emitChange()`!

Redux provides additional benefits; for one, since state is immutable, it's very easy to record new states as they appear over time. This is the basis for [redux-devtools](https://github.com/gaearon/redux-devtools), allowing you to quite literally step forward and backward through time. It also means it's possible to test applications simply by feeding it a collection of actions and analyzing the result; no extra effort required.

Summary
-------
