原型链（Prototype Chain）是 JS 实现"继承"和"属性查找"的核心机制。这是 JS 和 C++ 最本质的区别之一，因为 **C++ 没有原型链这个概念**——C++ 用的是完全不同的机制（类的静态继承 + 虚函数表）。下面详细讲解。

## 1. 先理解一个现象：为什么数组能调用 `.map()`？

```javascript
let arr = [1, 2, 3];
arr.map(x => x * 2); // 这个map方法是从哪来的？arr自己没定义它
```

`arr` 这个对象自身只存了 `[1,2,3]` 这些数据，`.map()` 方法并不在 `arr` 身上，而是在 `arr` 背后关联的一个"原型对象"上。这个"背后关联"就是原型链。

## 2. 每个对象都有一个隐藏的 `__proto__` 指针

```javascript
let arr = [1, 2, 3];

console.log(arr.__proto__ === Array.prototype); // true
console.log(arr.__proto__.__proto__ === Object.prototype); // true
console.log(arr.__proto__.__proto__.__proto__); // null（链的终点）
```

查找过程是这样的：

```
arr（自身：1,2,3）
  ↓ 没找到map，往上找
Array.prototype（有map、filter、forEach...）
  ↓ 没找到toString，再往上找（假设自定义查找toString）
Object.prototype（有toString、hasOwnProperty...）
  ↓
null（链的尽头）
```

这条"一路往上找"的链，就叫**原型链**。

## 3. 用 class 写继承时，原型链在背后做了什么

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }
  eat() {
    console.log(`${this.name} is eating`);
  }
}

class Dog extends Animal {
  bark() {
    console.log(`${this.name} is barking`);
  }
}

let dog = new Dog('旺财');
dog.eat();  // "旺财 is eating"（继承自Animal）
dog.bark(); // "旺财 is barking"（自己的方法）
```

背后的原型链结构：

```
dog（实例，自身只有name属性）
  ↓
Dog.prototype（有bark方法）
  ↓
Animal.prototype（有eat方法）
  ↓
Object.prototype
  ↓
null
```

调用 `dog.eat()` 时，JS 引擎沿着这条链一直往上找，直到在 `Animal.prototype` 上找到 `eat`。

## 4. 对比 C++：静态继承 vs 动态原型链

C++ 的继承是**编译期确定**的，类的内存布局、方法地址在编译时就已经固定：

```cpp
class Animal {
public:
    string name;
    void eat() { cout << name << " is eating" << endl; }
};

class Dog : public Animal {
public:
    void bark() { cout << name << " is barking" << endl; }
};

Dog dog;
dog.name = "旺财";
dog.eat();  // 编译器在编译期就已经知道eat()的地址
dog.bark();
```

| 特性                   | C++ 继承                           | JS 原型链                            |
| ---------------------- | ---------------------------------- | ------------------------------------ |
| 确定时机               | 编译期（静态）                     | 运行时（动态）                       |
| 本质                   | 内存布局 + 虚函数表(vtable)        | 对象之间的指针链（`__proto__`）      |
| 能否运行时修改继承关系 | ❌ 不能                             | ✅ 可以（运行时改`__proto__`）        |
| 方法查找方式           | 编译器直接生成地址跳转，或查vtable | 运行时沿链逐层查找属性               |
| 多态实现               | 靠虚函数表（vtable指针）           | 靠原型链天然支持，任何方法都可"覆盖" |

## 5. 惊喜之处：JS 原型链可以在运行时随意改动

这是 C++ 完全做不到的，因为 C++ 的类结构在编译后就固定了：

```javascript
let dog = new Dog('旺财');

// 运行时给整个Dog类"新增"一个方法，所有实例立刻都能用
Dog.prototype.sleep = function() {
  console.log(`${this.name} is sleeping`);
};

dog.sleep(); // "旺财 is sleeping" —— 类定义完之后依然可以动态扩展！
```

这在 C++ 中是不可能的：你不能在程序运行过程中给一个已经编译好的 `class Dog` 动态加一个新的成员函数。

## 6. 属性查找 + 遮蔽（Shadowing）现象

```javascript
class Animal {
  eat() {
    console.log('Animal eating');
  }
}

class Dog extends Animal {
  eat() { // 重写eat，此时Dog实例调用eat会找到自己这层，不会往上找Animal.eat
    console.log('Dog eating specially');
  }
}

let dog = new Dog();
dog.eat(); // "Dog eating specially" —— 在Dog.prototype这层就找到了，链查找提前结束
```

这类似 C++ 里子类重写虚函数的效果，但 JS 靠的是"沿着链找到第一个匹配就停"的天然机制，不需要 `virtual` 关键字。

## 总结对比表

| 概念           | C++                                | JavaScript                                |
| -------------- | ---------------------------------- | ----------------------------------------- |
| 继承的实现方式 | 编译期内存布局固定 + vtable        | 运行时对象间 `__proto__` 指针链           |
| 是否可动态修改 | 否                                 | 是（`prototype`可随时增删方法）           |
| 属性/方法查找  | 编译器直接定位                     | 沿原型链逐层查找，直到找到或到null        |
| 多态机制       | 需要`virtual`关键字+vtable         | 天然支持，方法查找本身就是"就近原则"      |
| 内存效率       | 每个实例可能各自独立，或共享vtable | 方法只存一份在`prototype`上，所有实例共享 |

一句话总结：**C++ 的继承是编译器在编译期"焊死"的静态结构，而 JS 的原型链是运行时靠指针一层层"往上问"的动态查找机制**——这也是为什么 JS 被称为"基于原型的语言"，而不是像 C++ 那样"基于类的语言"。