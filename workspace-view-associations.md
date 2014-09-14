# Workspace View Associations Roadmap

## Abstract

We currently promote access to the view layer through `atom.workspaceView`. Instead we want to promote access to the model first via `atom.workspace`. If the user needs access to a view, they can retrieve the view corresponding to a pane item, panel, or workspace model object via `atom.workspace.getView()`.

## Tasks

* [ ] Pane items can only be inserted once in the pane tree
* [ ] Pane item views are constructed via view providers
* [ ] Views can be retrieved for pane items
* [ ] Views can be retrieved for panes
* [ ] Views can be retrieved for the workspace itself
* [ ] Panels can be created at the model level
  * [ ] Find and replace uses the panel model API
    * [ ] Use view providers to construct panel views
    * [ ] Hide and show panels via model API
  * [ ] The tree view uses the panel model API
    * [ ] Handle panel resizing in the panel wrapper view
* [ ] Deprecate `atom.workspaceView`

## Description

### Registering View Providers

The workspace should allow view providers to be registered to associate views with certain models.

```coffee
atom.workspace.addViewProvider
  matchConstructor: Editor
  create: (editor, resolve, reject) ->
    try
      resolve(new EditorView(editor))
    catch error
      reject(error)
```

### Retrieving Views

Then we should allow views to be retrieved for a given pane item.

```coffee
editorView = atom.workspace.getView(editor)
```

The same mechanism can also be used to access pane views.

```coffee
paneView = atom.workspace.getView(atom.workspace.getActivePane())
```

### Model-Level Panels API

The final step is to incorporate a model-level API for controlling panels like find and replace and the tree view. In this example, I name the "tree view" to the "project browser" since using the word "view" to name a model is confusing.

```coffee
atom.workspace.addViewProvider
  matchConstructor: ProjectBrowser
  create: (projectBrowser, resolve, reject) ->
    try
      resolve(new ProjectBrowserView(projectBrowser))
    catch error
      reject(error)

# Create a panel based on the project browser (tree view)
panel = atom.workspace.addLeftPanel(new ProjectBrowser)
panel.isActive() # Is the panel showing?
panel.activate() # Show the panel
panel.deactivate() # Hide the panel
panel.setResizable(true) # Make the panel resizable.

atom.workspace.getView(panel) # Get the panel view
atom.workspace.getView(panel.getItem()) # Get the project browser view
```
