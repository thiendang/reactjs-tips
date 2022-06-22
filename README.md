# ReactJS Tips

This repo will show you the tips in ReactJS, this is my experience when I'm working on it. I also have references from other places!

### Here is the tips

- [React Hooks](#react-hooks)
  - [Working with useEvent](#working-with-useEvent)
- [React Redux](#react-redux)
  - [Use shallow compare for `useSelector`](#use-shallow-compare-for-useSelector)
  - [Only defined the values we want to use](#only-defined-the-values-we-want-to-use)

# React Navigation

### Working with `useEvent`
    The useEvent hook will keep the function reference and not recreate between components re-rendered.
- Why `useEvent`?
    Consider the following code snippet:


```javascript
function Parent() {
  const [message, setMessage] = useState('')
  // Some other logic in the component
  const handleOnClick = () => {
    sendMessage(message)
  }

  return <SendButton onClick={handleOnClick} />
}
```

We assume that internally we wrapped the “SendButton” with `React.memo` and when the “Parent” is re-rendered, the “SendButton” is also re-rendered.

`React.memo did shallowly compare` between re-renders and it will break the memoization because the handleOnClick function has a `unique function reference on every re-render.`

If the “SendButton” contains nested or many components, this could cause performance concerns because they will be re-rendered.

```javascript
const funcA = () => {
  sendMessage(message)
}

const funcB = () => {
  sendMessage(message)
}

funcA === funcB // false
```

Even though the function body is identical, each time it is re-rendered, a new “handleOnClick” function is created, and each one is unique.

- Solve with `useCallback`

```javascript
function Parent() {
  const [message, setMessage] = useState('')
  // Some other logic in the component
  const handleOnClick = useCallback(() => {
    sendMessage(message)
  }, [message])

  return <SendButton onClick={handleOnClick} />
}
```

We wrapped “handleOnClick” with `useCallback`. Unless the `message` dependencies change, it will not construct a new function and the “SendButton” will not re-render. Easy fix though 🤓.

Probably not. Consider If the `message is often changing`, “handleOnClick” will be unique in each re-rendered version. Furthermore, we can’t remove the message for the dependencies because handleOnClick requires the most recent message value.

We did several things, yet the problem still exists. Let’s discuss the `useEvent`.

- Say hello to useEvent

```javascript
function Parent() {
  const [message, setMessage] = useState('')
  // Some other logic in the component
  const handleOnClick = useEvent(() => {
    sendMessage(message)
  })

  return <SendButton onClick={handleOnClick} />
}
```

The <em>useEvent</em> interface is similar to useCallback, <strong>but it does not include a dependency list</strong>. The “handleOnClick” always be the same reference and the message will always reflect the current values.

As a result, memorizing “SendButton” will work because the function of onClick props will always be the same.

- Examples of useEvent

```javascript
function Component({ theme }) {
  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch('https://example.com')
        await response.json()
        Toster.success(theme, 'Successfull fetch the data')
      } catch (e) {
        Toster.error(theme, 'Something went wrong')
      }
    }
    fetchData()
  }, [theme])
}
```

This is a bit contrived example but anyway 🥲. As you can see. We are using <em>fetch</em> to get the data. If successful, the success toaster is activated; otherwise, the failure toaster is activated. Moreover, “Toaster” requires the use of <strong>theme and message as arguments.</strong>

The problem is that once the theme changes, we have to re-run the useEffect and fetch the data because the useEffect has a theme as a dependency. That is not what we want. We only want this component mounted for the first time. Try using the useEvent hook to address the problem.

```javascript
function Component({ theme }) {
  const onFetchSuccess = useEvent(() => {
    Toster.success(theme, 'Successfull fetch the data')
  })

  const onFetchFailed = useEvent(() => {
    Toster.error(theme, 'Something went wrong')
  })

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch('https://example.com')
        await response.json()
        onFetchSuccess()
      } catch (e) {
        onFetchFailed()
      }
    }
    fetchData()
  }, [theme])
}
```

It can be split into two functions. <em><strong>onFetchSuccess</strong></em>, <em><strong>onFetchFailed</strong></em> and wrapped with useEvent hook. As a result. These functions can access the latest <em>theme</em> value and useEffect to no need to add a theme as a dependency.

### When should useEvent NOT be used?
1. #### Call the function during the render()
  If useEvent is called during the render, it will throw an exception. In this case, useCallback still works.

```javascript
function Container({ item }) {
  // This will throw error. Don't do it.
  const renderItemListA = useEvent(() => {
    return <List items={items} />
  })
  // Call during render, then it's not an event.
  const renderItemListB = useCallback(() => {
    return <List items={items} />
  }, [items])

  return (
    <div>
      {renderItemListA()}
      {renderItemListB()}
    </div>
  )
}
```
2. #### Not all the functions are event
Consider the following snippet for filtering "member" depending on "search"
```javascript
function Container() {
  const [search, setSearch] = useState('')
  const [members, setMembers] = useState([])
  // Invalid mark as an event
  const updateMembers = useEvent(() => {
    if (search) {
      const filterMembers = members.filter(member => member.userEmail.indexOf(search) > -1)
    } else {
      setMembers(members)
    }
  })
  useEffect(() => {
    updateMembers
  }, [])
  //....
}
```
It’s NOT working. Since “updateMembers” is marked as an event. The useEffect will no longer depend on “search” and “members” will no longer be updated while searching. So make sure that you correctly mark the function as the event.

### Conclusions
The useEvent can solve the issue of revalidating too much of the useCallback or answer the question (should I useCallback everywhere?).

**[⬆ Back to Top](#here-is-the-tips)**

# React Redux

1. #### Use shallow compare for `useSelector`
   When we defined the value in useSelector it didn't check the object as well so we will use shallow compare to make it re-render only when the object had changed.

- Step 1: Create `useShallowEqualSelector` function

```javascript
import { shallowEqual, useSelector } from 'react-redux';

export function useShallowEqualSelector(selector) {
  return useSelector(selector, shallowEqual);
}
```

- Step 2: Enjoy it

```javascript
const { uid } = useShallowEqualSelector((state) => ({
  uid: state.userInfo?.uid,
}));
```

**[⬆ Back to Top](#here-is-the-tips)**

2. #### Only defined the values we want to use

❌ Wrong

```javascript
const { userInfo } = useShallowEqualSelector((state) => ({
        userInfo: state.userInfo,
}));

render(){
    return <Text>{userInfo.userName}</Text>
}
```

---

✅ Correct

```javascript
const { userName } = useShallowEqualSelector((state) => ({
        userName: state.userInfo.userName,
}));

render(){
    return <Text>{userName}</Text>
}
```

Why❓
Because when you defined the object but you only want to use some fields in there it will re-render when you dont want to. Example, you just want to use `userName` but you defined the `userInfo` object so when `userInfo.address` changed your component will re-render.

**[⬆ Back to Top](#here-is-the-tips)**

