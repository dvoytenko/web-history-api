# The case for the new Web History API

The widely available [History API](https://html.spec.whatwg.org/multipage/browsers.html#the-history-interface)
is difficult to use even for the most basic use cases. This has become an impediment for custom-built web apps
and shared component libraries.

There are several key use cases that require use of the History API, such as:

* Web app router for a single-page application (SPA). A client-side router intercepts navigation,
peformance necessary loading and rendering in the same window, and eventually updates URL in the
address bar and the history stack.
* UI overlaying components such as popups, dialogs, etc. There are usually no meaningful address
bar changes, but overlays still need be cancelable using back button. E.g. it's a bad UX if a user
clicks back button on a fancy date-range picker popup, but instead of closing the popup the browser
navigates back to a previous page.
* The ephemeral `history.state` could be a useful storage for fast reloads from bf-cache.
* An old-fashioned `<a href="#...">` fragment navigation.

However, there are major shortcommings in the existing API, including:

1. The `history.state` is unreliable and can disappear from under a web app at any time.
E.g. due to fragment navigation or a push inside an iframe.
2. The `history.state` doesn’t work like a stack. Typically stack properties would recursively
apply in the new state unless overridden. The `history.state`, instead, fully replaces the state
on navigations.
3. The `popstate` event makes very little sense: it fires on pop and fragment navigation
(which is a "push"). It's impossible to process intermediate pops after `window.history.go(-2)`, etc.
4. The history events are not cancelable. Implementing "Are you sure you want to leave without saving"
involves buffering all history changes and re-pushing them back.
5. Implementing the client-side navigation is too complicated. Web apps end up intercepting global click events
and rolling special APIs to navigate programmatically.
6. History and navigation have unpredictable synchronous vs asynchronous semantics.
7. Even in a very carefully crafted web app, an iframe can completely mess up the application's history
stack.

Due to these shortcomings most of apps routinely monkey-patch history APIs and roll their own
non-standard APIs.


## New History API

The main principals for the new API should be:

1. History should be a stack. The state and events should follow stack model. There should be clear
push and pop events for each state in the history and the new pushes should not completely reset the
history stack.
2. Synchronous vs asynchronous nature of the API should be clarified.
3. Ironically, the dependency on history API should be reduced in favor of higher-level APIs.

The following is the discussion of several use cases and possible approaches.

### Client-side navigation

A typical web app could rely on a router with heavy history patching, manual stack state management,
history event interceptors, global click interceptors, etc. Instead, the navigation use case could be
supported more directly using `window.onnavigate` event. It'd cover all possible ways a web app could
navigate, such as `location.assign`, `location.replace`, `location.href` setter,
`form.submit[method=get]`, etc.

A typical web app would only need to do the following to support client-side navigation:

```
window.onnavigate = e => {
  if (router.matches(e.location)) {
    // Stop browser from navigating.
    e.preventDefault();
    router.navigate(e.location);
  }
};
```

To navigate, the user code would simply use normal HTML markup, or would call `location.assign()`
in JavaScript.

That's it. No global listeners needed, no special state management. In fact, the browser's history
is not even manipulated directly here.

### Overlaying UX and back button support

Overlaying UX includes popups, dialogs, etc. A typical implementation would use `history.pushState`
and monitor `popstate` events. There are many issues with handling `popstate` events. There's no
event that says "this specific history state has been popped". Instead, a `popstate` event references
the new "current" state.

To address this, any of the following ideas could be considered:

* The new API could have a new event for all history states that are popped from the history stack.
* The new API could merge the new history state into the existing state, instead of overwriting the
state completely. More on this later.

There's another annoyance with using history stack for this use case. E.g. a popup could create a new
history state. However, if a web page is reloaded, it'd not typically want to re-open a popup. Thus
this history state is not very useful - it's more of a filler to intercept the back button. The back
button support, in this case, is more comparable to the Esc key handling, which most of such UIs
currently do anyway. In fact, maybe a simpler approach for the new API to allow the history state to
be bound to the Esc key, which will be automatically emited when the state is popped.

### Preserving history state

The `history.state` could be a solid option for many use cases, however, it's currently to unpredictable.
For instance, consider the following snippet:

```
window.history.pushState({a: 1}, '', '');

// Now I know my state!
window.history.state.a === 1

// User navigates to a fragment via <a href="#b">

// Oops. The state is lost.
window.history.state === null
```

Instead of completely overwriting the history state, the merge operation could be used, aka `Object.assign()`.
Thus, the snippet would work differently:

```
window.history.pushState({a: 1}, '', '');

// Now I know my state!
window.history.state.a === 1

window.history.pushState({b: 2}, '', '');

// New state:
window.history.state.b === 2
// But the old state is still available:
window.history.state.a === 1

// User navigates to a fragment via <a href="#b">

// The old state is still there:
window.history.state.a === 1
window.history.state.b === 2
```

### Vetoable history stack

A new `beforepopstate` event could be modelled after the [onbeforeunload](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/onbeforeunload).
It wouldn't directly veto history changes, but could instruct the browser show a confirmation prompt.

