```javascript
// immediate.js
const fs = require('fs');

fs.readFile(__dirname + '/immediate.js', () => {
  setTimeout(() => { console.log('timeout'); }, 100); // (2)

  // 200ms 待つ
  const last = Date.now() + 200;
  while (Date.now() < last) {}

  setImmediate(() => { console.log('immediate'); }); // (1)
});

// $ node immediate.js
// immediate
// timeout
```
