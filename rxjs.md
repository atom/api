# RxJS and Atom

[RxJS][rxjs] is build around the concept of "observables", which are first class objects representing streams of zero or more values. In Atom, we have two situations in which such facilities would be useful.

## Events

We currently implement events via emissary's `Emitter` mixin. In the following example, we use `::on` to subscribe to all changes and `::once` to subscribe to the next change. Both methods return subscriptions which can be disposed with

```coffee
subscription = editor.on 'text-did-change', -> # ...
subscription = editor.once 'text-did-change', -> # ...
```

Subscribing to events with strings is fragile and difficult to document, shim, and deprecate. We've discussed changing the API to use methods instead, such as `::onTextDidChange`, but there were concerns about losing `::once`. Returning an Rx observable gives us the ability to use `::once` semantics and also ton more.

Our `::on*` methods can be overloaded, so they subscribe if passed a callback and return an observable if no callback is passed.

```coffee
# If a callback is passed, subscribe as normal
subscription = editor.onTextDidChange -> # ...

# If no callback is passed, return an observable
observable = editor.onTextDidChange()

# You can subscribe to observables
subscription = observable.subscribe -> # ...

# But you can also apply operators, such as `::take`.
# This is equivalent to `::once` from the example above.
subscription = observable.take(1).subscribe -> # ...

# You can also apply other operators, such as `::throttle`
# Here we will not handle text changes more often than once per 500ms
subscription = editor.onTextDidChange().throttle(500).subscribe -> # ...
```

## Properties That Change

In addition to events, we also need to track values that change over time. Observables are a great tool for this as well, and could replace `emissary` behaviors. For any model property `foo`, we could have an `observeFoo` method along with `getFoo` and `setFoo`. This would return an observable based on the current value of the property. Whenever someone subscribed, the `onNext` callback would be called immediately with the current value, then again whenever the value changed.

```coffee
# Just like with `::on*` methods, `::observe*` can be passed a callback.
subscription = pane.observeActiveItem -> # ...

# Observe methods return an observable when called without a callback.
observable = pane.observeActiveItem()
subscription = observable.subscribe -> # ...

# If you only want changes and not the current value, skip it
# This is similar to passing `callNow: false` to config.observe,
# and could replace it
subscription = pane.observeActiveItem().skip(1).subscribe(@onActiveItemChanged)
```

Compositional operators can also be used to derive more advanced behaviors, such as `PaneContainer::observeActivePaneItem`, which will be based on the active item of the active pane.

```coffee
class Workspace
  observeActivePaneItem: ->
    @observeActivePane().selectSwitch (activePane) ->
      activePane.observeActiveItem()
```

## Collections That Change

In addition to properties that change, we also deal with collections that change, such as the current panes, pane items, text editors, etc. These can also be modeled as observables. In the following example, `::observePanes` returns an observable that yields every current and future pane to observers.

```coffee
# Can be called with a callback to subscribe immediately
subscription = atom.workspace.observePanes -> # ...

# When called without a callback, returns an observable
subscription = atom.workspace.observePanes().subscribe -> # ...
```

Again, the compositional nature of observables allows us to easily build derived collections, such as a collection based on all pane items.

```coffee
class Workspace
  observePaneItems: ->
    @observePanes().flatMap (pane) -> pane.observeItems()
```

Such a method could filter on optional type selectors, allowing you to observe only a certain subset of panes.

```coffee
class Workspace
  observeTextEditors: ->
    @observePaneItems('.text, .source')

  # The filter step would probably go in Pane::observeItems, but this
  # illustrates the concept.
  observePaneItems: (selector) ->
    @observePanes().flatMap (pane) ->
      pane.observeItems().filter (item) ->
        item.getTypeDescriptor?().matches(selector)
```

## Conclusion

RxJS provides a rich, seemingly mature library for working with streaming data in a compositional manner. We can utilize it in the following scenarios:

  * For events, by adding `::onFoo` methods that return an observable when they aren't passed a callback.
  * For changing properties and collections, by adding `::observeFoo` methods that return observables that invoke subscribers with current and future values.

It seems worth exploring as a robust solution to event handling in Atom that can slot in next to simpler approaches.

[rxjs]: https://github.com/Reactive-Extensions/RxJS
