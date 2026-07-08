# React Cheat Sheet
### Vue 3 Developer — Interview Prep Reference

---

## 1. The Immutability Rule

React detects state changes by **reference equality** — it checks whether something is a different object in memory, not whether the contents changed. You must always produce a **new value** — never mutate what's already in state.

Vue handles this automatically via its reactivity system. In React, the responsibility is entirely yours.

```tsx
// ❌ Always wrong — mutating existing state
state.name = 'Greg';
items.push(newItem);
items[0] = 'updated';

// ✅ Always right — producing something new
setState(prev => ({ ...prev, name: 'Greg' }));
setItems(prev => [...prev, newItem]);
setItems(prev => prev.map((item, i) => i === 0 ? 'updated' : item));
```

---

## 2. Common Array Operations in State

```tsx
const [items, setItems] = useState<string[]>([]);

// Add to end
setItems(prev => [...prev, newItem]);

// Add to beginning
setItems(prev => [newItem, ...prev]);

// Remove by index
setItems(prev => prev.filter((_, i) => i !== indexToRemove));

// Remove by value
setItems(prev => prev.filter(item => item !== valueToRemove));

// Update item at index
setItems(prev => prev.map((item, i) => i === indexToUpdate ? newValue : item));

// Replace entire array
setItems(newArray);

// Clear
setItems([]);
```

---

## 3. Updating Objects in State

```tsx
const [user, setUser] = useState({ name: 'Greg', role: 'engineer' });

// ❌ Wrong — same reference
user.name = 'Gregory';
setUser(user);

// ✅ Correct — spread to produce new object
setUser(prev => ({ ...prev, name: 'Gregory' }));
```

**Nested objects — spread at every level you're changing:**

```tsx
const [user, setUser] = useState({
  name: 'Greg',
  address: { city: 'Philadelphia', state: 'PA' }
});

// ❌ Wrong — address is still the same reference
setUser(prev => ({ ...prev, address: { city: 'Wayne' } }));

// ✅ Correct — spread nested object too
setUser(prev => ({
  ...prev,
  address: {
    ...prev.address,
    city: 'Wayne'
  }
}));
```

---

## 4. The Functional Updater Form

Always use the functional updater form `prev =>` when the new value depends on the old value. This guarantees React provides the current value regardless of closures.

```tsx
// ❌ Risky — can read stale value inside closures
setCount(count + 1);
setItems([...items, newItem]);

// ✅ Safe — React always provides current value
setCount(prev => prev + 1);
setItems(prev => [...prev, newItem]);
setIsOpen(prev => !prev); // toggling booleans
```

---

## 5. useState vs useRef

| | `useState` | `useRef` |
|---|---|---|
| Triggers re-render | ✅ Yes | ❌ No |
| Use when | Value drives the UI | DOM access / internal plumbing |
| How you read it | Direct variable | `.current` |
| Vue equivalent | `ref()` used in template | Plain variable in `<script setup>` |

```tsx
// useState — drives the UI
const [isOpen, setIsOpen] = useState(false);

// useRef — DOM access, never triggers re-render
const menuRef = useRef<HTMLElement>(null);
menuRef.current.style.height = '100px';
```

**Note:** Vue's `ref()` is reactive and drives the UI. React's `useRef` does NOT trigger re-renders — they share a name but serve different purposes.

---

## 6. useEffect Dependency Array Patterns

```tsx
// Runs once on mount only
useEffect(() => { ... }, []);

// Runs on mount + whenever value1 or value2 change
useEffect(() => { ... }, [value1, value2]);

// Runs after every render — almost never what you want
useEffect(() => { ... });
```

**Always clean up long-lived side effects:**

```tsx
useEffect(() => {
  const interval = setInterval(() => { ... }, 4000);
  const handleResize = () => { ... };
  window.addEventListener('resize', handleResize);

  return () => {
    clearInterval(interval);
    window.removeEventListener('resize', handleResize);
  };
}, []);
```

**removeEventListener requires the same function reference:**

```tsx
// ❌ Wrong — two different references, listener never removed
window.addEventListener('resize', () => doSomething());
return () => window.removeEventListener('resize', () => doSomething());

// ✅ Correct — same reference
const handleResize = () => doSomething();
window.addEventListener('resize', handleResize);
return () => window.removeEventListener('resize', handleResize);
```

---

## 7. Stale Closures

A stale closure occurs when a function captures a state value at render time and that value becomes outdated before the function executes.

**Most common in:** `setInterval`, `addEventListener`, `useEffect` with `[]`

```tsx
// ❌ activeSlide inside the interval is always 0 — stale closure
useEffect(() => {
  setInterval(() => {
    console.log(activeSlide); // always reads initial value
  }, 4000);
}, []);

// ✅ Option 1 — add to dependency array (effect re-runs on change)
useEffect(() => {
  const interval = setInterval(() => {
    console.log(activeSlide); // fresh value
  }, 4000);
  return () => clearInterval(interval);
}, [activeSlide]);

// ✅ Option 2 — use a ref to hold current value (interval stays alive)
const activeSlideRef = useRef(0);

const toggleActive = (i: number) => {
  setActiveSlide(i);
  activeSlideRef.current = i; // keep ref in sync
};

useEffect(() => {
  const interval = setInterval(() => {
    console.log(activeSlideRef.current); // always current
  }, 4000);
  return () => clearInterval(interval);
}, []); // empty — interval created once, ref provides fresh values
```

---

## 8. useState is Asynchronous

State setters do not update the value immediately. The value remains the old one for the remainder of the current function call.

```tsx
// ❌ Wrong — drawerActive is still the OLD value after setDrawerActive
const toggleDrawer = () => {
  setDrawerActive(!drawerActive);
  if (drawerActive) { ... } // reads pre-toggle value — logic is inverted
};

// ✅ Correct — compute next value first, use it consistently
const toggleDrawer = () => {
  const next = !drawerActive;
  setDrawerActive(next);
  if (next) { ... } // reliable
};
```

**Vue contrast:** Vue's reactivity is synchronous at the point of mutation — `ref.value = newValue` is immediately readable. React batches updates for performance, making the new value unavailable until the next render.

---

## 9. Conditional JSX Patterns

```tsx
// Two outcomes — ternary
{isLoggedIn ? <Dashboard /> : <LoginPage />}

// One outcome or nothing — && short circuit
{hasError && <ErrorMessage />}

// ⚠️ Gotcha — if left side is a number, React renders it
{items.length && <List />}     // renders "0" if empty
{items.length > 0 && <List />} // correct

// Completely different component states — early return
if (isLoading) return <Spinner />;
if (hasError) return <ErrorPage />;
return <MainContent />;
```

---

## 10. Event Handlers in JSX

```tsx
// ❌ Calls immediately on render — infinite loop
onClick={toggleActive(i)}

// ✅ Arrow function wrapper — calls only on click
onClick={() => toggleActive(i)}

// ✅ No args needed — pass reference directly
onClick={toggleActive}
onClick={handleClick}
```

**Always use arrow function wrapper when passing arguments.**
**Pass reference directly only when no arguments are needed.**

---

## 11. Button Type Default

`<button>` defaults to `type="submit"` inside a `<form>`. Always be explicit:

```tsx
<button type="submit">Submit</button>
<button type="button" onClick={cancel}>Cancel</button>
```

---

## 12. Vue → React Quick Reference

| Vue 3 | React | Notes |
|---|---|---|
| `ref()` / `reactive()` | `useState()` | Vue's is automatic, React's is explicit |
| `computed()` | `useMemo()` | Or just a plain derived variable |
| `watch()` / `watchEffect()` | `useEffect()` | Different mental model — see below |
| `onMounted` | `useEffect(() => {}, [])` | Empty array = runs once on mount |
| `onUnmounted` | `useEffect` return function | Cleanup return |
| `provide` / `inject` | `useContext` | |
| Composables | Custom hooks | |
| `<slot />` | `{children}` | |
| `v-if` | Ternary or `&&` | |
| `v-for` | `.map()` with `key` | |
| `v-model` | `value` + `onChange` | Manual in React |
| `@click` | `onClick` | |
| `class` | `className` | Reserved word in JS |
| `for` on labels | `htmlFor` | Reserved word in JS |
| `:disabled="true"` | `disabled` or `disabled={true}` | |
| `NuxtLink` | `<Link>` from `next/link` | |
| `useRoute()` | `useParams()` from `next/navigation` | App Router only |
| `defineEmits` | Callback props | No event system in React |
| Pinia / Vuex | Redux / Zustand | |
| `nuxt.config css[]` | Import in `layout.tsx` | |
| Auto-imported components | Manual imports — always | No magic in React |

---

## 13. useEffect vs Vue's watch — Mental Model

These look similar but are fundamentally different:

- **Vue's `watch`** reacts to *data changes*
- **React's `useEffect`** *synchronises* with something outside React

```tsx
// useEffect is not a lifecycle hook — it's a synchronisation tool
// Think: "keep this external thing in sync with this state"

useEffect(() => {
  document.title = `${count} items`; // sync document title with count
}, [count]);

useEffect(() => {
  const subscription = api.subscribe(userId); // sync external subscription
  return () => subscription.unsubscribe();
}, [userId]);
```

**You Might Not Need an Effect** — React's own docs page on this is essential reading. The most common mistake Vue developers make is overusing `useEffect` the way they'd use `watch`.

---

## 14. Custom Hooks

A custom hook is a function that starts with `use` and composes existing React hooks. Extract repeated hook logic into a custom hook when the same pattern appears in multiple components or multiple times in one component.

**Rules:**
- Must start with `use` — React enforces hook rules on anything prefixed with `use`
- Can call other hooks internally — that's all a custom hook is
- Returns whatever the consuming component needs — values, setters, callbacks, refs

**Example — `useCounter`:**

A hook that manages a counter with convenience methods, eliminating repetitive setter logic:

```tsx
import { useState } from 'react';

interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setCount: React.Dispatch<React.SetStateAction<number>>;
}

export default function useCounter(initialValue: number = 0): UseCounterReturn {
  const [count, setCount] = useState(initialValue);

  return {
    count,
    increment: () => setCount(prev => prev + 1),
    decrement: () => setCount(prev => prev - 1),
    reset: () => setCount(initialValue),
    setCount,
  };
}

// Usage
const { count, increment, decrement, reset } = useCounter(0);
```

**Example — `useSyncedRef`:**

A hook that keeps a ref in sync with a state value automatically — solves the stale closure problem in long-lived callbacks like `setInterval`:

```tsx
import { useRef, useEffect } from 'react';

function useSyncedRef<T>(value: T) {
  const ref = useRef<T>(value);

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref;
}

// Usage — replaces manual ref syncing
const [activeSlide, setActiveSlide] = useState(0);
const activeSlideRef = useSyncedRef(activeSlide); // always current inside intervals
```

**The Vue equivalent:** Composables (`useCounter`, `useFetch` etc.) — same concept, different framework.

---

*This document is a living reference — updated as new concepts are covered in practice.*
