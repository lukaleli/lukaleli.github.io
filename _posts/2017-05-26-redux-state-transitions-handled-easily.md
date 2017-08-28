---
layout: post
title:  "Redux state transitions handled easily"
date:   2017-05-26 14:34:25
categories: redux react javascript
image: /assets/article_images/2017-05-26-redux/cover.jpg
---
So you've started to write your first React application. After some time it turned out that you need to share data between different components. So you add an opinionated Redux store to manage your application's state. Then, at some point, you need to connect your app to the backend API to fetch and update some data. So you add `redux-saga` or `redux-thunk` to manage all the side effects of your redux actions. Then you add some additional flags in your chunks of data in the Redux store to reflect different states of your app that usually are dispatched as a side effects, e.g. `isUploading` or `isFetching`. All that to show some fancy loaders to the users when async operation is done in the background. But tracking these changes from, say, `isUploading: true` to `isUploading: false` and vice-versa is always a tedious task. Especially when you want to delay UI response to these changes (e.g. show some message for specified time and then hide it). And code that is produced in effect is usually hard to read. 

I started to look for a declarative solution that's easy to read and maintain. I always liked the idea of a callback/lifecycle methods that are self-descriptive and make it easy to hook up your code to the specific events. What if we could have specific value changes in Redux store mapped to well-named callback methods in our connected components, e.g.

{% highlight js %}

// TRANSITION isUploading: false >>> isUploading: true TRIGGERS:
onUploadStart()
// TRANSITIONS isUploading: true >>> isUploading: false TRIGGERS: 
onUploadEnd()
// TRANSITION error: undefined >>> error: Error(...) TRIGGERS:
onError()

{% endhighlight %}

That would be awesome, right?

## Solution
I came up with a simple utility function that hooks up to a `componentWillReceiveProps` lifecycle method of a redux-connected component and triggers appropriate callback method specified in declared mappings between callback method names and rules of state changes. Here's an implementation of that utility function that's named `mapPropsChangeToCallbacks`:

**mapPropsChangeToCallbacks.js** 

{% highlight js %}

// @flow

type Rule = {
  method: string,
  rule: Function
}

export default function (instance: object = {}, mappings: Array<Rule> = [], prev: object = {}, next: object = {}) {
  mappings.forEach(mapping => {
    if (mapping.rule(prev, next)) {
      instance[mapping.method] && instance[mapping.method]()
    }
  })
}

{% endhighlight %}

And its usage:

**MyComponent.js**
{% highlight js %}

import React, { Component } from 'react'
import { connect } from 'react-redux'
import mapPropsChangeToCallbacks from './mapPropsChangeToCallbacks'

// Specify props transition that should trigger specific callback
const mappings = [
  { method: 'onUploadStart', rule: (prev, next) => !prev.isUploading && next.isUploading },
  { method: 'onUploadEnd', rule: (prev, next) => prev.isUploading && !next.isUploading },
  { method: 'onUploadError', rule: (prev, next) => !prev.error && next.error }
]

// Connect your component to the redux store and map to component props
// In that case I'm only interested in tracking if document is uploading and if error has occurred
@connect( ({ documents }) => ({isUploading: documents.isUploading, error: documents.error}) )
class MyComponent extends Component {
  
  componentWillReceiveProps(nextProps) {
    // Call mapPropsChangeToCallbacks in componentWillReceiveProps lifecycle method
    mapPropsChangeToCallbacks(this, mappings, this.props, nextProps)
  }

  // These methods will get triggered when one of state changes you specified in mappings array occurred
  onUploadStart() {
    console.log('onUploadStart')
  }

  onUploadEnd() {
    console.log('onUploadEnd')
  }

  onUploadError() {
    console.log('onUploadError')
  }
}

export default MyComponent

{% endhighlight %}


We can additionally enhance our `mapPropsChangeToCallbacks` function by adding error throwing when you forget to specify callback method in a component class

**mapPropsChangeToCallbacks.js** 
{% highlight js %}

// @flow

type Rule = {
  method: string,
  rule: Function
}

export default function (instance: object = {}, mappings: Array<Rule> = [], prev: object = {}, next: object = {}) {
  mappings.forEach(mapping => {
    if (mapping.rule(prev, next)) {
      if (instance[mapping.method]) {
        instance[mapping.method]()
      } else {
        throw new Error('You forgot to declare ' + mapping.method + ' in your component class.')
      }
    }
  })
}

{% endhighlight %}

Hope you find it helpful!

## [Update]
As I started to use this approach I found it a bit cumbersome to use. I needed to override `didComponentUpdate` with `mapPropsChangeToCallbacks`. I decided to do something about it and I converted my solution to even more generic one- Higher Order Component (HOC). But what is HOC exactly? If you go to the [official React's page](https://facebook.github.io/react/docs/higher-order-components.html) you can read that: 
> A higher-order component (HOC) is an advanced technique in React for reusing component logic. HOCs are not part of the React API, per se. They are a pattern that emerges from React's compositional nature.

If you're using Redux library in your project you are familiar with the `connect` function from Redux's package. `connect` is responsible for wrapping your component with HOC that takes care of passing values from your store as props to your connected component. All that out of the box. And this is basically what HOC is: a component wrapper around your component. Widely used pattern is to write a function that receives a component as an argument and returns new decorated component with added functionality. Or if you are composing multiple HOCs then a good pattern is to apply currying which is also open for extension if we want to add some parameters to the HOC at the time of creation. 

Let's see how it looks like in action. Below is our improved version of `mapPropsToCallbacks` using HOC.

**withCallbacks.js**
{% highlight js %}

import React, { Component } from 'react'
import hoistNonReactStatics from 'hoist-non-react-statics'

export const mapPropsChangeToCallbacks = (
  instance: Object = {},
  mappings: Object = {},
  prev: Object = {},
  next: Object = {},
) => {
  if (!instance) return
  Object.entries(mappings).forEach(([prop, rule]) => {
    if (rule(prev, next)) {
      if (instance[prop]) {
        instance[prop]()
      } else {
        throw new Error('You forgot to declare ' + prop + ' in your component class.')
      }
    }
  })
}

/*
* withCallbacks function should go as the last one
* wrapping target component as it needs to refer
* to the component which has callbacks specified
* */

const withCallbacks = mappings => (WrappedComponent) => {
  class CallbackComponent extends Component {
    wrappedComponent

    componentDidUpdate(prevProps) {
      mapPropsChangeToCallbacks(this.wrappedComponent, mappings, prevProps, this.props)
    }

    render() {
      return (
        <WrappedComponent
          ref={(ref) => {
            this.wrappedComponent = ref
          }}
          {...this.props} />
      )
    }
  }

  return hoistNonReactStatics(CallbackComponent, WrappedComponent)
}

export default withCallbacks

{% endhighlight %}

As you can see in the code above we've basically done the same thing that we did over and over again in every component where we wanted to map our props changes to callbacks. But in our HOC implementation there are few additional steps we need to take care of. We need to keep the reference to wrapped component in our HOC so we can execute public callback methods specified in the wrapped component. To do that we need to use `ref` prop that takes callback returning reference to the component. We are storing that in the wrappedComponent variable. We also need to pass down the props that our component will be receiving in its lifecycle. We can do that by simply using spread operator. Last but not least- we need to copy all non-React static methods to the wrapped component. `hoist-non-react-statics` will do that automatically for us.
Now let's use our previous component and apply our `withCallback` function to it:

**MyComponent.js**
{% highlight js %}

import React, { Component } from 'react'
import { connect } from 'react-redux'
import { compose } from 'redux'
import withCallbacks from './withCallbacks'

class MyComponent extends Component {
  onUploadStart() {
    console.log('onUploadStart')
  }

  onUploadEnd() {
    console.log('onUploadEnd')
  }

  onUploadError() {
    console.log('onUploadError')
  }
}

const mappings = {
  onUploadStart: (prev, next) => !prev.isUploading && next.isUploading,
  onUploadEnd: (prev, next) => prev.isUploading && !next.isUploading,
  onUploadError: (prev, next) => !prev.error && next.error,
}
const enhance = compose(
  connect( ({ documents }) => ({isUploading: documents.isUploading, error: documents.error}) ),
  withCallbacks(mappings)
)

export default enhance(MyComponent)

{% endhighlight %}

By currying our `withCallback` function we can compose it with other HOCs easily using `compose` utility from `redux` library. 

**NOTE**

In case you're applying multiple HOCs to your component (as in the example above) `withCallbacks` **MUST go as the last function** in the composition chain. Why? Because we need to keep reference to the component that has callbacks specified. Otherwise our HOC won't work.

**NOTE 2**

Please note that I've changed the format of mappings from array of objects to the single object with properties in the format of `<callbackName>: <condition>`. I find it more concise and shorter :)

Enjoy!

