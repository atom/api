# Workspace::open API Roadmap

## Abstract

We currently keep one text buffer in memory that can be shared between multiple text editors, but the implementation is ad-hoc and package authors can't participate. This roadmap generalizes the pooling of text buffers to the pooling of *resources*, and allows package authors to provide custom resources for specific content types. They can also provide custom *representations*, which are constructed any a many-to-one relationship to resource objects. Tying everything together are *type descriptors*, which are a gentle generalization of top-level grammar scopes to describe arbitrary content types such as `.image.png`.

## Tasks

* [ ] Buffers are created by a resource provider
  * [ ] Add type providers API
  * [ ] Provide `.text.plain` or `.source.x` types for text files
  * [ ] Add resource provider API
  * [ ] Map `.text, .source` selector to TextBuffer resources
* [ ] Editors are created by a resource provider
  * [ ] Add representation provider API
  * [ ] Map `.text, .source` selector to Editor representations
  * [ ] Add language-specific type providers based on grammars
    * [ ] Path glob patterns
    * [ ] Path regexes
    * [ ] Content regexes
  * [ ] Select the grammar based on the type descriptor
* [ ] Switch existing packages to these APIs
  * [ ] Markdown preview
  * [ ] Archive editor
  * [ ] Image viewer

## Description

This is a high- overview of how I want the API to work when these steps are completed. Think of it as a good start to documenting the feature built by this roadmap.

### Type Descriptors

We need a way to describe different types of content in Atom, and I propose using type descriptor strings based on the scopes used in grammars. Currently,  grammars have top-level scopes like `.source.coffee` or `.text.html`.

We can extend this approach to identify other types of content in addition to text, such as `.image.png` and `.archive.tar`. We can then use these descriptors as a vocabulary for creating and querying editors for specific kinds of content.

### Type Providers

*Type providers* map a URI to a type descriptor. Atom supports both regular expressions and [minimatch][minimatch] glob patterns as type providers. Glob patterns are currently only supported for local paths, though glob patterns could potentially be extended to support arbitrary URIs.

In the example below, we map all paths with the `.rb` extension and files with a starting with a ruby shebang line to the `.source.ruby` type descriptor. In addition, a path matching the glob pattern listed in the second provider the next line is given the `source.ruby.rails` type descriptor.

```coffee
atom.workspace.addTypeProvider
  matchURI: "*.rb"
  matchContent: /^#!\\s*\/.*\\bruby/
  type: '.source.ruby'

atom.workspace.addTypeProvider
  matchURI: "app/models/**/*.rb"
  type: ".source.ruby.rails"
```

If multiple type descriptors match a given URI, the most specific selector (with the greater number of `.` characters) wins. In the case of a tie, the most recently added provider wins.

We've discussed other means of selecting a descriptor if multiple matches are provided, such as allowing for arbitrary functions or explicitly listing types that a given provider will defer to or supersede, but for now we're going to see how far we can get with the simple specificity rule.

### Resource Providers

When opening a URI, Atom attempts to recycle or create a *resource object* to represent the URI. Resources are stored in a pool, and the pool will never contain more than one resource for a given URI.

Resources represent the underlying data being edited. For example, even if there are multiple text editors open for the same path in Atom, they will all share the same underlying `TextBuffer` object as their resource.

To provide a custom resource instance for a given type, associate a resource provider with a type selector. Here's the built-in `TextBuffer` provider. Note that providers are asynchronous and will be invoked with a node-style callback.

```coffee
atom.workspace.addResourceProvider
  matchType: ".text, .source"
  create: (uri, resolve, reject) ->
    try
      resolve(new TextBuffer(uri))
    catch error
      reject(error)
```

### Representation Providers

Once the workspace has located or created a resource for the URI, it then attempts to create a *representation* for the resource. While resources are unique, there can be multiple representations for a given resource. It's always a representation object that gets added to the active pane when you open a URI.

The default representation for text files is the `Editor` class:

```coffee
atom.workspace.addRepresentationProvider
  matchType: ".text, .source"
  createWithResource: (buffer, resolve, reject) ->
    try
      resolve(new Editor(buffer)) # Should actually be async
    catch error
      reject(error)
```

You can also provide representations for raw URIs, bypassing the resource pool:

```coffee
atom.workspace.addRepresentationProvider
  matchType: ".text.gfm, .text.markdown"
  createWithURI: (uri, resolve, reject) ->
    try
      resolve(new MarkdownPreview(uri))
    catch error
      reject(error)
```

### Removing Providers

Because of the flexible add interfaces, it's not easy to call `::remove*Provider` with a unique instance. Instead, inspired by the RxJS idiom, the `add*Provider` methods return *disposables*.

A disposable is just an object with a `.dispose()` method that you can call at some point in the future to remove the added provider.

```coffee
disposable =
  atom.workspace.addTypeProvider
    matchURI: "app/models/**/*.rb"
    type: ".source.ruby.rails"

disposable.dispose()
```

### Releasing Resources (In Progress)

Resources should be automatically released from the pool when their representation is removed from the pane system. Currently we manage this manually by calling `.release()` on the buffer when destroying an editor. If we can make this happen automatically without being confusing that would be ideal.

[minimatch]: https://github.com/isaacs/minimatch
