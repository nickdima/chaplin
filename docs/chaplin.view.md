# [Chaplin.View](src/chaplin/views/view.coffee)

Chaplin’s `View` class is a highly extended and adapted Backbone `View`. All views should inherit from this class to avoid repetition.

Views may subscribe to Publish/Subscribe and model/collection events in a manner which allows proper disposal. They have a standard `render` method which renders a template into the view’s root element (`@el`).

The templating function is provided by `getTemplateFunction`. The input data for the template is provided by `getTemplateData`. By default, this method just returns an object which delegates to the model attributes. Views might override the method to process the raw model data for the view.

In addition to Backbone’s `events` hash and the `delegateEvents` method, Chaplin has the `delegate` method to register user input handlers. The declarative `events` hash doesn’t work well for class hierarchies when several `initialize` methods register their own handlers. The programatic approach of `delegate` solves these problems.

Also, `@model.on()` should not be used directly. Backbone has `@listenTo(@model, ...)` which forces the handler context so the handler can be removed automatically on view disposal. When using Backbone’s naked `on`, you have to deregister the handler manually to clear the reference from the model to the view.


## Features and purpose

* Rendering model data using templates in a conventional way
* Robust and memory-safe model binding
* Automatic rendering and appending to the DOM
* Registering regions
* Creating subviews
* Disposal which cleans up all subviews, model bindings and Pub/Sub events

<a id="initialize"></a>
### initialize(options)
* **options (default: empty hash)**
    * `autoRender` see [autoRender](#autoRender)
    * `container` see [container](#container)
    * `containerMethod` see [containerMethod](#containerMethod)
    * all standard [Backbone constructor
  options](http://backbonejs.org/#View-constructor) (`model`, `collection`,
  `el`, `id`, `className`, `tagName` and `attributes`)

  `options` may be specific on the view class or passed to the constructor. Passing
  in options during instantiation overrides the View prototype's defaults.

  Views must always call `super` from their `initialize` methods. Unlike
  Backbone's initialize method, Chaplin's initialize is required to
  create the instance's subviews and listen for model or collection disposal.

<a id="rendering"></a>
## Rendering: getTemplateFunction, render, …

  Your application should provide a standard way of rendering DOM
  nodes by creating HTML from templates and template data. Chaplin
  provides `getTemplateFunction` and `getTemplateData` for this purpose.

  Set [`autorender`](#autoRender) to true to enable rendering upon
  View instantiation. Will automatically append to a [`container`](#container)
  if one is set, although the method of appending can be overriden
  by setting the [`containerMethod`](#containerMethod) property
  (to `html`, `before`, `prepend`, etc).

<a id="getTemplateFunction"></a>
### getTemplateFunction()
* **function (throws error if not overriden)**

  Core method that returns the compiled template function. Usually
  set application-wide in a base view class.

  A common implementation will take a passed in `template` string and return
  a compiled template function (e.g. a Handlebars or Underscore template function).
```coffeescript
@template = require 'templates/comment_view'
```
or if using templates in the DOM
```coffeescript
@template = $('#comment_view_template').html()
```

if using Handlebars
```coffeescript
getTemplateFunction: ->
  Handlebars.compile @template
```
or if using underscore templates
```coffeescript
getTemplateFunction: ->
  _.template @template
```

  Packages like [Brunch With Chaplin](https://github.com/paulmillr/brunch-with-chaplin)
  precompile the template functions to improve application performance


<a id="getTemplateData"></a>
### getTemplateData()
* **function that returns Object (throws error if not overriden)**

  Empty method which returns the prepared model data for the template. Should
  be overriden by inheriting classes (often from model data).

```coffeescript
getTemplateData: ->
  @model.attributes

...

getTemplateData: ->
  title: 'Winnetou'
  author: 'Karl May'

```

  often overriden in a base model class to intelligently pick out attributes

<a id="render"></a>
### render
  By default calls the `templateFunction` with the `templateData` and sets the html
  of the `$el`. Can be overriden in your base view if needed, though should be
  suitable for the majority of cases.

<a id="attach"></a>
## attach
  Attach is called after the prototype chain has completed for View#render.
  It attaches the View to its `container` element.

<a id="DOM-options"></a>
## Options for auto-rendering and DOM appending

<a id="autoRender"></a>
### autoRender
* **Boolean, default: false**

  Specifies whether the the View's `render` method should be called when
  a view is instantiated.

<a id="container"></a>
### container
* **jQuery object, selector string, or element, default: null**

  A selector for the View's containg element into which the `$el`
  will be rendered. The container must exist in the DOM.

  Set this property in a derived class to specify the container element.
  Normally this is a selector string but it might also be an element or
  jQuery object. View is automatically inserted into the container when
  it’s rendered (in the `attach` method). As an alternative you
  might pass a `container` option to the constructor.

  A container is often an empty element within a parent view.

<a id="containerMethod"></a>
### containerMethod
* **String, jQuery object method (default: 'append')**

  Method which is used for adding the view to the DOM via the `container`
  element. (Like jQuery’s `html`, `prepend`, `append`, `after`, `before` etc.)

## Event delegation

<a id="listen"></a>
### listen
* **Object**

  Property that contains declarative event bindings to non-DOM
  listeners. Just like [Backbone.View#events](http://backbonejs.org/#View),
  but for models / collections / mediator etc.

  ```coffeescript
  class SomeView extends View
    listen:
      # Listen to view events with @on.
      'eventName': 'methodName'
      # Same as @listenTo @model, 'change:foo', this[methodName].
      'change:foo model': 'methodName'
      # Same as @listenTo @collection, 'reset', this[methodName].
      'reset collection': 'methodName'
      # Same as @subscribeEvent 'pubSubEvent', this[methodName].
      'pubSubEvent mediator': 'methodName'
      # The value can also be a function.
      'eventName': -> alert 'Hello!'
  ```

<a id="delegate"></a>
### delegate(eventType, [selector], handler)
* **String eventType - jQuery DOM event, (e.g. 'click', 'focus', etc )**,
* **String selector (optional, if not set will bind to the view's $el)**,
* **function handler (automatically bound to `this`)**

Backbone's `events` hash doesn't work well with inheritance, so
Chaplin provides the `delegate` method for this purpose. `delegate`
is a wrapper for jQuery's `@$el.on` method, and has the same
method signature.

```coffeescript
# For events affecting the whole view:
# delegate(eventType, handler)
@delegate('click', @clicked)

# For events only affecting an element or colletion of elements in the view, pass a selector:
# delegate(eventType, selector, handler)
@delegate('click', 'button.confirm', @confirm)
```

## Regions

Provides a means to give canonical names to selectors in the view. Instead of
binding a view to `#page .container > .sidebar` (via the container) you would
bind it to the declared region `sidebar` which is registered by the view that
contained `#page .container > .sidebar`. This decouples views from those that
nests them. It allows for layouts to be drastically changed without changing
the template.

### region

This is the region that the view will be bound to. This property is not
meant to be set on the prototype -- it is meant to be passed in as part
of the options hash.

Both of the following code snippets will bind the view `MyView` to the
declared region `sidebar`.

This one sets the region directly on the prototype:

```coffeescript
# myview.coffee
class MyView extends Chaplin.View
  region: 'sidebar'

# my_controller.coffee
# [...] inside action method
@view = new MyView()
```

And this one passes in the value of region to the view constructor:

```coffeescript
# myview.coffee
class MyView extends Chaplin.View

# my_controller.coffee
# [...] inside action method
@view = new MyView {region: 'sidebar'}
```

However the latter case allows the controller (through whatever logic) decide
where to place the view.

### regions

Region registration hash that works much like the declarative events hash
present in Backbone.

The following snippet will register the named regions `sidebar` and `body` and
bind them to their respective selectors.

```coffeescript
# myview.coffee
class MyView extends Chaplin.View
  regions:
    '#page .container > .sidebar': 'sidebar'
    '#page .container > .content': 'body'
```

When the view is initialzied the regions hashes of all base classes are
gathered and registered as well. When two views in an inheritance tree
both register a region of the same name, the selector of the most-derived view
is used.

### registerRegion(selector, name)
* **String selector**,
* **String name**

Functionally registers a region exactly the same as if it were in the regions
hash. Meant to be called in the `initialize` method as the following code
snippet (which is identical to the previous one using the `regions` hash).

```coffeescript
class MyView extends Chaplin.View
  initialize: ->
    super
    @registerRegion '#page .container > .sidebar', 'sidebar'
    @registerRegion '#page .container > .content', 'body'
```

### unregisterRegion(name)
* **String name**

Removes the named region as if it was not registered. Does nothing if
there is no region named `name`.

### unregisterAllRegions()

Removes all regions registered by this view, automatically called on
`View#dispose`.


## Subviews

Subviews are usually used for limited scenarios when you want to split a view up into logical sections that are continuously re-rendered or form wizards, etc. but *only when dealing with the same model*.

### subview(name, [view])
* **String name**,
* **View view (when setting the subview)**

  Add a subview to the View to be referenced by `name`. Calling with just the
  `name` argument will return the subview associated with that `name`.

  Subviews are not automatically rendered. This is often done in an
  inheriting view (i.e. in [CollectionView](./chaplin.collection_view.md)
  or your own PageView base class).

### removeSubview(nameOrView)
Remove the specified subview. Can be called with either the `name` associated with the subview, or a reference to the subview instance.

### Usage

```coffeescript
class YourView extends View
  renderSubviews: ->
    @subview 'name', new View
    @subview('name').render()

  attach: ->
    super
    @renderSubviews()
```

# Publish/Subscribe

The View includes the [EventBroker](./chaplin.event_broker.md) mixin to provide Publish/Subscribe capabilities using the [mediator](./chaplin.mediator.md)

## [Methods](./chaplin.event_broker.md#methods-of-chaplineventbroker) of `Chaplin.EventBroker`

### publishEvent(event, arguments...)
Publish the global `event` with `arguments`.

### subscribeEvent(event, handler)
Unsubcribe the `handler` to the `event` (if it exists) before subscribing it. It is like `Chaplin.mediator.subscribe` except it cannot subscribe twice.

### unsubscribeEvent(event, handler)
Unsubcribe the `handler` to the `event`. It is like `Chaplin.mediator.unsubscribe`.

### unsubscribeAllEvents()
Unsubcribe all handlers for all events.
