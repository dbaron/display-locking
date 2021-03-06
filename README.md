# Display locking

## Introduction

Subtree Visibility (a.k.a. Display Locking) is a CSS property designed to allow
developers and browsers to easily scale to large amount of content and control
when rendering [\[1\]](#foot-notes) work happens. More concretely, the goals
are:

* Avoid rendering and layout measurement work for content not visible
  to the user
* Support [user-agent features](https://github.com/WICG/display-locking/blob/master/user-agent-features.md)
  and all layout algorithms (e.g. responsive design, flexbox, grid) for this
  content

The following use-cases motivate this work:

1. Fast display of large HTML documents (examples: HTML one-page spec; other long
  documents)
2. Deep links and searchability into pages with hidden content (example: mobile
  Wikipedia; scroll-to-text support for collapsed sections)
3. Scrollers with a large amount of content, without resorting to virtualization
  (examples: Facebook and Twitter feeds, CodeMirror documents)
4. Measuring layout for content not visible on screen
5. Optimizing single-page-app transition performance

## Quick links

* [`subtree-visibility` spec draft](https://wicg.github.io/display-locking/index.html)
* [`contain-intrinsic-size` explainer](https://github.com/WICG/display-locking/blob/master/explainer-contain-intrinsic-size.md)
* [`beforematch` explainer](https://github.com/WICG/display-locking/blob/master/explainer-beforematch.md)

## Quick summary

In the below, "invisible to rendering/hit testing" means not drawn or returned
from any hit-testing algorithms in the same sense as `visibility: hidden`, where
the content still conceptually has layout sizes, but the user cannot see or
interact with it.

Also, "visible to UA algorithms" means that find-in-page, link navigation, etc
can find the element.

`subtree-visibility: visible` - default state, subtree is rendered.

`subtree-visibility: auto` - avoid rendering cost when offscreen
* Use cases: (1), (3), (5)
* Applies `contain: style layout`, plus `contain: size` when invisible
* Invisible to rendering/hit testing, except when subtree intersects viewport
* Visible to UA algorithms

`subtree-visilibity: hidden-matchable` - allow developer to control toggles between invisible/not-invisible states
* Use cases: (2), (3), (5)
* Applies `contain: style layout size`
* Invisible to rendering/hit testing
* Visible to UA algorithms. UA fires event when matched, but not automatically displayed

`subtree-visilibity: hidden` - hide content, but preserve cached state and still support style/layout measurement APIs
* Use cases: (4), (5)
* Applies `contain: style layout size`
* Invisible to rendering/hit testing
* Not visible to UA algorithms

`contain-intrinsic-size: <length> <length>` - placeholder sizing while invisible
* Use cases: (1), (3), (5)

## Motivation & background

On the one hand, faster web page loads and interactions directly improve the
user experience of the web. On the other hand, web sites each year grow larger
and more complex than the last, in part because they support more and more use
cases, and contain more information, and the most common UI pattern for the
web is scrolling. This leads to pages with a lot of non-visible (offscreen or
hidden) DOM, and since the DOM presently renders atomically, it inherently
takes more and more time to render on the same machine.

For these reasons, web developers need ways to reduce loading and rendering time
of web apps that have a lot of non-visible DOM. Two common techniques are to mark
non-visible DOM as "invisible" [\[2\]](#foot-notes), or to use virtualization
[\[3\]](#foot-notes). Browser implementors also want to reduce loading and
rendering time of web apps. Common techniques to do so include adding caching of
rendering state [\[4\]](#foot-notes), and avoiding rendering work
[\[5\]](#foot-notes) for content that is not visible.

These techniques can work in many cases but have drawbacks and limitations:

a. [\[2\]](#foot-notes) and [\[3\]](#foot-notes) usually means that such content
   is not available to user-agent features, such as find-in-page functionality.
   Also, content that is merely placed offscreen may or may not have rendering
   cost (it depends on browser heuristics), which makes the technique
   unreliable.
  
b. Caching intermediate rendering state is [hard
   work](https://martinfowler.com/bliki/TwoHardThings.html), and often has
   performance limitations and cliffs that are not obvious to developers.
   Similarly, relying on the browser to avoid rendering for content that is clipped
   out or not visible is sometimes not reliable, as it's hard for the browser to
   efficiently detect what content is not visible and does not affect visible
   content in any way.

Previously adopted web APIs, in particular the
[contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) and
[will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) CSS
properties, add ways to specify forms of rendering
[isolation](https://github.com/chrishtr/rendering/blob/master/isolation.md) or
isolation hints, with the intention of them being a mechanism for the web
developer to help the browser optimize rendering for the page.

While these forms of isolation help, they do not guarantee that isolated content
does not need to be rendered at all. Ideally there would be a way for the
developer to specify that specific parts of the DOM need not be rendered, and
pair that with a guarantee that when later rendered, it would not invalidate
more than a small amount of style, layout or paint in the rest of the document.

## Disclaimer

As the proposed features evolve, several competing API shapes might be
considered at the same time, the decisions on particular behaviors might not be
finalized, and some documentation may be out of date.

For the latest implemented behavior and API state, please consult the
[cheatsheet](https://github.com/WICG/display-locking/blob/master/cheatsheet.md).

For behaviors being discussed, as well as questions and other discussions,
please look over the [issues](https://github.com/WICG/display-locking/issues)

The rest of this document talks about one particular implementation option.
Whether or not this is the final proposed set of features is yet undecided.

## Description of proposal

Three new features are proposed:

1. A new `subtree-visibility` CSS property.
  This property controls whether DOM subtrees affected by the property are
  invisible to painting/hit testing. This is the mechanism by which rendering
  work can be avoided. Some values of `subtree-visibility` allow the user-agent
  to automatically manage whether subtrees affected are rendered or not. Other
  values give the developer complete control of subtree rendering. Note that the
  names of the tokens are being
  [discussed](https://github.com/WICG/display-locking/issues/110).
  However, the brief description of the tokens is below:
    * `subtree-visibility: auto`: this configuration allows the user-agent to
      automatically manage whether content is invisible to rendering/hit testing or not.
    * `subtree-visilibity: hidden`: this configuration gives the
      developer complete control of when the subtree is rendered. Neither the
      user-agent nor its features should need to process or render the subtree.
    * `subtree-visilibity: hidden-matchable`: this configuration
      allows the developer to control rendering, but it allows user-agent
      features such as find-in-page to process the subtrees and fire the
      activation event (described below).

  It is also worth noting that when the element is not rendered, then
  `contain: layout style size;` is added to its style to ensure that the
  subtree content does not affect elements outside of the subtree.
  Furthermore, when the element is rendered in the `subtree-visilibity: auto`
  configuration (i.e. the user-agent decides to render the element), then
  `contain: layout style;` applies to the element.

2. A new `contain-intrinsic-size` CSS property (spec draft to be added soon). This
  sizing property dictacts how much space should be reserved for the subtree
  when it is not rendered. More specifically, it is an intrinsic size to apply
  when `contain: size` style applies to the element. For more details, please
  refer to [this
  explainer](https://github.com/WICG/display-locking/blob/master/explainer-contain-intrinsic-size.md).

3. A new [`beforematch` event](https://github.com/WICG/display-locking/blob/master/explainer-beforematch.md)
  which is fired when one of the specified UA algorithms targets an element. For
  example, it would fire if find-in-page locates text that becomes the active,
  or currently selected, match. 

## Example usage

```html
<style>
.locked {
  subtree-visilibity: auto;
  contain-intrinsic-size: 100px 200px;
}
</style>

<div class=locked>
  ... some content goes here ...
</div>
```

The `.locked` element's `subtree-visibility` configuration lets the user-agent
manage rendering the subtree of the element. Specifically when this element is
near the viewport, the user-agent will begin rendering the element. When the
element moves away from the viewport, it will stop being rendered.

Recall that when not rendered, the property also applies size containment to the
element. This means that when not rendered, the element will use the specified
`contain-intrinsic-size`, making the element layout as if it had a single block
child with 100px width and 200px height. This ensures that the element still
occupies space when not rendered. At the same time, it lets the element size to
its true contents when the subtree is rendered (since size containment no longer
applies), thus removing the concern that estimates like 100x200 are sometimes
inaccurate (which would otherwise result in displaying incorrect layout for
on-screen content).

One intended use-case for this configuration is to make it easy for developers to
avoid rendering work for off-screen content.

A second use-case is to support simple scroll virtualization.

```html
<style>
.locked {
  subtree-visibility: hidden;
  contain-intrinsic-size: 100px 200px;
}
</style>

<div class=locked>
  ... some content goes here ...
</div>
```

In this case, the rendering of the subtree is managed by the developer only.
This means that if script does not modify the value, the element's subtree will
remain unrendered, and it will use the `contain-intrinsic-size` input when
deciding how to size the element. However, the developer can still call methods such
as `getBoundingClientRect()` to query and measure layout for the invisible content.

One intended use-case for this mode are measuring layout geomery for content not displayed.

A second use-case is preserving rendering state for [single-page app](https://en.wikipedia.org/wiki/Single-page_application)
content that is not currently visible to the user, but may be displayed again
soon via user interaction.

```html
<style>
.locked {
  subtree-visibility: hidden-matchable;
}
</style>

<div class=locked>
  ... some content goes here ...
</div>
```

Similar to above, the render of the subtree is managed by the developer.
However, it allows find-in-page to search for text within the subtree and fire
the activation signal if the active match is found. 

One intended use-case for this configuration is that the subtree is hidden and
"collapsed" (note the absense of `contain-intrinsic-size` which makes size
containment use empty size for intrinsic sizing). This is common when content is
paginated and the developer allows the user to expand certain sections with
button clicks. In the `subtree-visibility` case the developer may also listen to
the activation event and start rendering the subtree when the event targets the
element in the subtree. This means that find-in-page is able to expand an
otherwise collapsed section when it finds a match.

Another intended use-case is a developer-managed virtual scroller.

## Alternatives considered

The `display: none` CSS property causes content subtrees not to render. However,
there is no mechanism for user-agent features to cause these subtrees to render.
Additionally, the cost of hiding and showing content cannot be eliminated since
`display: none` does not preserve the layout state of the subtree.

`visibility: hidden` causes subtrees to not paint, but they still need style and
layout, as the subtree takes up layout space and descendants may be `visibility:
visible`. (It's also possible for descendants to override visibility, creating
another complication.) Second, there is no mechanism for user-agent features to cause
subtrees to render. Note that with sufficient containment and intersection
observer, the functionality provided by `subtree-visibility` may be mimicked with
some exceptions: find-in-page functionality does not work in unrendered content;
this relies on more browser heuristics to ensure contained invisible content is
cheap -- `subtree-visibility` is a stronger signal to the user-agent that work
should be skipped.

Similar to `visibility: hidden`, `contain: strict` allows the browser to
automatically detect subtrees that are definitely offscreen, and therefore that
don't need to be rendered. However, `contain: strict` is not flexible enough to
allow for responsive design layouts that grow elements to fit their content. To
work around this, content could be marked as `contain: strict` when offscreen
and then some other value when on-screen (this is similar to `subtree-visibility`).
Second, `contain: strict` may or may not result in rendering work, depending on
whether the browser detects the content is actually offscreen. Third, it does
not support user-agent features in cases when it is not actually rendered to the
user in the current application view.

<a name="foot-notes"></a>

[1]: Meaning, the [rendering
part](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md)
of the browser event loop.

[2]: Examples: placing `display:none` CSS on DOM subtrees, or by placing content
far offscreen via tricks like `margin-left: -10000px`

[3]: In this context, virtualization means representing content outside of the
DOM, and inserting it into the DOM only when visible. This is most commonly used
for virtual or infinite scrollers.

[4]: Examples: caching the computed style of DOM elements, the output of text /
block layout, and display list output of paint.

[5]: Examples: detecting elements that are clipped out by ancestors, or not
visible in the viewport, and avoiding some or most rendering [lifecycle
phases](https://github.com/WICG/display-locking/blob/master/lifecycle.md) for
such content.
