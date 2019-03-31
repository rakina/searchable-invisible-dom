Searchable Invisible DOM
========================

The proposal is to create some new way of exposing DOM to the browser, that is still *searchable*, but is *invisible*. That is, find in page must be able to search within this DOM and fragment navigation should be able to find a named invisible DOM, etc., but we shouldn't need to do all the work of layout/paint/etc.

# New Proposal: Display Locking

Searchable Invisible DOM has merged with [Display Locking](https://github.com/WICG/display-locking), and this explainer is **no longer maintained**. With display locking, developers can **avoid paying rendering costs**, while also allowing user-agent features such as **find-in-page, accessibility, indexability, focus navigation, anchor links, etc. to work** with the display locked elements.

Note that this explainer and repo is no longer maintained, please direct questions or issues to the [Display Locking repo.](https://github.com/WICG/display-locking)

Old Explainer
===

## Background


An increasing amount of content on the web is becoming virtualized: that is, kept in memory in some non-DOM data structure, with only the visible portion converted into DOM nodes and inserted into the document. Motivating examples include <https://m.twitter.com/>, the popular [React Virtualized](https://bvaughn.github.io/react-virtualized/#/components/List) component, and the [<virtual-scroller>](https://github.com/valdrinkoshi/virtual-scroller) layered API proposal.

Virtualized content can create a poor user experience in several areas:

-   **Find in page**: the browser's native find-in-page functionality searches over DOM, which does not contain any off-screen data.
-   **Anchor navigation**: if a visible piece of content links, using `<a href="#anchor">`, to some piece of content that is currently being virtualized, navigation to the anchor will fail.
-   **Indexability**: search engines ingest pages based on rendering them with a headless browser, and inspecting the contents of the DOM (along with other information such as style and layout). Virtualized content thus ends up not indexed by search engines.
-   **Accessibility**: accessibility tools create their accessibility tree from the DOM, and so don't include virtualized content.


Example
-------

Here we have a list of items, and we add more items to the DOM asynchronously, and set up
an event handler to remove the invisible-ness when needed. Note that this is a very simple example and doesn't handle more complex stuff such as adding the invisible-ness to elements that aren't on the viewport anymore, etc.

```html
<ul id="itemsList">
 <div>Item #1</div>
 <div>Item #2</div>
 <div>Item #3</div>
</ul>

<script>
remainingItems.forEach(item => {
  // Add the remaining list items asynchronously.
  requestIdleCallback((deadline) => {
    let itemEl = createElementForItem(item);
    itemEl.setAttribute("invisible", "");
  	 itemsList.appendChild(itemEl);
  }); 
});
 
// Set up handler to show element when needed.
itemsList.addEventListener("activateinvisible", (e) => {
  e.preventDefault();
  let item = e.activatedElement;
  // Show this element and some elements nearby, and hide everything else.
  // This is a very unefficient way of doing it :)
  const currentPosition = getPosition(item);
  for (let i = 0; i < itemsList.children.length; i++) {
    if (shouldBeShown(itemsList.children[i], currentPosition))
      itemsList.children[i].removeAttribute("invisible");
    else
      itemsList.children[i].setAttribute("invisible", "");
   }
 }); 
</script>
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
-   Defer running of scripts under the element
-   [There might be more!]

You can use it by setting the `invisible` attribute value to `static`.

```html
<div invisible="static">
 <my-element>
   I don't have my fancy stuff...
 </my-element>
 <img src="notloaded.jpg">
 <link rel="style" type="text/css" href="notloaded.css">
 <script>
   notRun();
 </script>
</div>
 
<script>
 customElements.define("my-element", class extends HTMLElement {
   constructor() {
     super();
     addFancyStuffToElement();
   }
 });
</script>
```
As we can see from the above example, `static` mode is quite strong and brings many advantages. However web authors need to handle a lot more things by themselves when using the mode, such as asynchronously upgrading the custom elements in the background. 

Activation Event
----------------

| property  | value  |
|---|---|
| bubbles  | true  |
| composed | false |
| target  | the ancestor with the `invisible` attribute |
| activatedElement  | The element that needs activation  |

When a searchable invisible node needs to be shown (see cases below) a DOM event of type `activateinvisible` will be sent to all of its flat tree inclusive ancestors that have the `invisible` attribute, up until the `invisible` root (highest ancestor with the `invisible` attribute) of the element, so that every `invisible` ancestor can be made visible. If there are `N` such ancestors, `N` events will be sent, each of them bubbling up. Events will be sent from the innermost elements first.

Consider this tree structure:
```html
<div id="first">
 <div id="second" invisible>
  <div id="third">
    blah
    <div id="fourth" invisible="static">
     bleh
    </div>
  </div>
 </div>
<div>
```
If we need to activate `#fourth` div, then two `activateinvisible` Events will be fired. One targeted at `#fourth` and one targeted at `#second`. The `activatedElement` in both of them is the `#fourth` div.

If instead we need to activate `#third` div, only one `activateinvisible` Event will be fired, at `#second` with `#third` in the `activatedElement` field. 

### Timing

The `activateinvisible` Event is sent when one of these situations occur:

-   Fragment URL navigation is happening, and the element to be navigated to is searchable-invisible.  After the events are handled, the element will be scrolled-to (unless for some reason the web author kept it invisible)

-   Active match (currently focused match) of find-in-page is changed (through find next/prev [up/down buttons]) such that the next active match is under a searchable-invisible element. After the events are handled, the active match element will be highlighted and scrolled-to (again, unless for some reason the web author kept it invisible)

### Event Handling

If the event is not canceled, the default event handler of the event will remove the `invisible` attribute of the target. As the event is sent to all searchable invisible ancestors, this means it will remove invisibility of all ancestors, making the element and its ancestors show up.

Note that in most cases, you might not want to rely on the default event handler because it only shows the element (and it's ancestors' subtree). Most likely you'll want to also do make some other elements invisible, or maybe make neighboring elements not invisible etc.

Web authors can cancel the event and do more advanced stuff like showing elements not in the viewport, etc. and also use event delegation on the common ancestors of many invisible elements.

Also note as the event is not `composed`, it will not pass through shadow boundaries, so web authors need to make sure if there are `invisible` elements inside a shadow tree, the event listener for it is appropriately placed.

Problems, revisited
-------------------

#### Find in page
Since the contents are in the DOM already, find-in-page should be able to search through the contents of the searchable invisible element. Matches in searchable invisible elements are counted in the total match. They aren't highlighted because they are invisible, but when a certain match needs to be shown (as the active/main/focused match), the activation event `activateinvisible` will be dispatched.

#### Anchor navigation
When we want to navigate to a searchable invisible element, the activation event `activateinvisible` will be dispatched and navigation continues after that.

#### Indexability
Crawler bots will be able to see the contents of searchable invisible elements.

#### Accessibility
Accessibility tree nodes can be constructed from searchable invisible elements in the DOM.


More Examples
----------------
Adding items that needs heavy setup, and handling when it needs to be shown but not ready yet.
```js
listOfExpensiveItems.forEach(expensiveItem => {
  // expensiveItems have basic contents that might be useful/needed early. It will do
  // expensive operations needed for the full version asynchronously.
  // When we do need to show the element we will force the expensive operations.
  let nonCompleteElement = document.createElement(expensiveItem.tagName);
  nonCompleteElement.setAttribute("invisible", "");
  nonCompleteElement.innerHTML = expensiveItem.basicHTMLcontent;
  itemsList.appendChild(nonCompleteElement);

  requestIdleCallback(expensiveItem.doExpensiveOperation); 
  nonCompleteElement.addEventListener("activateinvisible", () => {
    expensiveItem.finishExpensiveOperation();
  });
});
```

Using `activateinvisible` events to load items nearby that aren't fetched yet.

```js
item.addEventListener("activateinvisible", (e) => {
  let el = e.activatedElement;
  // shouldLoadMoreItemsNearElement returns true if we are nearing the end of currently loaded
  // elements, and we should start fetching from server etc to add more elements to the list.
  if (shouldLoadMoreItemsNearElement(el)) {
    loadItemsAfter(el).then((newItems) => insertAsInvisibleElements(items));
  }
});
```

Upgrading custom elements in `static` subtrees asynchronously.

```js
let nonUpgradedElement = document.createElement("my-element");
nonUpgradedElement.setAttribute("invisible", "static");
itemsList.appendChild(nonUpgradedElement);
nonUpgradedElements.push(item);

...

function upgradeElements(deadline) {
  let i = 0;
  for (i = 0; i < nonUpgradedElements.length && deadline.timeRemaining() > 0; ++i){
    customElements.upgrade(nonUpgradedElements[i]);
  }
  if (i < nonUpgradedElements.length) {
    nonUpgradedElements = nonUpgradedElements.slice(i);
    requestIdleCallback(upgradeElements);
  }
}
```

Detecting an element is not in the viewport anymore and adding invisibility to it.
```js
TODO
```
