---
title: You Can (Not) Redo —— C 函数调用的 coroutine 如何在 lua_pcallk 中 yield
date: 2024-09-28 16:26:00 +0800
categories: [Programming Languages, Lua]
tags: [Lua, coroutine, continuation]
description: 简要介绍了 Lua 中的 coroutine 和 continuation 机制
---

## Lua 中的 coroutine

Lua 麻雀虽小五脏俱全，除却 first-class 的 function、神经病一般 array 和 map 杂糅的 table 外，它还贴心地为你提供了不知道你用不用得上的 coroutine。

Lua 的 coroutine 是一种有栈协程实现，它存储调用时的一整个栈，因此你可以在函数的任何地方 yield 。也就是说，你可以如下面的代码这般，像调用一个普通函数一样调用一个会 yield 的函数，内层函数的 yield 会直接被返回到最顶层上。

```lua
-- 一个随处可见平平无奇的函数
local f = function ()
    coroutine.yield(1)
end
    
local co = coroutine.create(function () 
    f()
    coroutine.yield(2)
end)

coroutine.resume(co) -- true    1
coroutine.resume(co) -- true    2
coroutine.resume(co) -- true
coroutine.resume(co) -- false    cannot resume dead coroutine
```

与之相反，当你使用像 Python 这样使用无栈实现的协程时，则无法达到这样的效果：

```python
# 并不是一个平平无奇的函数, 调用它返回一个 generator 对象
def gen_f():
    yield 1

def gen_co():
    f = gen_f()
    f.send(None)
    yield 2

co = gen_co()
co.send(None) # 2, 谁动了我的 1 ?
co.send(None) # StopIteration
```

为什么不能？因为无栈协程本质上是用一个 closure 来存储协程执行的信息，`f`和`co`是两个不相干的 closure (或许相干，因为我们的例子里 f 被存储在 co 里)，因此 f 的 yield 虽然返回了一个值，但`co`只能够关心自己这一层级上的 yield，而对`f`的 yield 无从感知。而有栈协程则存储的时整一个调用栈，因此它能感知到在这一整个执行过程中的 yield ，并且在下一次执行时从该处恢复。

要想使得 Python 也能像 Lua 一样，我们必须使用特殊的语法，而不能像调用普通函数一样直接调用：

```python
def gen_f():
    yield 1

def gen_co():
    f = gen_f()
    yield from f
    yield 2

co = gen_co()
co.send(None) # 1
co.send(None) # 2
co.send(None) # StopIteration
```

当然，我们为了更简单和直观，实际上演示的是 Python 中的 generator 而不是 coroutine 。Python 标准库实现的 coroutine (asyncio) 就是用 generator 来实现的，感兴趣的朋友可以阅读《Fluent Python》一书中的相关章节。有栈协程和无栈协程孰优孰劣的争论旷日持久，有栈协程能够尽可能抹平异步函数跟普通函数的差异、更加方便调度，而无栈协程却远比有栈协程更加地轻量化。但在语言实现标准库中协程选用的方案中，双方却各有胜场，没有出现说哪一边具有压倒性的优势。

扯远了，让我们说回 Lua 来。Lua 的原生协程相比较其他语言的协程来说功能比较有限，因为它不像 Javascript 或者 Python 那样，提供内建的 Event Loop 来让你运行协程，你得使用第三方库来实现；另外，在不改造源码的情况下，每个 Lua VM 只能运行在单线程上。虽然我在这里看似好像把它贬得一文不值，然而事实是，Lua 协程不仅有人使用，而且还有很多人使用，关于其多线程协程库的实现俯拾皆是。

Lua 的协程在所有代码都是 Lua 代码时工作得相当好，你可以在你任何想返回的地方返回，在任何你想让协程继续的地方继续，毕竟我们只是从一个`lua_State`切换到另一个 `lua_State`而已。不过，我们不要忘记 Lua 称呼自己为 “embeddable scripting language”，其提供了许多 API ，从而使得能够从宿主语言中直接调用 Lua 函数或者从 Lua 代码中调用宿主语言在 Lua VM 里注册的函数，当这种情况发生时，原本很自然的随地大小 yield 又变得稍微有点不一样了。

## You Can Not Redo —— 在 C Function 中用 lua_pcall 调用协程

我们有时候会想要在一个 C Function 里处理一些逻辑后，调用我们在 Lua 中写好的函数，让它来帮我们做后续的事情，就像这样：

```c
// c side
int l_foo(lua_State* L){
    // do something
    lua_getglobal(L, "bar");
    lua_pcall(L,0,1,NULL); // zero argument, one result, no msgh
    int bar_val = lua_tonumber(L,-1); // 假设返回一个 int
    // do something else
    return ret_n; // 返回值数量, 真正的返回值放在 L 指着的栈里
}

// 某个不为人知的地方注册了这个函数
lua_pushcfunction(L, l_foo);
lua_setglobal(L, "foo");
```

```lua
-- lua side
-- 简单返回一个1
local bar = function ()
    coroutine.yield(1)
end

local co = coroutine.create(foo)
coroutine.resume(co) -- true    some_val 期望如此
```

我们可能会期待这个`coroutine.resume(co)`正常返回一个值 ，然而令人惊讶的是，它居然告诉我们“attempt to yield across metamethod/C-call boundary"来阻止我们这样做，怎会如此？

其实原因并不难猜到，C 函数调用跟 Lua 函数调用的最大区别是两个使用的栈是不同的。当在 Lua 里面调用 C 函数时，其使用的是宿主语言的栈 (在这里就是C)；而单纯的 Lua 函数使用的是`lua_State`里维护的栈，在每次创建协程 (即调用`lua_newthread`) 时，都会创造出一个新的`lua_State`来。

在我们的例子里，当 yield 发生时，其栈的图景是这样的：

![c_stack_before_resume](/assets/img/attachments/c_stack_before_resume_0928.svg)
_Before Resume_

由于 yield 本身使用的是`setjump`和`longjump`来实现，因此当 yield 时，Foo 的栈就会被 unwind ，回到运行`coroutine.resume(co)`时刻的栈：

![c_stack_after_resume](/assets/img/attachments/c_stack_after_resume_0928.svg)
_After Resume_

注意到，在下一次执行`coroutine.resume(co)`前，我们还可能调用其他 Lua Runtime 内的函数或者其他 C 函数，因此，Foo Stack 会被这些调用破坏掉 (corrupted) ，(即使我们能调用`coroutine.resume()`)我们也无法回到这个栈进行尚未完成的运算。这就是 Lua 阻止我们在一个 C 函数内 yield 的原因。

我们说过，当所有函数都是 Lua 函数时没这个问题，因为我们的栈并不在 C Stack里，而是保存在`lua_State`里，其自然不会因为`longjump`而被破坏掉。所以概括地讲，这种错误只会发生在 (resume) C Function registered in lua -> Lua Function ->  (yield) 这样的情况下，即 "yield across" 的情况。

## You Can Redo —— Lua 5.2+ 的 lua_pcallk

解决这样问题的方式当然是把 C Stack 在那个时刻的状态保存下来，但我们没有办法这样做，因为 Lua 是 "embeddable script language"， C 才是承载 Lua 的那个宿主语言。所幸， Lua 5.2+ 提供了一个折衷的办法，让代码的编写者来决定 yield 发生时需要哪些状态，并把这些状态通过`ctx`保存起来，传递给被称作 "continuation" 的函数，继续下面本应该进行的执行。当然，这个 continuation 肯定不是原来的函数了，而只是一个原来函数意志的继承者。

由于从 C 进入 Lua 的边界有 3 个函数，`lua_pcall`、`lua_call`和`lua_yield`，因此 Lua 5.2 之后提供了对应的含 continuation 的 3 个函数：`lua_pcallk`、`lua_callk`和`lua_yieldk`，其中的 k 指代的便是那个 continuation 。

```c
// lua_pcallk 的函数签名 Lua 5.3
int lua_pcallk (lua_State *L, // the same
                int nargs,
                int nresults,
                int msgh,
                lua_KContext ctx,
                lua_KFunction k); // continuation function
// lua_KFunction 的定义
typedef int (*lua_KFunction) (lua_State *L, int status, lua_KContext ctx);
```

我们可以用它来改写我们上面的代码逻辑，使得`foo`可以在 yield 后还能再次 resume。`lua_pcallk`在被调用函数 yield 时返回`LUA_YIELD`，正常情况返回`LUA_OK`，在其余情况则返回原本的错误码。同时，其还会把`k`存储在`L`中，供下次 resume 时使用，i.e. 第一次 resume 时调用`l_foo`，之后的 resume 都调用`finish_foo`。

```c
// c side
int finish_foo(lua_State *L, int status, lua_KContext ctx){
    // ...
    if (status == LUA_YIELD){
        // do something, you can just return yield value or do some operations on it
    }
    else if(status == LUA_OK){
        // do something
    }
}

int l_foo(lua_State* L){
    int status;
    lua_KContext ctx;
    // do something
    lua_getglobal(L, "bar");
    status = lua_pcallk(L,0,1,NULL,ctx,finish_foo); // zero argument, one result, no msgh
    
    return finish_foo(L,status,ctx);
}

// 某个不为人知的地方注册了这个函数
lua_pushcfunction(L, l_foo);
lua_setglobal(L, "foo");
```

```lua
-- lua side
-- 简单返回一个1
local bar = function ()
    yield 1
end

local co = coroutine.create(foo)
coroutine.resume(co) -- true    some_val
coroutine.resume(co) -- true    final_val
```

好了，就这样，我们实现了在 C 函数中使用 Lua 协程 yield 的成就，おめでとう! (👏)