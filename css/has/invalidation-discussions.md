# `:has()` invalidation discussion status

## Common base of the discussion

### `:has()` prototyping status in chromium project

After the Chrome version 95, when you enable the `enable-experimental-web-platform-features`, you can use most of the `:has()` usages with the javascript APIs (`querySelector`, `querySelectorAll`, `closest`, `matches`) except some pseudos related with tree boundary crossing (`:host()`, `:host-context()`, `:slotted()`, `::part()`, ...).

In other words, we have `:has()` selector matching functionality for most cases except tree boundary crossing.

`:has()` matching is `O(n)` operation where `n` is the number of descendant elements. When there are multiple subject elements that share same descendants, `:has()` matching can be `O(n^2)` because of the repetitive `:has()` argument matching on the shared descendants.

By using a cache that stores the `:has()` matching status, the repetitive `:has()` argument matching problem can be solved.

But the cache is currently available within a single lifecycle (a single Javascript call, a single style recalculation lifecycle), so we still have the `O(n)` problem of `:has()` matching for every lifecycle.

### Now, we are trying to do `:has()` invalidation prototyping.

With the merged CLs, we can see that the `:has()` works in style rules. We can style ancestor element with the condition of a descendant. (e.g. `.ancestor:has(.descendant) { background-color: green }`). But this only works on loading time because `:has()` invalidation is not supported yet.

#### How do we think about the `:has()` invalidation?
To share the overview of `:has()` style invalidation, we will use following simplified case.

* Descendant have boolean state of clicked/not-clicked which toggles when it clicked.
* Descendant and its ancestor have different style for the descendant having the clicked state.

For the above case, without ':has', we need to add similar state to the ancestor and synchronize the ancestor state with the descendant state by javascript.

```html
<style>
    .checked { font-weight: bold }                      /* rule1 */
    .ancestor.has-checked { background-color: yellow }  /* rule2 */
</style>
<div class='ancestor'>
    <div><div><div id=descendant>click</div></div></div>
</div>
<script>
descendant.addEventListener('click', function() {
    // Change #descendant element state.
    // This leads to style invalidation for the #descendant element
    // due to the rule1.
    this.classList.toggle('checked');

    // Get ancestor element
    ancestor = this.closest('.ancestor');
    
    // Change .ancestor element state.
    // This leads to style invalidation for the .ancestor element
    // due to the rule2.
    ancestor.classList.toggle('has-checked');
});
</script>
```

Style invalidation perspective, the javascript logic for the ancestor element consists of two parts. One is finding the `.ancestor` element, and the other one is triggering style invalidation by updating state of the element.

We can use `:has()` to remove the javascript code that finds the `.ancestor` element and triggers style invalidation on the element.

```html
<style>
    .checked { font-weight: bold }                        /* rule1 */
    .ancestor:has(.checked) { background-color: yellow }  /* rule2 */
</style>
<div class='ancestor'>
    <div><div><div id=descendant>click</div></div></div>
</div>
<script>
descendant.addEventListener('click', function() {
    // Change #descendant element state.
    // This leads to style invalidation for the #descendant element
    // due to the rule1.
    this.classList.toggle('checked');
    //
    // This also leads to upward traversal from #descendant element
    // to find .ancestor element due to the rule2.
    // 
    // And when it finds the .ancestor element, the element will be 
    // invalidated due to the rule2.
});
</script>
```

This means that, what ':has' style invalidation need to do are, 1. finding `.ancestor` element, 2. triggering style invalidation on the `.ancestor` element.

So, in this case, 
1. when browser engine detect a chage of adding or removing the class value `checked` on an element,<br>
(<code>.ancestor:has( <b style='background-color: yellow'>.checked</b>)) {...}</code>)
2. it will travers to ancestors of the changed element to find elements with the class value `ancestor`,<br>
(<code><b style='background-color: yellow'>.ancestor</b>:has(<b style='background-color: yellow; color: yellow'>&nbsp;</b>.checked) {...}</code>)
3. and trigger style invalidation of the ancestor element as if the `:has(.checked)` state was changed on the ancestor element.<br>
(<code>.ancestor<b style='background-color: yellow'>:has( .checked)</b> {...}</code>)

If we assume we have a virtual pseudo-class that represents true when there is a descendant element with a class value of `checked` (something like `:has-descendant-class-checked`), then we can read the last step as:

3. and trigger style invalidation of the ancestor element as if the `:has-descendant-class-checked` pseudo state is changed.<br>
(<code>.ancestor<b style='background-color: yellow'>:has-descendant-class-checked</b> {…}</code>)

We can say that these steps are : *"Finding elements that are possibly affected by a mutation, and creating virtual pseudo state mutation event on the elements to trigger style invalidation"*

Existing browser engine already has a functionality of invalidating style of an element when a pseudo state is changed on it. So, what `:has` style invalidation need to do is, *"to traverse upward to find possible elements affected by the `:has()` state, and trigger invalidation on the element"*.

The time complexity of the step2 (finding `.ancestor`) will be `O(m)` where `m` is the tree depth of the changed element - it will traverse all ancestors of the changed element to find elements with the class value `ancestor`. (Please note that this is different with the `:has()` selector matching complexity which is `O(n)` where `n` is the number of descendants)

#### What is the possible limitations for `:has()` invalidation ?

We can generate infinite number of selector expressions by combining `:has()` with other selectors. Each of those introduces different types and amount of complexity and performance impact. Those also make it more difficult to discuss. Effectively, we need to discuss what might be necessary limitations on `:has()` style invalidation, and where things bigin to get into uncomfortable levels of complexity and performance impact.

Listing the criteria by which `:has()` expressions can be distinguished can help extract and clarify limitations.


|    Criteria   | Example |
| :----------- | :----- |
| Allow `:has()` argument starts with `>` ? | `.hero:has(> img)` |
| Allow `:has()` argument starts with descendant combinator ? | `.hero:has(img)` |
| Allow attribute/elemental selectors in `:has()` ?  | `.hero:has(img)`<br>`.hero:has(.splash)`<br>`.hero:has([alt])`<br>`.hero:has(#main-feature)`<br> |
| Allow compound selector in `:has()`?  | `.product-card:has(.shirt.sale[active])` |
| Allow selector list in `:has()`?  | `.product-card:has(.shirt, .pants)` |
| Allow `:has()` argument starts with `~` or `+` ? | `.hero:has(+ section)`<br>`.hero:has(~ section)` |
| Allow complex selector in `:has()` ? | `.product:has(.recommended > .adapter)`<br>`section:has(figure.schematic figcaption)` |
| Allow non-terminal `:has()` ? | `.product-card:has(.shirt.sale[active]) .button` |
| Allow `:has()` in logical combinations ? | `:not(:has(foo))`<br>`:is(:has(foo), :has(bar))` |
| Allow pseudo elements in `:has()` ? | `:has(::first-letter)` |
| Allow logical combinations in `:has()` ? | `:has(:not(foo))`<br>`:has(:where(foo, bar))`<br>`:has(.a:has(.b))` |
| Allow linguistic pseudo-classes in `:has()` ? | `:has(:dir(rtl))`<br>`:has(:lang(fr-be))` |
| Allow location pseudo-classes in `:has()` ? | `:has(:any-link)`<br>`:has(:visited)`<br> `:has(:target)`<br>`:has(:target-within)`<br>...|
| Allow user action pseudo-classes in `:has()` ? | `:has(:hover)`<br>`:has(:active)`<br>`:has(:focus)`<br>...|
| Allow time-dimensional pseudo-classes in `:has()` ? | ... |
| Allow resource state pseudos in `:has()` ? | ... |
| Allow input pseudo-classess in `:has()` ? | ... |
| Allow tree structural pseudos in `:has()` ? | ... |

#### What are the basic and important use cases to start?

Listing all the possible limitations and examining the complexity and performance impact of all limitations is too difficult and time consuming. Actually it looks impossible and inefficient to get all those and discuss with those.

Considering *"What will be the basic and important use cases?"* will help us choose the limitations to start with.

We have these three use cases from the previous discussions. (Shared by [Rune](https://twitter.com/runeli), originally from [Una](https://twitter.com/una))


1. **Interactions** -- Hovering a button within a card makes the card style change (i.e. background color, border, or even interaction like having a shine animation gloss over it). Sure you can solve this when a single card has a single action, but what happens when that card has multiple actions? You need JS for that.
2. **States** -- Styling things like containers with certain states (i.e. "empty" or "pending") is either not consistent (:empty/:not:empty due to elements vs text nodes and whitespace), or not possible in CSS. Say you want an empty state experience and in your JS you populate an Inbox with a cute message that says "Yay you've reached Inbox 0!". You may want to style the page a certain way, detecting when the child class is present. I.e. if .empty-message is present, it's parent/ancestor .inbox-container could get a style, like a sunshine background. JS is currently required for that.
3. **Pairings** -- Say you have a shopping card. The shopping card has a "Buy" button. The buy button may be :disabled when the item is sold out. It would be nice to be able to style either the parent card or any parent element with the "Sold Out" style. JS is currently required for that. (I guess this is also a state example)

It looks that lots of `:has()` usage would be similar with these cases. (Styling parent or ancestor element by its descendant condition)

#### Given the use cases, what limitations would it make sense to start with?

All the three use cases above tries to style parent or ancestor element by the condition of its descendant element. So, allowing these cases would be enough for supporting those cases.
* Allow `:has()` argument starts with `>`
* Allow `:has()` argument starts with descendant combinator
* Allow attribute/elemental selectors in `:has()`
* Allow compound selector in `:has()`

When we have any progress with the above limitations, we can extend it to the pseudo classes mentioned at the use cases (`:hover`, `:disabled`). So, using the pseudos in `:has()` will be limited for now.

For the start, we will discuss the `:has()` invalidation with these limitation.
* Not allow selector list in `:has()`
* Not allow `:has()` argument starts with `~` or `+`
* Not allow complex selector in `:has()`
* Not allow non-terminal `:has()`
* Not allow `:has()` in logical combinations
* Not allow all pseudos in `:has()`

Please note that, these limitations are not necessarily proposals for where we end up, it's just about how we simplify and grow the discussion.

## Updates from the discussions

I have some questions.
* Does the above summary make sense to all? Are there any missing or arguable points that blocks discussion?
* More specifically about `:has()` invalidation overview,
  * is it acceptable to add a step that traverses ancestors to find possibly affected element?<br>
    (The traversal will be `O(m)` where `m` is tree depth of the changed element)
  * does it make sense to extract upward traversal filtering condition from the selector in a style rule?<br>
    (From `.product:has(.shirt)`, we can get filtering condition of class value `product` for the mutation of changing the class value `shirt`)
  * does it make sense to trigger style invalidation of the element that matches the upward traversal filtering condition?
