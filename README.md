<p align="center"><img src="https://cdn-images-1.medium.com/max/1600/1*DKS5pYfsAz-H45myvnWWVw.gif" style="max-width:75%;"></p>

<h1>react-tracking</h1>

- React specific tracking library, usable as a higher-order component (as `@decorator` or directly)
- Compartmentalize tracking concerns to individual components, avoid leaking across the entire app
- Expressive and declarative (as opposed to imperative) API to add tracking to any React app
- Analytics platform agnostic

Read more in the [Times Open blog post](https://open.nytimes.com/introducing-react-tracking-declarative-tracking-for-react-apps-2c76706bb79a).

## Installation

```
npm install --save react-tracking
```

## Usage
`@track()` expects two arguments, `trackingData` and `options`.
- `trackingData` represents the data to be tracked (or a function returning that data)
- `options` is an optional object that accepts three properties:
  - `dispatch`, which is a function to use instead of the default dispatch behavior. See the section on custom `dispatch()` later in this document.
  - `dispatchOnMount`, when set to `true`, dispatches the tracking data when the component mounts to the DOM. When provided as a function will be called on componentDidMount with all of the tracking context data as the only argument.
  - `process`, which is a function that can be defined once on some top-level component, used for selectively dispatching tracking events based on each component's tracking data. See more details later in this document.

#### Tracking `props`

The `@track()` decorator will expose a `tracking` prop on the component it wraps, that looks like:

```js
{
  // tracking prop provided by @track()
  tracking: PropTypes.shape({
    // function to call to dispatch tracking events
    trackEvent: PropTypes.func,

    // function to call to grab contextual tracking data
    getTrackingData: PropTypes.func,
  })
}
```

This PropType is exported for use, if desired:

```js
import { TrackingPropType } from 'react-tracking';
```

Alternatively, if you just want to just silence proptype errors when using [eslint react/prop-types](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/prop-types.md), you can add this to your eslintrc:

```json
{
  "rules": {
    "react/prop-types" : ["error", { "ignore": ["tracking"] }]
  }
}
```

### Usage as a Decorator
`react-tracking` is best used as a `@decorator()` using the [babel decorators plugin](https://github.com/loganfsmyth/babel-plugin-transform-decorators-legacy).

The decorator can be used on React Classes and on methods within those classes.

```js
import React from 'react';
import track from 'react-tracking';

@track({ page: 'FooPage' })
export default class FooPage extends React.Component {

  @track({ action: 'click' })
  handleClick = () => {
    // ... other stuff
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        Click Me!
      </button>
    );
  }
}
```

### Usage on Stateless Functional Components

You can also track events by importing `track()` and wrapping your stateless functional component, which will provide `props.tracking.trackEvent()` that you can call in your component like so:

```js
import track from 'react-tracking';

const FooPage = (props) => {
  return (
    <div onClick={() => {
        props.tracking.trackEvent({ action: 'click' });

        // ... other stuff
      }}
    />
  )
}

export default track({
  page: 'FooPage'
})(FooComponent);
```

This is also how you would use this module without `@decorators`, although this is obviously awkward and the  decorator syntax is recommended.

### Custom `options.dispatch()` for tracking data

By default, data tracking objects are pushed to `window.dataLayer[]` (see [src/dispatchTrackingEvent.js](src/dispatchTrackingEvent.js)). This is a good default if you use Google Tag Manager. You can override this by passing in a dispatch function as a second parameter to the tracking decorator `{ dispatch: fn() }` on some top-level component high up in your app (typically some root-level component that wraps your entire app).

For example, to push objects to `window.myCustomDataLayer[]` instead, you would decorate your top-level `<App />` component like this:

```js
import React, { Component } from 'react';
import track from 'react-tracking';

@track({}, { dispatch: (data) => window.myCustomDataLayer.push(data) })
export default class App extends Component {
  render() {
    return this.props.children;
  }
}
```

NOTE: It is recommended to do this on some top-level component so that you only need to pass in the dispatch function once. Every child component from then on will use this dispatch function.

### When to use `options.dispatchOnMount`

You can pass in a second parameter to `@track`, `options.dispatchOnMount`. There are two valid types for this, as a boolean or as a function. The use of the two is explained in the next sections:

#### Using `options.dispatchOnMount` as a boolean

To dispatch tracking data when a component mounts, you can pass in `{ dispatchOnMount: true }` as the second parameter to `@track()`. This is useful for dispatching tracking data on "Page" components, for example.

```js
@track({ page: 'FooPage' }, { dispatchOnMount: true })
class FooPage extends Component { ... }
```

Will dispatch the following data (assuming no other tracking data in context from the rest of the app):

```
{
  page: 'FooPage'
}
```

Of course, you could have achieved this same behavior by just decorating the `componentDidMount()` lifecycle event yourself, but this convenience is here in case the component you're working with would otherwise be a stateless functional component or does not need to define this lifecycle method.

_Note: this is only in affect when decorating a Class or stateless functional component. It is not necessary when decorating class methods since any invocations of those methods will immediately dispatch the tracking data, as expected._

#### Using `options.dispatchOnMount` as a function

If you pass in a function, the function will be called with all of the tracking data from the app's context when the component mounts. The return value of this function will be dispatched in `componentDidMount()`. The object returned from this function call will be merged with the context data and then dispatched.

A use case for this would be that you want to provide extra tracking data without adding it to the context.

```js
@track({ page: 'FooPage' }, { dispatchOnMount: (contextData) => ({ event: 'pageDataReady' }) })
class FooPage extends Component { ... }
```

Will dispatch the following data (assuming no other tracking data in context from the rest of the app):


```
{
  event: 'pageDataReady',
  page: 'FooPage'
}
```

### Top level `options.process`

When there's a need to implicitly dispatch an event with some data for *every* component, you can define an `options.process` function. This function should be declared once, at some top-level component. It will get called with each component's tracking data as the only argument. The returned object from this function will be merged with all the tracking context data and dispatched in `componentDidMount()`. If a falsy value is returned (`false`, `null`, `undefined`, ...), nothing will be dispatched.

A common use case for this is to dispatch a `pageview` event for every component in the application that has a `page` property on its `trackingData`:

```js
@track({}, { process: (ownTrackingData) => ownTrackingData.page ? {event: 'pageview'} : null)
class App extends Component {...}

...

@track({page: 'Page1'})
class Page1 extends Component {...}

@track({})
class Page2 extends Component {...}
```

When `Page1` mounts, event with data `{page: 'Page1', event: 'pageview'}` will be dispatched.
When `Page2` mounts, nothing will be dispatched.

### Advanced Usage

You can also pass a function as an argument instead of an object literal, which allows for some advanced usage scenarios such as when your tracking data is a function of some runtime values, like so:

```js
import React from 'react';
import track from 'react-tracking';

// In this case, the "page" tracking data
// is a function of one of its props (isNew)
@track((props) => {
  return { page: props.isNew ? 'new' : 'existing' }
})
export default class FooButton extends React.Component {

  // In this case the tracking data depends on
  // some unknown (until runtime) value
  @track((props, [event]) => ({
    action: 'click',
    label: event.currentTarget.title || event.currentTarget.textContent
  }))
  handleClick = (event) => {
    if (this.props.onClick) {
      this.props.onClick(event);
    }
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        {this.props.children}
      </button>
    );
  }

}
```

NOTE: That the above code utilizes some of the newer ES6 syntax. This is what it would look like in ES5:

```js
// ...
  @track(function(props, args) {
    const event = args[0];
    return {
      action: 'click',
      label: event.currentTarget.title || event.currentTarget.textContent
    };
  })
// ...
```

#### Example `props.tracking.getTrackingData()` usage

Any data that is passed to the decorator can be accessed in the decorated component via its props. The component that is decorated will be returned with a prop called `tracking`. The `tracking` prop is an object that has a `getTrackingData()` method on it. This method returns all of the contextual tracking data up until this point in the component hierarchy.

```js
import React from 'react';
import track from 'react-tracking';

// Pass a function to the decorator
@track(() => {
  const randomId = Math.floor(Math.random() * 100);

  return {
    page_view_id: randomId
  }
})
export default class AdComponent extends React.Component {
  render() {
    const { page_view_id } = this.props.tracking.getTrackingData();

    return (
      <Ad pageViewId={page_view_id} />
    );
  }

}
```

### Tracking Data

Note that there are no restrictions on the objects that are passed in to the decorator.

**The format for the tracking data object is a contract between your app and the ultimate consumer of the tracking data.**

This library simply merges the tracking data objects together (as it flows through your app's React component hierarchy) into a single object that's ultimately sent to the tracking library.
