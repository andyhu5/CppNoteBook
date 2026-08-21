好的,这个问题需要区分"找重复的**值**"还是找重复值对应的**所有位置索引**,下面提供几种常用的 Array 方法实现思路。

## 方法一:用 `filter` + `indexOf`/`lastIndexOf` 找出重复值,再定位所有索引

```javascript
let arr = [1, 1, 2, 2, 3, 4, 5, 7, 7, 9, 9];

// 第一步:找出哪些值是重复的（去重后的重复值列表）
let duplicateValues = arr.filter((item, index) => arr.indexOf(item) !== index);
console.log(duplicateValues); // [1, 2, 7, 9]（每个重复值只会出现一次，因为后面出现的会被filter选中）

// 第二步:针对每个重复值，找出它在原数组中的所有位置
let result = {};
duplicateValues.forEach(val => {
  result[val] = [];
  arr.forEach((item, index) => {
    if (item === val) result[val].push(index);
  });
});
console.log(result);
// { '1': [0, 1], '2': [2, 3], '7': [7, 8], '9': [9, 10] }
```

**原理说明:** `arr.indexOf(item)` 永远返回该值**第一次**出现的索引。如果当前遍历到的 `index` 和"第一次出现的索引"不相等,说明这个位置上的值之前已经出现过一次了,即为重复。

## 方法二:用 `reduce` 一次遍历,同时统计位置(效率更高,推荐)

```javascript
let arr = [1, 1, 2, 2, 3, 4, 5, 7, 7, 9, 9];

let positionMap = arr.reduce((acc, item, index) => {
  if (!acc[item]) acc[item] = [];
  acc[item].push(index);
  return acc;
}, {});

console.log(positionMap);
// { '1': [0,1], '2': [2,3], '3': [4], '4': [5], '5': [6], '7': [7,8], '9': [9,10] }

// 筛选出真正"重复"的（位置数组长度大于1的）
let duplicates = Object.fromEntries(
  Object.entries(positionMap).filter(([key, positions]) => positions.length > 1)
);
console.log(duplicates);
// { '1': [0,1], '2': [2,3], '7': [7,8], '9': [9,10] }
```

**这种写法只需要遍历一次数组**,比方法一(内部嵌套遍历,存在 O(n²) 开销)性能更好。

## 方法三:如果只想知道"重复元素第二次及以后出现"的位置(不含首次出现位置)

```javascript
let arr = [1, 1, 2, 2, 3, 4, 5, 7, 7, 9, 9];
let seen = new Set();
let duplicatePositions = [];

arr.forEach((item, index) => {
  if (seen.has(item)) {
    duplicatePositions.push(index); // 只记录第2次及以后出现的位置
  } else {
    seen.add(item);
  }
});

console.log(duplicatePositions); // [1, 3, 8, 10]
```

**适用场景:** 比如你想保留每个值第一次出现的位置,把"多余的重复项"标记出来准备删除,这种写法就很实用。

## 三种方法对比

| 方法                 | 输出形式                  | 时间复杂度      | 适用场景                   |
| -------------------- | ------------------------- | --------------- | -------------------------- |
| `filter + indexOf`   | 先找出重复值,再单独查位置 | O(n²)，性能较差 | 数据量小、追求代码直观     |
| `reduce` 统计位置表  | `{值: [位置数组]}`        | O(n)，性能更好  | 推荐首选，尤其数据量大时   |
| `Set` 记录已见过的值 | 只列出"多余"的重复位置    | O(n)            | 只需要知道哪些位置可以删除 |

## 如果只是想快速确认"是否存在重复"(不需要具体位置)

```javascript
let arr = [1, 1, 2, 2, 3, 4, 5, 7, 7, 9, 9];
let hasDuplicates = new Set(arr).size !== arr.length;
console.log(hasDuplicates); // true
```

这个方法最快最简洁,原理是:如果没有重复,`Set` 去重后大小应该和原数组长度一致;如果有重复,`Set` 会变小。

## 一句话总结

**如果要精确到每个重复值对应的所有索引位置,推荐用 `reduce` 一次遍历生成"值→位置数组"的映射表(O(n)性能最优)**;如果只是简单判断"有没有重复",用 `Set` 比较长度是最快的写法。