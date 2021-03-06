## What?

This is a testing ground for adding namespace resolution to the 
`ember-jj-abrams-resolver` (found [here](https://github.com/stefanpenner/ember-jj-abrams-resolver)).

## Why?

Namespaces will allow us to scope modules/assets to make redistribution and reuse of Ember libraries
much easier.

## Explain!

Assume the following module structure (roughly equivalent to folder structure):

```
app/posts/route.js
app/posts/index/route.js
app/posts/index/template.hbs
app/posts/index/item-controller.js
```

Suppose that you wanted to use a third party library (perhaps distributed via Bower) named google-maps.

Lets assume that you downloaded this into a folder named `vendor` in your current project, and that it contained
the following folder structure:

```
g-map/component.js
g-map/template.js
g-map/view.js
g-marker/component.js
g-marker/template.js
models/map_options.js
models/map_type.js
```

Now, if you want to use this map component from within your app code (say from within `app/posts/index/item-controller/template.hbs`)
you could use the following:

```handlebars
{{google-maps@g-map lat=29.169444 long=-82.123508}}
```

This would instruct the resolver/container to find the component named `g-map` in the `google-maps` namespace.

Now the interesting thing that needs to be dealt with is that from within the external namespace the items in that namespace
should not need to be prefixed. This means that if `g-map` wanted to render out a number of `g-marker` components it would
use standard syntax (without the namespace specified):

```handlebars
{{g-marker some-options=here}}
```

### A Few Conventions

* The namepaced code should be able to specify its own custom resolver (so that its internal structure is not dictated by each applications structure),
  and the location for that will be in a module named `<namespace name>/resolver` (or via global lookup as `NamespaceGlobal.Resolver`).
* The namespaced code should be able to supply initializers (via a module named `<namespace name>/initializers`) that would be run on application boot.

## How?

For this to work properly the container will need to maintain a list of namespaces that it is aware of, and a resolver that goes along with it.
When a lookup happens in application code, the container will first determine the namespace it goes with, then lookup or instantiate the resolver
for that namespace (looking in `/resolver.js` first then falling back to the default resolver).

### NamespacedContainer

* The internally looked up items will be in an `ember` namespace. This would include `route:basic`, `controller:array`, `controller:object` etc.
* All container lookups (and internal functions) would be modified to require a namespace parameter.
* A new `NamespacedContainer` object will be created that will be initialized with a default namespace, but will have the same method signature as the current container.
* Each namespace will get a `NamespacedContainer` object instead of the raw container. This allows lookups from within the namespace to use that namespaces code/modules by default.

### Lookup path

* The namespace "lookup path" can be modified within each object created.

  An example syntax of this might look like:

  ```javascript
  export default Ember.Route.extend({
    lookupPath: ['app', 'ember']  // this would be the default value and would not be required
  });
  ```

  Which would amount to look in the `app` namespace, then look in the `ember` namespace.

  The default lookup path can be modified, and items would be looked up in the specified order:

  ```javascript
  export default Ember.Route.extend({
    lookupPath: ['app', 'google-maps', 'ember']
  });
  ```
* Any object that specifies a different `lookupPath` would get a different instance of `NamespacedContainer` for that set (but only one would be created for each combination of default namespace and `lookupPath`).
* `lookupPath` is a concatenated property.
* The lookupPath is locked down at extend (i.e. design) time and should not be modified after instatiation..
* Template namespaces load path can be specified via either namespaced lookups (i.e. `{{google-maps@g-marker lat=89 long=99}}`) or by importing that namespace (i.e. `{{import 'google-maps'}}`).
* Importing a namespace in a template would only affect that particular template (not the entire application's templates).
* Namespaces will be registered (similar to `libraries` now) so that errors/assertions can be made if the namespace isn't present during development.

### Ember CLI

* Will be modified to automatically load any Bower packages that specify an `ember-plugin` property value of true (removing the need to manually modify script imports).
* Registers `<namespace (aka bower package) name>/resolver` into the main (non-namespaced) container.
* Requires/registers `<namespace (aka bower package) name>/initializers` to be run at application boot.
