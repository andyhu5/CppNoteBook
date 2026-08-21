是的，`splice()` 是一个非常灵活的方法，可以完成增（Create）、删（Delete）、改（Update）三种操作，唯一做不到的是"查"（Read），需要配合其他方法。下面逐一说明。

## 基本语法

```javascript
array.splice(start, deleteCount, item1, item2, ...)
```

- **start**：开始操作的索引
- **deleteCount**：要删除的元素个数
- **item1, item2...**：要插入的新元素（可选）

返回值：被删除元素组成的数组（原数组会被**直接修改**）。

## 1. Create（新增）— deleteCount 设为 0

```javascript
let arr = [1, 2, 5];
arr.splice(2, 0, 3, 4); // 在索引2处插入3和4，不删除任何元素
console.log(arr); // [1, 2, 3, 4, 5]
```

## 2. Delete（删除）— 不传插入项

```javascript
let arr = [1, 2, 3, 4, 5];
arr.splice(1, 2); // 从索引1开始删除2个元素
console.log(arr); // [1, 4, 5]
```

## 3. Update（修改）— 同时删除和插入

```javascript
let arr = [1, 2, 3, 4, 5];
arr.splice(2, 1, 'X'); // 删除索引2的元素，替换成'X'
console.log(arr); // [1, 2, 'X', 4, 5]
```

## 4. Read（查询）— splice 做不到，需用其他方式

```javascript
let arr = [1, 2, 3, 4, 5];
console.log(arr[2]);        // 直接索引访问：3
console.log(arr.slice(2,3)); // 用slice查询（不修改原数组）：[3]
```

## 小结对比

| 操作   | 方法                         | 是否修改原数组       |
| ------ | ---------------------------- | -------------------- |
| Create | `splice(i, 0, ...items)`     | ✅ 是                 |
| Delete | `splice(i, count)`           | ✅ 是                 |
| Update | `splice(i, count, ...items)` | ✅ 是                 |
| Read   | `arr[i]` 或 `slice()`        | ❌ 否（splice不适用） |

一个需要特别注意的点：**splice() 会直接修改原数组**（这是它和 `slice()` 最大的区别），如果你希望保留原数组、返回一个新数组，可以考虑用 `slice()` + 展开运算符等不可变（immutable）的写法来实现同样的增删改效果。