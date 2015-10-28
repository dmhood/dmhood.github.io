---
layout: post
title: "Styling in React and Atomic CSS"
description: ""
category: Software
tags: [software, react, javascript, css]
---
{% include JB/setup %}

React.js certainly seems to have [ascended as a major FE tool](http://www.google.com/trends/explore?hl=en-US&q=angular.js,+react.js,+ember.js&date=1/2013+30m&cmpt=q&tz=Etc/GMT%2B7&tz=Etc/GMT%2B7&content=1) over the last year or so.  As best practices continue to be ironed out, most elements of web development seem to fitting into the new ecosystem nicely.  However, discussing CSS in the context of React still stirs up some vigorous debate.  CSS best practices in React applications is a fairly divisive topic, particularly since [the developers themselves](https://speakerdeck.com/vjeux/react-css-in-js) seem to eschew “best practices.”  Essentially the two camps are divided into 1.) moving towards component-level styles and declaring CSS directly in their components, or 2.) A more traditional approach of keeping styles in separate, global stylesheets (including using preprocessors like SASS/LESS/etc.).

I encourage you to listen to the presentation above, but in summary it advocates putting your component CSS inside of the component itself as opposed to an outside stylesheet.  There are a myriad of reasons to do so, but essentially it fixes some key and long-standing issues with typical CSS practices (these are taken from the presentation above):

1. Everything is global
2. Dependencies are hard to manage
3. Difficult to remove dead code/unused rules
4. Sharing constants (class names, tags) is tough
5. Separate minification process
6. Non-deterministic resolution (async loading causes potentially different styles)
7. Breaks isolation (teams can override components/modules from other teams, often by accident)


Now many of these issues can be mitigated with best practices, preprocessors, or what have you, but they are all band-aids to the inherently global nature of CSS.   No matter how good your development process, the same issues always seem to keep cropping up.  Sharing code with other teams, using third party libraries, etc. will ensure that regardless of you or your team's skill with CSS, some things are just out of your control and you should be insulating yourself as much as possible.

But isn't the alternative inline styling? And inline styling is bad!  First, what I’m (and others) are advocating isn’t exactly inline styling, it's defining your styles inside of your React components (it’s more than a semantic difference, I promise).  Nevertheless, grouping your CSS with your markup IS bad in a typical web app, but for completeness' sake let’s identify why it is considered such a bad practice, and how it doesn’t apply in this situation:

###### Breaks DRY- I might have to change the same rule 20 times just to edit every button/margin/whatever!  The entire point of CSS is to avoid presentational HTML!
Anytime you find yourself repeating a common style in react, you should be asking yourself why it isn’t in it’s own component.  I’m not saying you won’t have minor duplication here and there, but using components liberally to enforce DRY principles should essentially eliminate most duplication of styles.

###### Hard to Maintain- I need to search through markup to find the rules that I want to change!
This isn’t much of an issue if you are thoughtful about how you separate components.  More importantly, it is going to be much simpler to find a rule inside a specific component that you need to change than searching through several global stylesheets/SASS files and figuring out the proper specificity.

###### Specificity Nightmare- If I have inline styles AND external sheets, specificity issues are sure to creep up on me!
Removing the global namespace of CSS makes it MUCH easier to reason about how styles are being applied to the markup.  The only global CSS that should still be present are really just thematic elements/constants like background-color, etc.

###### No Separation of Concerns- But we are mixing the presentation with structure/behavior!
We are just separating our concerns in a different (and more logical) way.  React separates concerns between components, not technologies like HTML/CSS/JS.  More succinctly, we control complexity by dividing our application into self-contained components that box everything they need to render in one place, not by forcing a separation between intrinsically mixed page assets.

###### Cacheability- I can’t cache my CSS if it’s embedded in the markup!
We can still cache components, but see below for a way to address this issue.

The critical point here isn’t that we are just going back to putting styles in the markup, it is that components themselves aren’t just markup. Components are entirely self-contained independent views, and in order for them to operate effectively (and for us to reason about them easily) they have to be able to render themselves in entirety.  Cascading global CSS disrupts the barrier components have from their surrounding context, irrevocably weakening their ability to fulfill their purpose.  Having CSS in a global namespace instead of the components themselves is like letting a team of artists only draw an outline while you scramble to fill everything in afterwards with a box of Crayolas.

Additionally, this practice isn’t really inline styling. Common CSS classes/rules that are the same within a component can still be pulled out and referenced with a variable, making it easy to change a component’s style on the fly.  Large style blocks can be easily generated inside their own functions, stored in their own variables, etc. so that there is no problems with readability.  And app-level styles like color themes can still be referenced on a traditional stylesheet.  Generating styles programmatically and not directly placing them in an attribute isn’t just syntactic sugar, but a way for more granular and effective organization.

While I truly  think this way is more effective (particularly for large teams), we haven’t really solved all of the issues with CSS.  We still cannot cache our styles between pages without external sheets.  Also, if we are working between multiple teams and importing other react-components, we still may have duplication of styles that we can't really help.  Finally, tons of inline styles can hugely increase the size of the DOM.  While none of these are huge issues (and I think the tradeoffs are worth it for getting rid of some of the major CSS problems above), they are annoyances that we have to deal with.  Luckily, adding one more element to the puzzle can help mitigate each of these.

[Atomic CSS](http://acss.io/) is a framework that seeks to address many of the issues we are trying to solve with React components, only limited to CSS itself.  [This article](http://www.smashingmagazine.com/2013/10/21/challenging-css-best-practices-atomic-approach/) goes over the reasonings and main benefits behind using a modular CSS approach (although it is a bit dated in syntax, the ideas are the same).  Adopting a tool like Atomic CSS fixes many remaining issues with putting our CSS inside the component, such as cacheability of resources and repetition of styles across components (again, mostly when importing components from other teams/3rd party).  

React even solves many of the problems people have with Atomic CSS itself.  The major criticism of Atomic (or any extremely granular CSS technique) is how difficult it is to extricate from the markup and change if required by design, but with a component-based application this is no longer an issue!  React makes is much easier to confine our related Atomic classes inside one component, and we can even fix readability/syntax complaints about Atomic with the use of modules like [classnames](https://github.com/JedWatson/classnames).  There isn't any problems with losing the self-documenting nature of our CSS either, since the components do all of the self-documenting we need.  The combination of Atomic CSS with React gives us an incredibly modular CSS toolset that can be used in every component without fear of losing cacheability, breaking DRY, or becoming too difficult to maintain.

Atomic CSS isn't for everyone, but for large teams it can be a great tool in order to avoid the massive style bloat that is sure to develop over time.  It doesn't even need to be specifically Atomic CSS--any granular CSS framework can go a long way to improving productivity and performance.  However, even if something like Atomic doesn't look appealing (as there is a bit of a learning curve), component-based styles still offer some tremendous advantages.  I hope this post will at least continue to stir up the discussion of how best to implement our CSS styles in React applications.
