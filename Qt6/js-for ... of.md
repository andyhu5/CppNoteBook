## `for...of`

因为 Set 本身可迭代,不需要 forEach 这种回调式写法,直接用 `for...of` 更直观、性能也更好(没有函数调用开销):

```javascript
for (let x of set) {
  console.log(x);
}
```

一句话总结:**`set.forEach(console.log)` 语法上没问题,但因为 Set 的 forEach 回调是三个参数,直接传函数名会暴露多余信息;真正想简洁又准确,用 `for...of` 是更好的选择。**