---
title: TypeClass, Functor, Applicative 和 Monad
date: 2024-09-22 14:48:00 +0800
categories: [Programming Languages, FP(函数式编程)]
tags: [fp, haskell, monad]
description: 简要介绍了 Haskell 中的 Functor, Applicative 和 Monad
---

## Type Class -- 从Higher Order Function说起

使用 Javascript、 Python 等语言的读者想必对 Higher Order Function 的概念十分熟悉。Higher Order Function 本身是一个 Function，但是其至少满足以下条件中的一个性质：

1. 接受一或多个 Function 作为参数
2. 能够返回 Function

在 Haskell 中，类似地，我们也有 "Higher Order Class"，其能够接受一或多个类型 (Type) 作为参数，并由此产生出新的类型来，我们把这样的 "Higher Order Class" 叫做 Type Class [^1] 。并且，我们还可以对这些 Type Class 定义一定的限制 (Constraint) ，使得当类型声称自己是某Type Class 的实例 (Instance) 时，其必须满足这些限制。例如，可以这样定义一个 Type Class [^2] ：

```haskell
class Monoid m where -- 定义一个Type Class Monoid
  mempty :: m -- Monoid约束1 幺元
  mappend :: m -> m -> m -- Monoid约束2 二元运算函数，满足当两个参数都属于S时，其运算结果也为S
  mconcat :: [m] -> m -- 实用函数, 类似OOP接口中提供了默认实现的函数
  mconcat = foldr mappend mempty 
```

类型可以声明自己是某 Type Class 的实例，并且提供相关的约束定义：

```haskell
instance Monoid [] where -- 这里[]也是一个Type Class, 其将类型a作为入参，返回[] a这个类型，代表一个容纳a类型元素的List
  mempty :: m
  mempty = []
  mappend :: m -> m -> m
  mappend = \x -> \y -> x ++ y -- 显然地, x ++ (y ++ z) == (y ++ x) ++ z
  -- mconcat :: [m] -> m -- 这里用默认定义就好
```

Type Class 是一个强大的功能，有了 Type Class，我们可以方便地实现如下目标：

1. 当碰到某个类型声称自己是某 Type Class 的实例时，我们便可假定其拥有该 Type Class 具有的性质，并据此对其进行相关操作。譬如说，当我们看到类型a是Monoid的实例时，便可假定 `x::a` 和 `y::b` 具有``x `mappend` y == y `mappend`  x ``的性质
2. 利用 Type Class 构造出我们自己的类型，并将其使用到任意一处该 Type Class 可以使用的地方
3. 当我们需要某个非用户类型具有某个 Type Class 的性质时，我们可以在用户代码中声明其是该 Type Class 的实例，这点是接口做不到的
4.  ……

[^1]: 不要与 OOP 中的类 (Class) 混淆，这是一个不同的概念。我们或许可以叫它元类 (Metaclass) 或者是类型构造器 (Class Constructor) ，其与 OOP 中的接口类似，但在细微之处又略有不同。可参见：[Type Class in perspective](https://diogocastro.com/blog/2018/06/17/typeclasses-in-perspective/) 

[^2]: 这个 Type Class 的 Kind 是 \*->\*，接受一个类型作为参数，返回另外一个类型。\* 在Haskell 中一般指代一个普通类型 (Ordinary Type) ，即那些非高阶的类型，像是Int、String。Monoid 在这里定义了一个幺半群，但运算的同一性和结合性由 Type Class 的实例来自己确保

> Monoid, Functor, Applicative, Monad 所需要满足的数学性质都会放在最后，因为我们不需要这些数学性质也能使用和定义这些Type Class（虽然可能是错误地使用和定义）

## Functor

Functor 是一种 Monoid，所以我们可以这样定义一个 Functor (虽然实际上并不是这么写的)：

```haskell
class Monoid f => Functor f where
  fmap :: (a -> b) -> f a -> f b
  (<$>) :: (a -> b) -> f a -> f b -- fmap的中序形式
  (<$>) = \f x -> fmap f x
```

于是我们可以说，如果一个类型 `f` 是 Functor, 那么它可以接受一个签名为 `(a->b)` 的函数，将 `f a` 转化为 `f b` 。这听起来有点抽象，我们可以举一个更具体的例子。比方说，上面我们提到的 `[]` 就是一个 Functor，假设我们有一个`toString :: Int -> String`方法和一个 `[] Int`类型的值，我们就能利用`toString`将其转化为`[] String`。

```haskell
ints :: [] Int
ints = [1,2,3]
toString :: Int -> String
toString = \x -> show x
strs = fmap toString ints
-- ["1","2","3"]
```

很熟悉对吧，这就是我们在其他语言中经常用到的 map，它把一个容器内的元素映射为别的元素，并保持容器的结构不变（比如说是容纳 n 个元素的 List，那么映射完后仍然是容纳 n 个元素的 List）。只是 Functor 比“容器”这个概念更为准确，Functor 里面不一定真的需要容纳元素，比如说 Haskell 中的 Proxy 并不容纳元素，但它也是一种 Functor； 容器也不一定是 Functor，比如说我们把一个 Set 映射到另一个 Set'，并不能保证 Set' 仍然具有 Set 的性质（即无重复元素）。Functor 就是单纯地，给定`a -> b`，能把`f a`映射到`f b`而已。

## Applicative

Functor 提供了使用`a -> b`将`f a`映射至`f b`的能力，那假设我们不仅需要`(a -> b) -> f a -> f b`的能力，还需要能够直接用`f (a -> b)`将`f a`映射到`f b`呢？Functor 显然无法直接满足这个需求(我们还要开发个取出`a->b`的函数)，从`fmap`的定义就可以看出来。而这个需求很自然，比方说下面的场景：

```haskell
ops1 = [1,2,3]
aps = fmap (*) l -- [(*1),(*2),(*3)]
ops2 = [1,2,3]
-- 我们该怎么做把[(*1),(*2),(*3)]应用到[1,2,3]上?
```

于是，我们就需要一种新的Type Class，好来实现我们的需求。Haskell 中提供这个能力的Type Class就是 Applicative。

```haskell
class Functor f => Applicative f where -- Applicative是一种Functor，可以通过数学方式证明
	pure :: a -> f a -- 我们需要一种方法来将(a->b)变成f (a->b), 好进行下一步的应用(apply), 这个过程叫纯化(pure)
	(<*>) :: f (a -> b) -> f a -> f b -- 这个操作符叫做应用(apply), 一般读作apply或者ap
	liftA2 :: (a -> b -> c) -> f a -> f b -> f c -- Haskell用来实现(<*>)的一个方法, 它跟(<*>)可以互定义,也就是说只需要定义(<*>)和liftA2当中的一个方法就够了
	(<*>) = listA2 id
	(*>) :: f a -> f b -> f b -- 顺序执行, 但丢弃左边的返回值. 可以用于只要副作用不要返回值的情况, 比方说我们想向IO输出一个东西，但是输出完后它的返回值是IO (), 要了也没啥用
```

上面的问题便迎刃而解：

```haskell
ops1 = [1,2,3]
aps = fmap (*) l -- [(*1),(*2),(*3)]
ops2 = [1,2,3]
res = aps <*> ops2 -- [1,2,3,2,4,6,3,6,9], 有点类似笛卡尔积
```

还可以写得更紧凑一点：

```haskell
res = pure (*) <*> [1,2,3] <*> [1,2,3]
```

Haskell 关于 Applicative 的文档还给了这样一个例子，或许更能帮助我们理解 Applicative 中何为 apply：

```haskell
data MyState = MyState {arg1 :: Foo, arg2 :: Bar, arg3 :: Baz}
produceFoo :: Applicative f => f Foo
produceBar :: Applicative f => f Bar
produceBaz :: Applicative f => f Baz
mkState :: Applicative f => f MyState
mkState = MyState <$> produceFoo <*> produceBar <*> produceBaz
```

让我们来一步步看看`mkState`怎么获得一个`f MyState`的。

1. 在这里的 MyState 可以看作是一个`Foo -> Bar -> Baz -> MyState`的函数 (实际上也是) ，`<$>`和`<*>`都是左结合的，`mkState`从左到右求值；
2.  `MyState <$> produceFoo` 得到一个 `f Bar -> Baz -> MyState`
3. `MyState <$> produceFoo <*> produceBar` 得到一个 `f Baz -> MyState`
4. …

于是我们可以这样说：apply 就是给定一个函数的函子 (Functor)，将其实际参数应用到函数上，形成新的函子的过程。Applicative 的实例具有这样的能力。

## Monad

有了 Functor 和 Applicative，我们的生活变得更美好，现在我们可以把一个 Functor 内的元素映射为另外一种元素；通过 Applicative，这种元素不仅只局限于普通的类型，甚至也可以是函数。是时候来点更深邃、更黑暗的幻想了！比方说，假设说我们有一堆函数，它的输出是下一个函数的输入（有点像管道），然后我们有一个运算符`>>=`，能把这些函数串起来，怎么样？

```haskell
functor1 >>= function1 >>= function2 >>= function3
```

心诚则灵，在 Haskell 里，Monad 的实例便具有这种能力：

```haskell
class Applicative m => Monad m where
    return :: a -> f a
    (>>=) :: m a -> (a -> m b) -> m b -- 叫做 bind
    (>>) :: m a -> m b -> m b -- 顺序执行, 但丢弃左边的返回值
    (>=>) :: (a -> m b) -> (b -> m c) -> a -> m c -- Kleisli Arrow, 像是常规函数中用来组合的(.)
```

试想一下，要是没有 Monad， 我们的生活该多么困难。如果我们只有`a -> b`的`f1`和`b->c`的`f2`，当使用`fmap`时，我们必须这样做：

```haskell
l = [...]
l' = fmap f1 l
l'' = fmap f2 l'
```

当`f1`和`f2`都被装在 Functor 里时也可以用 Applicative，但是以一种非常别扭的姿势：

```haskell
pure f2 <*> pure f1 <*> l
```

而且这两种方式的致命缺点是，当我们碰到一些会失败 (Fail) 的方法时，我们不一定有处理它的能力，而 Monad 则能把 Fail 包裹在 Monad 里返回出来。

尽管有了 Monad，但有时候我们也会写出些不太好看的代码。为了避免写出像下面这样别扭的代码，Haskell 还提供了`do`语法，其本身是`>>=`的一个语法糖(不要误会这里的return，虽然我们最后确实return了一个值，但却不是这个return让它return的，其只是把`f a b`变成一个 Monad)。

```haskell
-- 别扭的代码
as >>= (\a -> bs >>= (\b -> return (f a b)))
-- do notation
do a <- as
   b <- bs
   return f a b
```

`>>`也可以用`do`来写，只要我们不取出左边的值就行（取出来也行，反正后面那行也没用到）。

```haskell
-- >>
as >> bs
-- do notation
do as
   bs
```

其等价于：

```haskell
as >>= (\_ -> bs)
```

Monad 的应用在许多语言中都有体现，比方说 C# 中的 LINQ、 Javascript 中的 Promise等，其显著特征就是保持了 Monad 的 wrap 在变动的过程不会丢失，即维持了 Monad 的上下文 (Context) ，从而可以实现平坦 (Flat) 的顺序 (Sequence) 调用。

## 几种Type Class要满足的数学性质

### Functor

1. Identity -- 同一性
```haskell
fmap id == id
```
2. Composition -- 结合性
```haskell
fmap (f . g) == fmap f . fmap g
```

### Applicative

1. Identity
```haskell
pure id <*> v = v
```
2. Composition
```haskell
pure (.) <*> u <*> v <*> w = u <*> (v <*> w)
```
3. Homomorphism
```haskell
pure f <*> pure x = pure (f x)
```
4. Interchange -- apply 的先后不重要
```haskell
u <*> pure y = pure ($ y) <*> u
```

### Monad

1. Left identity -- 把值先包起来再映射等价于直接在值上映射
```haskell
return a >>= k = k a
```
2. Right identity -- Monad 再包一层还是 Monad , i.e. 不会变成 Monad (Monad m)
```haskell
m >>= return = m
```
3. Associativity
```haskell
m >>= (\x -> k x >>= h) = (m >>= k) >>= h
```

[^1]: 
