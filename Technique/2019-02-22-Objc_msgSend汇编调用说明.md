https://www.jianshu.com/p/a22f9775b991
https://mikeash.com/pyblog/friday-qa-2017-06-30-dissecting-objc_msgsend-on-arm64.html

# ARM64

ARM64 有31个integer寄存器，宽64位（8字节）。
用符号 x0 ~ x30 表示，还可以用w0 ~ w30 访问 低32 位。
在函数调用种，寄存器x0~x7用来存储前8个参数。
这意味着objc msgSend 接受self位于x0, selector cmd 参数位于x1。


```
0x0000 cmp     x0, #0x0  // 判断 self 是否为 nil
0x0004 b.le    0x6c
0x0008 ldr    x13, [x0] // 获取 x0 寄存器，也就是 self 存储到 x13 寄存器中
0x000c and    x16, x13, #0xffffffff8 // 通过 按位运算 x16 存储 isa 中 class 地址信息
0x0010 ldp    x10, x11, [x16, #0x10] // 它将类的缓存信息加载到x10和x11中
```


```
typedef uint32_t mask_t;

struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
}
```
从ldp指令看，x10储存_buckets的值，而x11的高32位存储_occupied的值，低32位存储_mask的值。

`0x0014 and    w12, w1, w11 `

x1包含_cmd，所以w1包含_cmd的低32位。 
w11包含如上所述的_mask。该指令将两者逻辑与后放入w12。
这个结果相当于 _cmd ％ table_size 的值。
这为缓存表中的hash表的key值


`0x0018 add    x12, x10, x12, lsl #4`

要想从哈希表中加载数据，需要知道其实际地址，因此只有方法名索引还不够。

`x12, lsl #4` 代表着x12左移4位（乘 16 ），与 bucket地址 （x10）相加后赋值给x12。

因为每个哈希表bucket都是16字节。现在x12指向了要搜索的第一个bucket的地址

该指令加载x12(当前bucket)的地址给x9和x17。每个bucket包含一个SEL和一个IMP。 x9现在指向当前bucket的SEL，x17指向当前bucket的IMP。


```
0x0020 cmp    x9, x1
0x0024 b.ne   0x2c

```
这些指令将x9中的选择子与x1中的_cmd进行比较。如果它们不匹配，那么当前的bucket不包含我们正在查找的选择子。在这种情况下，第二条指令跳转到偏移0x2c，用来处理不匹配的bucket。如果匹配到选择子，那么就找到了正在查找的条目，并继续执行下一条指令。

`0x0028 br    x17`

这里无条件跳转到x17，它代表当前bucket加载的IMP。在这里执行实际的目标方法的实现函数，同时也是objc_msgSend的快速查询的最终阶段。
