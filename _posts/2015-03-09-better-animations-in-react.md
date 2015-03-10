---
layout: post
title: "Better Animations in React"
description: ""
category: "Software"
tags: [software, react, javascript]
---
{% include JB/setup %}


Animating content in react can be a bit tricky, especially if you are trying to rely on the [ReactCSSTransitionGroup](http://facebook.github.io/react/docs/animation.html) library that is part of the React/addons package.  The crux of the issue is that React’s DOM-diffing doesn’t exactly work when you need to animate an element leaving the page, especially if that element is being replaced by a similar component (or just the same component with different data).  In this case, instead of diffing the current DOM, you need an entirely new component to be rendered, while apply a ‘leave’ animation to the old component/DOM node.  ReactCSSTransitionGroup seeks to do this for you by applying an animation-leave and animation-enter class to the two nodes, but since it relies on the transitionEnd event this can be pretty frustrating.

The transition event (in my experience) is simply not reliable enough to use in a user-facing application.  TransitionEnd can fail to fire if the user changes tabs, if there is any sort of error, or a myriad of other reasons.  When this fails to fire, React won’t remove the old node from the page while still adding the new node.  The result is a broken looking experience at best, and likely an unusable page at worst.

The solution to this is to use a different mechanism to signal that the node should be removed from the DOM.  I’ve been using a library adapted from Khan Academy's [TimeoutTransitionGroup](https://github.com/Khan/react-components/blob/master/js/timeout-transition-group.jsx), which uses simple timeouts to signal animation competition.  The downside of this is that you must manually set the timeout time to be the same as the animation time, but this is a small price to pay for an infinitely more reliable mechanism.  

It would be great if ReactCSSTransitionGroup was more robust.  Maybe adding a fallback timeout event or the ability to specify your own events for node removal would help, but I guess that is what the underlying TransitionGroup library is for.
