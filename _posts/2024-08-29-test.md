When following the plot, you want to see things happen in the place where you are reading. Not five functions deep, amid forgotten bugfixes. 

```js
const message1 = { content: "message", timestamp: 123456789 }

function sideEffect(message) {
  message.content = "different message"
}

sideEffect(message1)

const message2 = { content: "message", timestamp: 123456789 }

function noSideEffect(message) {
  return { ...message, content: "different message" }
}

const message3 = noSideEffect(message2)
```
