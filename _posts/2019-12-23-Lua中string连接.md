---
title: Lua中的string连接
date: 2019-12-23 22:09:02
tags: [Lua]
categories: 技术文章
---

# Lua中的string连接

Lua中常用的字符串连接有三种：1 直接用`..` 2 `table.concat()` 3 `string.format()`，但是三者性能表现有很大差距，而且有些情况下还会因为使用方法不当产生巨大的内存开销，有必要研究一下其中的原理。

### ..连接符

不少有关于Lua性能分析的文章中都会谈到`..`连接符，每次都要把`..`批判一番，它似乎一无是处。但在这里我要为`..`小小辩护一下：

这些文章中往往举出的例子是《Programming in Lua》中的这一段：

```lua
local buff = ""
for line in io.lines() do
       buff = buff .. line .. "\n"
end
--这段代码读取一个350kb大小文件的时候需要约1分钟，同时会移动(开辟后又被回收)50GB内存。
```

想要解释这个问题，必须先了解一下Lua的string机制：当我们创建一个字符串的时候，Lua首先会检查内部是否已经有相同的字符串了，如果有就会直接返回其引用，没有的话才会真正创建。

这样在`buff = buff .. line .. "\n"`字符串拼接的时候，Lua先要查找是否有相同的字符串，然后会把后者整个拷贝然后拼接，创建一块新的内存，buff指向这块内存缓存下来。每一次查找都是一次hash，每一次新字符串的创建都涉及到内存的分配，所以这一段代码无论在时间性能还是空间角度都是非常浪费的。

但是问题的根源是Lua特殊的string机制，并不是`..`操作符本身；Lua的机制决定了**在for循环中去连接字符串并把中间结果保存下来**是非常危险的，无论是用的`..`还是string.format()。而想要解决这个问题，可以在循环中把新增的字符串缓存到table中，最后在循环外table.concat()。

事实上，`..`这个运算符（而不是函数）的实现是非常高效的：

```
local str = "a" .. "b" .. "c" .. "d" .. "e" .. "f";
```

这样一段代码，很多人认为"a" .. "b" .. "c"需要先执行tmp = "a" .. "b"，再执行tmp = tmp .. "c"的操作，这样的操作如果进行很多次的话，其时间复杂度就会平方增长，然而如果我们使用`luac -l text.lua`指令查看生成的opcode：

        1       [1]     LOADK           0 -1    ; "a"
        2       [1]     LOADK           1 -2    ; "b"
        3       [1]     LOADK           2 -3    ; "c"
        4       [1]     LOADK           3 -4    ; "d"
        5       [1]     LOADK           4 -5    ; "e"
        6       [1]     LOADK           5 -6    ; "f"
        7       [1]     CONCAT          0 0 5
        8       [1]     RETURN          0 1
会发现`..`的仅仅是一个操作符，在Lua的内部实现中它会一次性把要连接的字符串全部找出来然后memcpy到新字符串上，无论`..`连接了多少个字符串，CONCAT都只会执行一次，`..`不会保存中间结果，并没有多余的内存申请和hash查找。

### table.concat(）

concat内部实现函数是tconcat，如下：

```
static** **int** tconcat (lua_State *L) {
  luaL_Buffer b;
  lua_Integer last = aux_getn(L, 1, TAB_R);
  **size_t** lsep;
  **const** **char** *sep = luaL_optlstring(L, 2, "", &lsep);
  lua_Integer i = luaL_optinteger(L, 3, 1);
  last = luaL_optinteger(L, 4, last);
  // 申请一块buff，存合并后的数据，初始buff大小为8192
  // 当buff大小用完后就直接用luaV_concat在栈上做字符串连接
  luaL_buffinit(L, &b);
  // 把table里面索引从i到last的值取出来，并放到buff里面，buff大小为BUFSIZ(8192)
  // 每当写满一个buff，就把buff生成一个TString，并放到栈上，并把buff清空重新写
  for (; i < last; i++) {
     addfield(L, &b, i);
     luaL_addlstring(&b, sep, lsep);
  }
  // 把最后一个元素放入buff
  // 把buff生成一个TString，并放到栈上
  if (i == last)  */\* add last value (if interval was not empty) \*/*
     addfield(L, &b, i);
  // 把栈上之前所有生成的TString通过luaV_concat（实际就是..）合并成一个TString，作为返回值
  luaL_pushresult(&b);
  return 1;
}
```

从源码上看，table.concat没有频繁申请内存，只有当写满一个8192的BUFF时，才会生成一个TString，最后生成多个TString时，会有一次内存申请并合并。在大规模字符串合并时，应尽量选择这种方式。

### string.format()

string.format本身是一个函数调用，会根据%将字符串分割成多个部分，然后进行格式字符串的扫描和复制，然后拼接出最终结果存储在buffer上。比起`..`额外的开销主要来源于格式字符串的扫描和复制。

### 结论：

对比以上三种拼接方式的原理可知：

1 大多数情况下，使用`..`即是最优。单单`..`的操作不会产生多余的临时string；

2 当需要反复拼接的时候，`table.concat`效率更高。尤其是在类似于for循环中，`..`会产生内存的分配或者拷贝，中间会产生很多的临时string，增加额外的gc压力，而`table.concat`可以利用`luaL_Buffer`大幅减少拼接的次数，避免产生更多的临时string。

3 string.format只适合于处理较为复杂的带格式的字符串，代码运行效率上没什么优势。

