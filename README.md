Searchable Invisible DOM
========================

The proposal is to create some new way of exposing DOM to the browser, that is still *searchable*, but is *invisible*. That is, find in page must be able to search within this DOM and fragment navigation should be able to find a named invisible DOM, etc., but we shouldn't need to do all the work of layout/paint/etc.

## Background


An increasing amount of content on the web is becoming virtualized: that is, kept in memory in some non-DOM data structure, with only the visible portion converted into DOM nodes and inserted into the document. Motivating examples include <https://m.twitter.com/>, the popular [React Virtualized](https://bvaughn.github.io/react-virtualized/#/components/List) component, and the [<virtual-scroller>](https://github.com/valdrinkoshi/virtual-scroller) layered API proposal.

Virtualized content can create a poor user experience in several areas:

-   **Find in page**: the browser's native find-in-page functionality searches over DOM, which does not contain any off-screen data.

-   **Anchor navigation**: if a visible piece of content links, using `<a href="#anchor">`, to some piece of content that is currently being virtualized, navigation to the anchor will fail.

-   **Indexability**: search engines ingest pages based on rendering them with a headless browser, and inspecting the contents of the DOM (along with other information such as style and layout). Virtualized content thus ends up not indexed by search engines.

-   **Accessibility**: accessibility tools create their accessibility tree from the DOM, and so don't include virtualized content.


Example
-------

```html
stuff
```


Behavior
--------

By default, invisible elements are quite similar to `display:none` elements except that find-in-page, fragment url navigation, crawler bots and accessibility will know to not ignore them. Elements that are searchable invisible are marked with the `invisible` attribute. If a previously searchable invisible element got its `invisible` removed, the element will show up as normal (unless it is a flat-tree descendant of another searchable invisible element)

The element's subtree is treated as if it has `display:none`. If an element has the `invisible` attribute, all of its inclusive flat-tree descendants ("searchable invisible descendants") are all searchable invisible too.

```html
<div invisible>
 You can't see me!
 <div>
   Can't see me either!
 </div>
</div>
```

There is a stronger mode, `static` searchable invisible DOM. In this mode, the behaviors in the normal version are there but some expensive operations in the searchable invisible subtree are also not done/deferred until the invisible attribute is removed.

Some things that the `static` mode will do:

-   Defer custom element upgrades, unless specifically trigerred by `CustomElementsRegistry.upgrade`
-   Defer loading of resources
-   [There might be more!]

Problems, revisited
-------------------

### Find in page
Since the contents are in the DOM already, find-in-page should be able to search through the contents of the searchable invisible element. Matches in searchable invisible elements are counted in the total match. They aren't highlighted because they are invisible, but when a certain match needs to be shown (as the active/main/focused match), the activation event `activateinvisible` will be dispatched.

### Anchor navigation
When we want to navigate to a searchable invisible Element, the activation event will be dispatched and navigation continues after that.

### Indexability
Crawler bots will be able to see the contents of searchable invisible elements.

### Accessibility
Accessibility tree nodes can be constructed for searchable invisible elements.

Activation Event
----------------

When a searchable invisible node needs to be shown (see cases below) a DOM event of type `activateinvisible` will be sent to all of its flat tree inclusive ancestors that have the `invisible` attribute, up until the `invisible` root (highest ancestor with the `invisible` attribute) of the element. If there are `N` such ancestors, `N` events will be sent, each of them bubbling up.

| property  | value  |
|---|---|
| bubbles  | true  |
| composed | false |
| target  | the ancestor |
| activatedElement  | The element that needs activation  |


### Timing

The `activateinvisible` Event is sent when one of these situations occur:

-   Fragment URL navigation is happening, and the element to be navigated to is searchable-invisible.  After the events are handled, the element will be scrolled-to (unless for some reason the web author kept it invisible)

-   Active match (currently focused match) of find-in-page is changed (through find next/prev [up/down buttons]) such that the next active match is under a searchable-invisible element. After the events are handled, the active match element will be highlighted and scrolled-to (again, unless for some reason the web author kept it invisible)

### Event Handling

If the event is not canceled, the default event handler of the event will remove the `invisible` attribute of the target. As the event is sent to all searchable invisible ancestors, this means it will remove invisibility of all ancestors, making the element and its ancestors show up.

Note that in most cases, you might not want to rely on the default event handler because it only shows the element (and it's ancestors' subtree). Most likely you'll want to also do make some other elements invisible, or maybe make neighboring elements not invisible etc.

Web authors can cancel the event and do more advanced stuff like showing all previous elements, etc. A very rough example of how this might be used is as follows:

