---
title: nodejs array join
date: 2020-03-25 17:15:07
tags:
- Array
- Uint8Array
- nodejs join
---

# Array 和 Uint8Array 初始化

new Array(3) 初始化为 3个 undefine 元素的 []
new Uint8Array(3) 初始化为 3个 ‘0’ 元素的 []

## Array join

```js
console.log(new Array(3).join('0')) // 输出: "00"
console.log(new Uint8Array(3).join('0')) // 输出: "00000"
```

## leftPad 的两种实现，运用 Uint8Array 和 Array 的区别

```js
function leftPadByUint8Array(input, num) {
  if (input.length >= num) return input

  return (new Uint8Array(num - input.length)).join('') + input
}

function leftPadByArray(input, num) {
  if (input.length >= num) return input

  return (new Array(num - input.length + 1)).join('0') + input
}

console.log(leftPadByUint8Array('00567', 8) == leftPadByArray('0567', 8))
```