---
title: 'TDD的起点'
date: 2022-05-15 11:18:07
tags: [f0rb说]
published: true
hideInList: false
feature: 
isTop: false
---
## y=f(x)


而Java8的`java.util.function.Function`接口的定义是这样的
```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
    
    ...
}
```
如果我们把这里的`R apply(T t)`改写成`Y f(X x)`，调用的代码这样写：
```c
// define
X x;
Y y;
Function<X, Y> func;

// invoke
y = func.f(x);

```
我们就会发现，数学里的函数和编程里的函数在抽象模型上和字面写法上并没有什么本质的区别，它们都遵循同样一个模型：**Input-Process-Output(*IPO*)模型**。其中x表示输入，f表示处理，y表示输出，只不过在数学里x被称作自变量，y被叫作因变量，而在开发中x称作入参，y称作返回值。

那么，我们对于这样的IPO模型进行测试的时候，就需要给出一个输入`Given`，等待它的执行`When`，`Then`判断它的输出是否符合我们的期望，这便是测试用例里**Given-When-Then**写法的由来。

`Input-Process-Output`是对`y=f(x)`的阐述。

`Given-When-Then`是对`y=f(x)`的演绎。

## Given-When-Then与Input-Process-Output

Given-When-Then定式和Input-Process-Output模型就是TDD中的一对经典的矛盾组合，我们进行TDD的过程就是一个以Given-When-Then之矛进攻Input-Process-Output之盾的过程。

我们来看一下这样一组例子。

先给出一个测试用例：x=1, y=1，套用一下Given-When-Then的写法：
```
given x=1
when y=f(x)
then assert y=1
```
我们可以给出满足期望的一个函数为：`y=f(x)=x`，写成代码就是：
```c
int f(int x) {
    return x;
}
```

再给出第二个测试用例：x=2, y=4，继续套用Given-When-Then的写法：
```
given x=2
when y=f(x)
then assert y=4
```
我们可以将函数修正为：`y=f(x)=x^2`，这个函数可以同时满足以上两个测试用例，而对应的代码修改为：
```c
int f(int x) {
    return x * x;
}
```

我们可以看到在这两个例子中，最中间最核心的一句话就是`y=f(x)`，从上往下的`given-when-then`三句话依次对应了`y=f(x)`中从右往左的三个关键字`x`, `f`, `y`。而`f(x)=x`和`f(x)=x^2`就是我们通过两个测试用例(Test)驱动(Driven)出来的结果(Development)。

## 小结

所以，下次当你还没想好怎么编写你的测试用例的时候，不妨从这个模板开始：
```java
    @Test
    void name() {
        X x;
        Y y = f(x);
    }
```
然后定义一下变量`x`的类型和初始值(重构)，再定义一下`y`的类型和期望值(重构)，最后再给函数f和测试用例`重命名`一个合适的名字(重构)，开启又一次TDD之旅。
