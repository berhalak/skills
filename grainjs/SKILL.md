---
name: grainjs
description: Use when building UI with grainjs — creating DOM elements, using Observables/Computed, conditional rendering with dom.maybe/dom.domComputed, styled components, class bindings, event handling, disposal, MultiHolder, obsArray, or any reactive UI pattern using grainjs.
---

# GrainJS Reference

GrainJS is a TypeScript library for building reactive UIs with observables and direct DOM construction. No virtual DOM, no JSX — just functions that create and bind real DOM elements.

---

## DOM Construction

The `dom()` function creates elements. Arguments can be child nodes, strings (text nodes), attribute objects, arrays (flattened), functions (called with element), or dom methods.

```typescript
import { dom, styled, Observable, Computed, Disposable, MultiHolder } from 'grainjs';

dom('a', { href: 'https://example.com' },
  dom.cls('biglink'),
  'Hello ', dom('span', 'world')
);
// <a href="https://example.com" class="biglink">Hello <span>world</span></a>
```

`dom.update(existingElement, ...args)` adds children/bindings to an existing element (e.g. `document.body`).

---

## Observables

Create reactive state containers. First arg is the owner (for automatic disposal), second is the initial value.

```typescript
const show = Observable.create(owner, false);
show.get();        // false
show.set(true);    // updates value, notifies listeners

// Listen to changes
const listener = show.addListener(val => console.log(val));
listener.dispose(); // stop listening
```

### Observable as owner of disposable values

Observables can own disposable objects — when the observable value changes or the observable is disposed, the old value is disposed too:

```typescript
const obs = Observable.create<MyClass|null>(owner, null);
MyClass.create(obs, ...args);  // obs owns this instance
obs.set(null);                 // previous MyClass instance is disposed
```

---

## Computed Observables

Derived values that auto-update when dependencies change. The `use()` function reads an observable's value AND registers it as a dependency.

```typescript
const obs1 = Observable.create(owner, 5);
const obs2 = Observable.create(owner, 12);

// Dynamic dependencies via use()
const sum = Computed.create(owner, use => use(obs1) + use(obs2));

// Explicit dependencies (slightly more efficient)
const sum2 = Computed.create(owner, obs1, obs2,
  (use, v1, v2) => v1 + v2);
```

### Evaluation Order

GrainJS recalculates computeds in dependency order. If `total` depends on `tax` and `tip`, which both depend on `amount`, changing `amount` recalculates `tax`, then `tip`, then `total`.

### onWrite — writable computed

By default, calling `set()` on a computed throws an error. Use `.onWrite()` to make it writable — the callback is invoked whenever `set()` is called:

```typescript
const bothCheck = Computed.create(owner, obsCheck1, obsCheck2, (_use, c1, c2) => {
  if (c1 && c2) return true;
  if (c1 || c2) return 'indeterminate';
  return false;
}).onWrite((val) => {
  if (val === 'indeterminate') return;
  obsCheck1.set(val);
  obsCheck2.set(val);
});

// Now you can write to it:
bothCheck.set(true);  // calls onWrite, which updates obsCheck1 and obsCheck2
```

Returns the computed instance for chaining.

---

### bundleChanges

Defer computed recalculation until multiple observables are updated:

```typescript
import { bundleChanges } from 'grainjs';

bundleChanges(() => {
  taxRate.set(0.0875);
  tipRate.set(0.20);
});
// Computeds recalculate only once after both changes
```

### subscribe()

Subscribe to multiple observables. Callback fires immediately and on every change:

```typescript
subscribe(obs1, obs2, (use, v1, v2) => console.log(v1, v2));
// or dynamically:
subscribe(use => console.log(use(obs1), use(obs2)));
```

---

## DOM Bindings with Observables

### Dynamic attributes, classes, text

```typescript
const isBig = Observable.create(null, true);
const href = Observable.create(null, 'https://example.com');
const name = Observable.create(null, 'world');

dom('a',
  dom.cls('biglink', isBig),           // class toggled by observable
  dom.attr('href', href),              // attribute bound to observable
  'Hello ', dom('span', dom.text(name)) // text bound to observable
);

isBig.set(false);       // removes 'biglink' class
href.set('about:blank'); // changes href
name.set('Bart');        // changes text
```

### Computed callbacks in bindings

Pass a `use =>` callback anywhere an observable is accepted — it auto-creates a Computed:

```typescript
dom('a',
  dom.cls('small-link', use => !use(isBig)),
);
```

### dom.hide()

```typescript
dom('div', dom.hide(isHiddenObs));
```

---

## Class Bindings

### dom.cls()

Toggle a single CSS class based on an observable boolean:

```typescript
dom.cls('active', isActiveObs)        // adds/removes 'active'
dom.cls('highlight', use => use(a) > 5) // computed condition
```

### Styled component .cls() modifier

For styled components, use the `.cls('-modifier')` pattern:

```typescript
const cssButton = styled('button', `
  border-radius: 0.5rem;
  font-size: 1rem;
  &-small { font-size: 0.6rem; }
  &-primary { background: blue; color: white; }
`);

// Apply modifier class
cssButton(cssButton.cls('-small'), 'Small Button');

// Toggle modifier with observable
cssButton(cssButton.cls('-primary', isPrimaryObs), 'Button');
```

---

## Conditional Rendering: dom.maybe()

Renders content when observable is truthy, removes it when falsy. Content is fully disposed and rebuilt on each toggle.

```typescript
dom('div',
  dom.maybe(isChangedObs, () =>
    dom('button', 'Save')
  )
);
```

Works with `dom.create()` for class components:

```typescript
dom.maybe(show, () => dom.create(TempCalculator));
```

**Key behavior**: The callback is called each time the value becomes truthy. When it becomes falsy, the DOM is removed and disposed.

---

## Dynamic DOM: dom.domComputed()

Switches between different DOM structures based on an observable value. The previous DOM is disposed before the new one is created.

```typescript
dom('div',
  dom.domComputed(isChangedObs, (isChanged) =>
    isChanged
      ? [dom('button', 'Save'), dom('button', 'Revert')]
      : dom('button', 'Close')
  )
);
```

The callback receives the plain value (not the observable) and can return a single element, an array of elements, or null.

**dom.maybe vs dom.domComputed**:
- `dom.maybe(obs, () => content)` — show/hide based on truthiness
- `dom.domComputed(obs, val => content)` — rebuild DOM whenever value changes, receiving the actual value

---

## Lists: dom.forEach()

### Static arrays

```typescript
const items = ['Apples', 'Pears', 'Peaches'];
dom('ul', items.map(item => dom('li', item)));
```

### Observable arrays

```typescript
const items = Observable.create(null, ['Apples', 'Pears']);
dom('ul',
  dom.forEach(items, item => dom('li', item))
);
items.set(['Bananas']); // removes old, inserts new
```

### obsArray for incremental changes

```typescript
import { obsArray } from 'grainjs';

const items = obsArray(['Apples', 'Pears']);
dom('ul',
  dom.forEach(items, item => dom('li', item))
);
items.push('Peaches');           // appends one <li>
items.splice(1, 1, 'Bananas');   // replaces Pears with Bananas
```

The per-item callback must return a single DOM element or null (not an array).

---

## Styled Components

`styled()` generates unique CSS class names and injects styles into the document. Call at module top level (import time).

```typescript
const cssTitle = styled('h1', `
  font-size: 1.5em;
  text-align: center;
  color: palevioletred;
`);

const cssWrapper = styled('section', `
  padding: 0.5em 4em;
  background: papayawhip;
`);

dom.update(document.body,
  cssWrapper(cssTitle('Hello world'))
);
```

### Extending styles

```typescript
const cssTitle2 = styled(cssTitle, `font-size: 1rem; color: red;`);
```

### Nested selectors with &

The `&` represents the generated class name. Nested styles MUST appear after main styles:

```typescript
const cssButton = styled('button', `
  border-radius: 0.5rem;
  border: 1px solid grey;
  font-size: 1rem;

  &:active {
    background: lightblue;
  }
  &-small {
    font-size: 0.6rem;
  }
  @media print {
    & { display: none; }
  }
`);
```

### .cls() helper for modifier classes

```typescript
cssButton(cssButton.cls('-small'), 'Test');
cssButton(cssButton.cls('-small', isSmallObs), 'Test'); // observable toggle
```

### keyframes

```typescript
import { keyframes } from 'grainjs';

const rotate360 = keyframes(`
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
`);

const cssSpinner = styled('div', `
  animation: ${rotate360} 2s linear infinite;
`);
```

### Convention

Prefix styled component variables with `css` (e.g., `cssButton`, `cssWrapper`). Place them at module bottom.

---

## Disposal & Ownership

### Disposable base class

```typescript
class MyChart extends Disposable {
  constructor() {
    const onResize = () => this._updateChartSize();
    window.addEventListener('resize', onResize);
    this.onDispose(() => window.removeEventListener('resize', onResize));
  }
}
```

### .create() pattern (always preferred over new)

```typescript
const chart = MyChart.create(owner, ...args);
```

The owner disposes the object automatically. `.create()` also handles constructor exceptions safely.

### Holder — single replaceable slot

```typescript
this._holder = Holder.create(this);
Bar.create(this._holder, 1);  // held
Bar.create(this._holder, 2);  // disposes previous Bar, holds new one
this._holder.clear();         // disposes contained object
```

### MultiHolder — multiple owned objects

```typescript
this._mholder = MultiHolder.create(this);
Bar.create(this._mholder, 1); // added
Bar.create(this._mholder, 2); // added (both kept)
// all disposed when _mholder is disposed
```

Use MultiHolder when you need to own multiple disposable objects in a container that isn't itself a Disposable class.

### DOM disposal

```typescript
dom('div',
  dom.autoDispose(someComputed),   // disposed when element is removed
  dom.onDispose(() => cleanup()),  // callback when element is removed
);
```

`dom.maybe()`, `dom.domComputed()`, and `dom.forEach()` automatically dispose removed DOM and associated resources.

---

## DOM Components

### Class components

Extend `Disposable`, implement `buildDom()`. Use `dom.create()` to instantiate:

```typescript
class TempCalculator extends Disposable {
  private _celsius = Observable.create(this, 25);
  private _fahrenheit = Computed.create(this, use => (use(this._celsius) * 9 / 5) + 32);

  public buildDom() {
    return dom('div',
      dom('input', { type: 'text', value: '25' },
        dom.on('input', (ev, elem) => this._celsius.set(parseFloat(elem.value)))
      ),
      dom('p', dom.text(use => `${use(this._fahrenheit)}F`)),
    );
  }
}

dom.update(document.body, dom.create(TempCalculator));
```

### Functional components

Receive an `owner` as first param:

```typescript
function tempCalculator(owner: MultiHolder, initialValue: number) {
  const celsius = Observable.create(owner, initialValue);
  const fahrenheit = Computed.create(owner, use => (use(celsius) * 9 / 5) + 32);

  return dom('div',
    dom('input', { type: 'text', value: String(initialValue) },
      dom.on('input', (ev, elem) => celsius.set(parseFloat(elem.value)))
    ),
    dom('p', dom.text(use => `${use(fahrenheit)}F`)),
  );
}

dom.update(document.body, dom.create(tempCalculator, 30));
```

---

## Event Handling

```typescript
dom('button', 'Click',
  dom.on('click', (event, elem) => { ... }),
  dom.on('focus', (event, elem) => { ... }),
);
```

### Keyboard events

```typescript
dom('input', { type: 'text' },
  dom.onKeyDown({
    Enter: (ev, elem) => submit(),
    Escape: (ev, elem) => cancel(),
    ArrowLeft$: (ev, elem) => { ... }, // $ suffix = don't stop propagation
  })
);
```

### Delegated events

```typescript
dom.onMatch('.some-selector', 'click', (event, elem) => { ... });
```

---

## Event Emitters

Separate Emitter instances per event type (no string-based event names):

```typescript
const cartItemAdded = new Emitter();
const cartItemRemoved = new Emitter();

cartItemAdded.addListener(item => console.log('Added:', item));
cartItemAdded.emit(newItem);
```

---

## Quick Patterns

| Task | Pattern |
|------|---------|
| Create observable | `Observable.create(owner, initialValue)` |
| Create computed | `Computed.create(owner, use => use(obs1) + use(obs2))` |
| Toggle class | `dom.cls('name', boolObs)` |
| Bind text | `dom.text(obs)` or `dom.text(use => ...)` |
| Bind attribute | `dom.attr('name', obs)` |
| Show/hide content | `dom.maybe(obs, () => dom(...))` |
| Switch content | `dom.domComputed(obs, val => dom(...))` |
| Repeat items | `dom.forEach(obsArray, item => dom(...))` |
| Style component | `const cssX = styled('div', \`...\`)` |
| Modifier class | `cssX(cssX.cls('-mod'), ...)` |
| Extend style | `styled(cssBase, \`...\`)` |
| Class component | `class X extends Disposable { buildDom() {} }` |
| Use component | `dom.create(ComponentClass, ...args)` |
| Own multiple | `MultiHolder.create(owner)` |
| Replaceable slot | `Holder.create(owner)` |
| Dispose on detach | `dom.autoDispose(disposable)` |
