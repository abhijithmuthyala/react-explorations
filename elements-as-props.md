# Deep Dive into Passing Elements as Props: A Misunderstood React Optimization Pattern

One of the most common re-render optimizations in React is the **Elements as Props** pattern. If a component's re-render triggers a re-render of an expensive child component that doesn't consume any **state** or **props** from the parent component, then that child component's render result can instead be passed as a prop to the parent component. The parent component can then **inject** that render result at an appropriate **slot** in its own render result.

## Example

**From**: `ExpensiveComponent` re-renders every time `ParentComponent` re-renders.

```jsx
function ParentComponent() {
  const [state, setState] = useState("editing");

  return (
    <div>
      <Button state={state} />
      <ExpensiveComponent />
    </div>
  );
}
```

*to*: `ExpensiveComponent` doesn't re-render due to `ParentComponent`s own state updates.

```js
function GrandParent() {
  return (
    <div>
      <ParentComponent slot={<ExpensiveComponent />} />
    </div>
  )
}

function ParentComponent({slot}) {
  const [state, setState] = useState("editing")

  return (
    <div>
      <Button state={state} />
      {slot}
    </div>
  ) 
}
```

It is believed that the reason this pattern works is because of the *stable react element reference* of the *slot* accepted as a prop. When the `ParentComponent` re-renders due to its own state update, the `slot` prop will be *referentially stable* and that should allow *React* to safely skip re-rendering the `ExpensiveComponent`. Right? Sounds good, but I'm here to prove this wrong.

## Proof by contradiction

### Assumption

React skips re-rendering a component if its associated *element reference* remains stable between re-renders.

### Contradicting examples

#### Example 1

- Memoize the expensive component's element reference so that it is forced to remain stable between re-renders.
- Create a new *props* object in every render and inject it into the stable element reference.

```js
const expensiveRenderResult = <ExpensiveComponent />;

function ParentComponent() {
  const [count, setCount] = useState(0);

  /*
   force a stable element ref but inject a new props object between re-renders
   */
  const slot = useMemo(() => ({ ...expensiveRenderResult }), []);
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

The alert from `ExpensiveComponent` will appear every single time `ParentComponent` re-renders. This happens despite the fact that the `slot` element is *referentially stable* between re-renders and is thus a direct contradiction to our original assumtion.

#### Example 2

This time, let's create a new element reference in every render, but memoize the element's props object.

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

The alert from `ExpensiveComponent` doesn't appear now when `ParentComponent` re-renders, despite the fact that the `slot` element reference is different between re-renders. Once again a direct contradiction to our original assumption.

#### What's going on?

##### Case 1

Same element ref and different props ref => Component is re-rendered.

##### Case 2

Different element ref and same props ref => Component is not re-rendered.

It is clear that what actually matters is the referential equality of the `props` object, not the `element` reference itself. And that makes absolute sense - When the inputs to the component (props in this case) doesn't change, the output (element) should be the same, so it should be safe to skip re-rendering.

If we look at the source code of React,
[the props referential equality check is the first thing that React does to evaluate whether a component should be re-rendered or not](https://github.com/facebook/react/blob/ea6e05912aa43a0bbfbee381752caa1817a41a86/packages/react-reconciler/src/ReactFiberBeginWork.js#L3856-L3857).

### Why the optimisation pattern works

Now that we've established that a stable props reference between re-renders (and not the *element* ref) opens the doors to a render bail out, the reason why the optimisation pattern works will be straight-forward.

Back to the original example:

```js
function GrandParent() {
  return (
    <div>
      <ParentComponent slot={<ExpensiveComponent />} />
    </div>
  )
}

function ParentComponent({slot}) {
  const [state, setState] = useState("editing")

  return (
    <div>
      <Button state={state} />
      {slot}
    </div>
  ) 
}
```

When the `ParentComponent` re-renders due to its own state update, the props reference of the `slot` element will be *referentially stable* and that should allow *React* to attempt a render bail out!
