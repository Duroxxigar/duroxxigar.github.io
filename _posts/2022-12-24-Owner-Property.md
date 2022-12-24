---
title: "What is \"Owner\" in Godot?"
categories:
  - Godot
---

In the simplist terms, the `owner` is the root node of a scene. When saving a scene in the editor, it will be set for you. However, if you are dynamically creating a node and trying to save a new scene, you must set the property yourself. Assuming your script is on the root node, the code would look something like this.

```swift
var new_node = Node.new()
add_child(new_node)
new_node.owner = owner
```

The property should be set after being added as a child. Otherwise, it may not set properly.

Knowing that `owner` is going to be the root node of a scene, this means that you can use this as a quick way to get the root node. So in nested nodes, you don't have to chain `get_parent()` calls. You can simply do `owner.get_node("SomeNodeICareAbout")`. This kind of pattern can be useful for a component style design where you might want to interact with the root node automatically. Now of course, you can also just export the node path for the root node and manually set it.

If you are running a scene as a standalone scene and you try to access the `owner` property on the root node of the scene, it will not have one because it is the root node already. However, if you instanced that scene into another scene, it will have a valid `owner` set. Keep in mind that if you instance a scene through code, that method returns the root node of the scene, so you can still set the owner there for a dynamically instanced scene.