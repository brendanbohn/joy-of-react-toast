# Stretch Goal for Toast Project: Allow Toasts to dismiss themselves after a given amount of time

It's common for toast libraries to allow toasts to disappear after a given duration. See [React Hot Toast's duration prop](https://react-hot-toast.com/docs/toast#default-durations) as an example. Our current implementation requires a user action (clicking the close button or escape key) to dismiss toasts. This is good for certain types of toasts, perhaps an important message that you need to know that the user has seen. Other toasts with less importance might benefit from disappearing on their own after spending some time on the screen.

## My process and solution

My starting point was essentially the same as what Josh shared for the final solution to the Toast Project.

### Step 1: Add a setTimeout within a UseEffect that runs based on a new duration prop

```js
// Toast.js

// I've added an duration prop defaulting to 3000
function Toast({ id, variant, duration = 3000, children }) {
  const { dismissToast } = React.useContext(ToastContext);

  // Implement a useEffect that calls dissmissToast
  // after a given druation
  useEffect(() => {
    if (!duration) {
      return;
    }

    const timeoutId = window.setTimeout(() => {
      dismissToast(id);
    }, duration);

    // Always cleanup!
    return () => {
      window.clearTimeout(timeoutId);
    };
  }, [duration, id, dismissToast]);

  // The rest of Toast code goes here
}
```

This solution works if there is only one Toast, but as other Toasts are added to the page in quick succession you'll notice a bug pop up. The Toasts do indeed dismiss themselves but they do so in the opposite order you are hoping for and these dismisalls are spaced evenly by the duration amount rather than determined by the time they were initially rendered to the page.

#### Why are timeouts not working as expected?

Put a `console.log('Inside useEffect)` inside of the `useEffect` in Toast.js and you'll see that this effect runs for all Toasts on the page whenever the state of `toasts` changes. When a Toast is added or removed, all existing Toasts on the page re-run their `useEffect` which cleans up their timeout and resets a new timeout -- starting the countdown all over again. This means that they can end up staying on the page for much longer than they're supposed to, resetting there countown with each render.

The issue lies with the `dismissToast` function we're getting from `ToastContext` which is passed as a dependency to the `useEffect`.

```js
// Toast.js

useEffect(() => {
  // logic
  // dismissToast dependency is the issue
}, [duration, id, dismissToast]);
```

The other dependencies are scalar values that go un-changed between renders but `dismissToast` is different. Looking at `ToastProvider.js` you can see that the function body internals rely on `toasts`. You might think that you could wrap it in a `useCallback` to preseve between renders but seeing as it would have `toasts` as a dependency, the function reference would still be updated. This causes the useEffect to run again since its `dismissToast` dependency has changed. We would also have the issue of working with stale `toasts` when using `dismissToasts` in this way.

```js
// ToastProvider.js
function dismissToast(idToDismiss) {
  // we rely on the toast array here
  const nextToasts = toasts.filter(({ id }) => id !== idToDismiss);
  setToasts(nextToasts);
}
```

### Step 2: Ensure values coming from useContext don't cause unwanted useEffect re-running

Lets pass `setToasts` to `value` within ToastProvider because we know that reference will be maintained between renders.

```js
// ToastProvider.js

function ToastProvider({ children }) {
  // logic...

  return (
    <ToastContext.Provider
      value={{
        toasts,
        createToast,
        setToasts, // setToasts is added
        dismissToast,
      }}
    >
      {children}
    </ToastContext.Provider>
  );
}
```

Now we'll use `setToasts` within the `useEffect` rather than `dismissToast`.

```js
// Toast.js

function Toast({ id: currentId, variant, duration = 3000, children }) {
  const { setToasts, dismissToast } = React.useContext(ToastContext);

  useEffect(() => {
    if (!duration) {
      return;
    }

    const timeoutId = window.setTimeout(() => {
      setToasts((toasts) => {
        const nextToasts = toasts.filter(({ id }) => {
          return id !== currentId;
        });
        return nextToasts;
      });
    }, duration);

    return () => {
      window.clearTimeout(timeoutId);
    };
  }, [duration, currentId, setToasts]);

  // component logic..
}
```

The key here is to pass an updater function to `setToasts` that reads the current state of `toasts` without relying on it as a dependency for the effect. This fixes the issues we had when `dismissToast` was used.

At this point everything should be working.

## Other solutions?

While my solution seems to be working, I'm still very new to React so I'm not 100% I'm solving it correctly. I'm curious what the best practice solution would be for this problem.

Josh has shown the pattern of hiding the set state functions behind an abstraction (such as `dismissToast`). But these abstracted functions rely on state variables that don't allow their references to be preserved to prevent extra useEffet calls. Since setState functions have access to the most recent state through the function syntax, a dependency array isn't required that would cause extra useEffect calls in situations similar to the one I've outlined above.
