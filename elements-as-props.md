# Deep Dive into Passing Elements as Props: A Misunderstood React Optimization Pattern

One of the most common re-render optimizations in React is the **Element as Props** pattern. If a *component*'s re-render triggers a re-render of an expensive child *component* that doesn't consume any *state* or *props* from the parent *component*, then that child *component*'s *element* instance can be passed as a prop to the parent *component*. The parent *component* can then inject that *element* at an appropriate *slot* in its own render result.

## Example

**From**: `ExpensiveComponent` re-renders every time `ParentComponent` re-renders.

```jsx
function ParentComponent() {
  const [state, setState] = useState(<some_value>);

  function handleClick() {
    setState(<some_value>)
  }

  return (
    <div>
      <Button onClick={handleClick} />
      <ExpensiveComponent />
      <SomeOtherComponent state={state}>
    </div>
  );
}
```

**To**: `ExpensiveComponent` doesn't re-render due to `ParentComponent`s own state updates.

```jsx
function GrandParent() {
  return (
    <div>
      <ParentComponent slot={<ExpensiveComponent />} />
      <AnotherComponent />
    </div>
  )
}

function ParentComponent({slot}) {
  const [state, setState] = useState(<some_value>)

  function handleClick() {
    setState(<some_value>)
  }

  return (
    <div>
      <Button onClick={handleClick} />
      {slot}
      <SomeOtherComponent state={state}>
    </div>
  ) 
}
```

It is believed that the reason this pattern works is because of the *stable react element reference* of the `slot` *element* accepted as a prop. When the `ParentComponent` re-renders due to its own state update, the `slot` *element* will be *referentially stable* and that should allow React to safely skip re-rendering the `ExpensiveComponent`. Right? Sounds good, but *element referential equality* has nothing to do with React attempting a render bailout - not directly atleast.

## Proof by contradiction

### Assumption

If an *element reference* remains *referentially stable* between re-renders, React skips re-rendering the *component* associated with that *element*.

### Contradicting examples

#### Example 1

- Memoize the `ExpensiveComponent`'s *element* reference so that it is forced to remain stable between re-renders.
- Create a new *props* object in every render and inject it into the stable *element* reference.

[open stackblitz sandbox](https://stackblitz.com/edit/stable-element-ref-rerender?file=src%2FApp.tsx)

```jsx
const element = <ExpensiveComponent />;

function ParentComponent() {
  const [count, setCount] = useState(0);

  /*
   force a stable element ref but inject a new props object between re-renders
   */
  const slot = useMemo(() => ({ ...element }), []);
  slot.props = {};

  return (
    <div>
      Count: {count}
      <button onClick={() => setCount(count + 1)}>Increment</button>
      {slot}
    </div>
  );
}

function ExpensiveComponent() {
  alert('RENDERING EXPENSIVE COMPONENT');
  return <p>Very expensive component</p>;
}
```

The alert from `ExpensiveComponent` will appear every single time `ParentComponent` re-renders. This happens despite the fact that the `slot` *element* is forced to be *referentially stable* between re-renders. This observation is thus a contradiction to our original assumtion.

#### Example 2

This time, let's create a new *element* reference in every render, but memoize the *element*'s *props* object.

[open stackblitz sandbox](https://stackblitz.com/edit/stable-props-unstable-element?file=src%2FApp.tsx)

```js
function ParentComponent() {
  const [count, setCount] = useState(0);

  // new element ref every render but stable props ref
  const slot = { ...<ExpensiveComponent /> };
  const propsMemo = useMemo(() => ({}), []);
  slot.props = propsMemo;

  return (
    <div>
      Count: {count}
      <button onClick={() => setCount(count + 1)}>Increment</button>
      {slot}
    </div>
  );
}

function ExpensiveComponent() {
  alert('RENDERING EXPENSIVE COMPONENT');
  return <p>Very expensive component</p>;
}
```

The alert from `ExpensiveComponent` doesn't appear now when `ParentComponent` re-renders, despite the fact that the `slot`'s *element* reference is forced to be different between re-renders. Once again, a contradiction to our original assumption.

#### What's going on?

##### Case 1

Same *element* reference but different *props* reference => Component is re-rendered.

##### Case 2

Different *element* reference but same props reference => Component is not re-rendered.

It is clear that what actually matters is the referential equality of the *props* object, not that of the *element* reference itself. And that makes absolute sense - When the inputs to a *component* (*props* in this case) remain the same (*same* means *referentially stable* for objects), that *component*'s render result should be the same, and therefore, it should be safe to skip re-rendering it.

If we look at the source code of React,
[the props referential equality check is the first thing that React does to evaluate whether a component should be re-rendered or not](https://github.com/facebook/react/blob/ea6e05912aa43a0bbfbee381752caa1817a41a86/packages/react-reconciler/src/ReactFiberBeginWork.js#L3856-L3857).

### Why the optimisation pattern works

Now that we've established that a stable *props* reference between re-renders (and not the *element* reference) opens the doors to a render bail out, the reason why the optimisation pattern works will be straight-forward.

Back to the original example:

```jsx
function GrandParent() {
  return (
    <div>
      <ParentComponent slot={<ExpensiveComponent />} />
      <AnotherComponent />
    </div>
  )
}

function ParentComponent({slot}) {
  const [state, setState] = useState(<some_value>)

  function handleClick() {
    setState(<some_value>)
  }

  return (
    <div>
      <Button onClick={handleClick} />
      {slot}
      <SomeOtherComponent state={state}>
    </div>
  ) 
}
```

When the `ParentComponent` re-renders due to its own state update, the *props* reference of the `slot` *element* will be *referentially stable* and that should allow React to attempt a render bail out!
