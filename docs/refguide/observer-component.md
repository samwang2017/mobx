---
title: (@)observer
sidebar_label: (@)observer
hide_title: true
---

# @observer

<div id='codefund' ></div>

<details>
    <summary style="color: white; background:green;padding:5px;margin:5px;border-radius:2px">egghead.io lesson 1: observable & observer</summary>
    <br />
    <div style="padding:5px;">
        <iframe style="border: none;" width=760 height=427  src="https://egghead.io/lessons/javascript-sync-the-ui-with-the-app-state-using-mobx-observable-and-observer-in-react/embed" ></iframe>
    </div>
    <a style="font-style:italic;padding:5px;margin:5px;"  href="https://egghead.io/lessons/javascript-sync-the-ui-with-the-app-state-using-mobx-observable-and-observer-in-react">Hosted on egghead.io</a>
</details>

The `observer` function / decorator can be used to turn ReactJS components into reactive components.
It wraps the component's render function in `mobx.autorun` to make sure that any data that is used during the rendering of a component forces a re-rendering upon change.
It is available through the separate `mobx-react` package.

```javascript
import { observer } from "mobx-react"

var timerData = observable({
    secondsPassed: 0
})

setInterval(() => {
    timerData.secondsPassed++
}, 1000)

@observer
class Timer extends React.Component {
    render() {
        return <span>Seconds passed: {this.props.timerData.secondsPassed} </span>
    }
}

ReactDOM.render(<Timer timerData={timerData} />, document.body)
```

Tip: when `observer` needs to be combined with other decorators or higher-order-components, make sure that `observer` is the innermost (first applied) decorator;
otherwise it might do nothing at all.

Note that using `@observer` as decorator is optional, `observer(class Timer ... { })` achieves exactly the same.

## Gotcha: dereference values _inside_ your components

MobX can do a lot, but it cannot make primitive values observable (although it can wrap them in an object see [boxed observables](boxed.md)).
So not the _values_ that are observable, but the _properties_ of an object. This means that `@observer` actually reacts to the fact that you dereference a value.
So in our above example, the `Timer` component would **not** react if it was initialized as follows:

```javascript
React.render(<Timer timerData={timerData.secondsPassed} />, document.body)
```

In this snippet just the current value of `secondsPassed` is passed to the `Timer`, which is the immutable value `0` (all primitives are immutable in JS).
That number won't change anymore in the future, so `Timer` will never update. It is the property `secondsPassed` that will change in the future,
so we need to access it _in_ the component. Or in other words: values need to be passed _by reference_ and not by value.

## ES5 support

In ES5 environments, observer components can be simple declared using `observer(React.createClass({ ...`. See also the [syntax guide](../best/decorators.md)

## Stateless function components

The above timer widget could also be written using stateless function components that are passed through `observer`:

```javascript
import { observer } from "mobx-react"

const Timer = observer(({ timerData }) => <span>Seconds passed: {timerData.secondsPassed} </span>)
```

## Observable local component state

Just like normal classes, you can introduce observable properties on a component by using the `@observable` decorator.
This means that you can have local state in components that doesn't need to be managed by React's verbose and imperative `setState` mechanism, but is as powerful.
The reactive state will be picked up by `render` but will not explicitly invoke other React lifecycle methods except `componentWillUpdate` and `componentDidUpdate`.
If you need other React lifecycle methods, just use the normal React `state` based APIs.

The example above could also have been written as:

```javascript
import { observer } from "mobx-react"
import { observable } from "mobx"

@observer
class Timer extends React.Component {
    @observable secondsPassed = 0

    componentWillMount() {
        setInterval(() => {
            this.secondsPassed++
        }, 1000)
    }

    render() {
        return <span>Seconds passed: {this.secondsPassed} </span>
    }
}

ReactDOM.render(<Timer />, document.body)
```

For more advantages of using observable local component state, see [3 reasons why I stopped using `setState`](https://medium.com/@mweststrate/3-reasons-why-i-stopped-using-react-setstate-ab73fc67a42e).

## Observable local state in hook based components

To work with local observable state inside function components, the [`useLocalStore`](https://github.com/mobxjs/mobx-react#uselocalstore-hook) and [`useAsObservableSource`](https://github.com/mobxjs/mobx-react#useasobservablesource-hook) hooks can be used.

## Connect components to provided stores using `inject`

<a style="color: white; background:green;padding:5px;margin:5px;border-radius:2px" href="https://egghead.io/lessons/react-connect-mobx-observer-components-to-the-store-with-the-react-provider">egghead.io lesson 8: inject stores with Provider</a>

_Tip: it is recommended to use `React.createContext` instead_

The `mobx-react` package also provides the `Provider` component that can be used to pass down stores using React's context mechanism.
To connect to those stores, pass a list of store names to `inject`, which will make the stores available as props.

_ N.B. the syntax to inject stores has changed since mobx-react 4, always use `inject(stores)(component)` or `@inject(stores) class Component...`.
Passing store names directly to `observer` is deprecated._

Example:

```javascript
const colors = observable({
   foreground: '#000',
   background: '#fff'
});

const App = () =>
  <Provider colors={colors}>
     <app stuff... />
  </Provider>;

const Button = inject("colors")(observer(({ colors, label, onClick }) =>
  <button style={{
      color: colors.foreground,
      backgroundColor: colors.background
    }}
    onClick={onClick}
  >{label}</button>
));

// later..
colors.foreground = 'blue';
// all buttons updated
```

See for more information the [`mobx-react` docs](https://github.com/mobxjs/mobx-react#provider-and-inject).

## When to apply `observer`?

The simple rule of thumb is: _all components that render observable data_.
If you don't want to mark a component as observer, for example to reduce the dependencies of a generic component package, make sure you only pass it plain data.

With `@observer` there is no need to distinguish 'smart' components from 'dumb' components for the purpose of rendering.
It is still a good separation of concerns for where to handle events, make requests etc.
All components become responsible for updating when their _own_ dependencies change.
Its overhead is neglectable and it makes sure that whenever you start using observable data the component will respond to it.
See this [thread](https://www.reddit.com/r/reactjs/comments/4vnxg5/free_eggheadio_course_learn_mobx_react_in_30/d61oh0l) for more details.

## Optimizing components

See the relevant [section](../best/react-performance.md).

## Characteristics of observer components

-   Observer only subscribe to the data structures that were actively used during the last render. This means that you cannot under-subscribe or over-subscribe. You can even use data in your rendering that will only be available at later moment in time. This is ideal for asynchronously loading data.
-   You are not required to declare what data a component will use. Instead, dependencies are determined at runtime and tracked in a very fine-grained manner.
-   Usually reactive components have no or little state, as it is often more convenient to encapsulate (view) state in objects that are shared with other component. But you are still free to use state.
-   `@observer` implements `shouldComponentUpdate` in the same way as `PureComponent` so that children are not re-rendered unnecessary.
-   Reactive components sideways load data; parent components won't re-render unnecessarily even when child components will.
-   `@observer` does not depend on React's context system.
-   In mobx-react@4+, the props object and the state object of an observer component are automatically made observable to make it easier to create @computed properties that derive from props inside such a component. If you have a reaction (i.e. `autorun`) inside your `@observer` component that must _not_ be re-evaluated when the specific props it uses don't change, be sure to derefence those specific props for use inside your reaction (i.e. `const myProp = props.myProp`). Otherwise, if you reference `props.myProp` inside the reaction, then a change in _any_ of the props will cause the reaction to be re-evaluated. For a typical use case with React-Router, see [this article](https://alexhisen.gitbooks.io/mobx-recipes/content/observable-based-routing.html).
