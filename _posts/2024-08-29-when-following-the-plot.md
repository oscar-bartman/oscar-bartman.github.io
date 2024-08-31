you want to see things happen in the place where you are reading. 

Not five levels deep,

amid forgotten bugfixes. 

```js
function sideEffect(message) {
  message.content = "side effect"
}

function noSideEffect(message) {
  return { ...message, content: "no side effect" }
}
```
