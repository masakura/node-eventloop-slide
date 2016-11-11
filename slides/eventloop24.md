```javascript
// timeout.js
setTimeout(() => { console.log('1000A'); }, 1000); // (2)
setTimeout(() => { console.log('500'); }, 500);    // (1)
setTimeout(() => { console.log('1000B'); }, 1000); // (3)

// 1500 ms 待つ
const last = Date.now() + 1500;
while (Date.now() < last) {}

// $ node timeout.js
// 500
// 1000A
// 1000B
```
