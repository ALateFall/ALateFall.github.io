[toc]

# 格式化字符串漏洞

## 格式化字符串基础

`%p` 将给定的值以地址形式输出，可应用于32位和64位，比`%x`好用

`%s` 将给定的值作为地址，输出该地址处对应的字符串直到`\n`。

`value$` 指定第几个参数，注意不算格式化字符串本身。例如`printf("%2$d", a, b)`输出的是传递的第二个参数，因此输出`b`的值。

`%n` 不同于其他形式，`%n `会计算在其前面的字符个数，并将该值赋值给对应的参数。

例如`printf("test%2$n", a, &b);`中，由于格式化字符串`%n`前面有四个字符，因此该语句执行完毕后，`b`的值为4。此外，例如`printf("test%200d%2$n", a, &b);`执行完毕后，`b`的值为204，因为`%200d`会被视为200个字符。

`%hhn` 写入单字节

`%hn` 写入双字节

## 格式化字符串漏洞基本利用

正常情况下，格式化字符串中的占位符和传递给其的参数是一一对应的。如：

```c
printf("the value of a is %d, the value of b is %l", a, b);
```

其机制为，格式化字符串是传递给函数`printf`的第一个参数，`a`和`b`分别是传递给函数`printf`的第二个和第三个参数。因此，实际上`a`和`b`在32位程序中，是放在栈上的，64位程序中是放在寄存器中的。

由此，若用户没有将格式化字符串中的占位符与后面的参数一一对应，那么程序将会不按照预期运行。取极端的情况，用户仅仅使用了`printf("%d %d %d")`，那么在32位下，程序将会从栈上按顺序取三个值直接输出，64位下程序将会从寄存器中按照函数的传递顺序取出对应的值输出。

## 程序崩溃

由于 `%s` 可以将指定的值作为地址，输出对应地址的字符串，因此可以以以下方式使得程序崩溃：

```c
printf("%s%s%s%s%s%s%s%s%s%s%s");
```

原因是，32位下不太可能出现栈上的这么多连续的值都是可以访问的地址的情况（64位同理）。因此使用一连串`%s`即可使得程序崩溃。

## 泄露内存

### 泄露栈内存

很简单，`printf("%p%p%p%p")`即可泄露四个栈上的值并以十六进制输出。

### 泄露任意地址内存

其实是比较核心的地方。

实际上，当我们使用`printf`函数时，例如`printf("aaa%p%paaa")`时，`printf`的第一个参数是字符串的地址而不是字符串的值本身，但该字符串本身也会存在栈上，但该字符串本身在栈上的位置是未知的。然而，我们可以将其泄露出来。

如下所示：

```c
#include <stdio.h>

int main(){
    char a[100];
    scanf("%s", a);
    printf(a);
    return 0;
}
```

可以看到，里面有一个格式化字符串漏洞，其中的输入是我们可控的。

使用`gcc`编译并运行：

`gcc test.c -o test`

![image-20230221163935049](https://ltfallpics.oss-cn-hangzhou.aliyuncs.com/images/image-20230221163935049.png)

可以看到，第四个`0x`处的值是`0x702507025`，其实就是`%p%p%p...`，说明该字符串本身在栈上是第四个。然而，这是64位程序，要知道64位下`printf`的前6个参数是在寄存器里面的，因此该字符串实际上是`printf`函数的第4+6=10个参数，不算格式化字符串地址本身，就是第九个参数。

因此，若我们输入`[addr]%9$s`，那么就可以将`[addr]%9$s`本身作为地址，输出该地址处的值。

## 覆盖内存

### 覆盖为小数字

使用`aa%k$nxx`可以将第`k`个参数赋值为2。可以和上面泄露任意地址一起使用。

### 覆盖为大数字

以32位为例子，比如我们要将一个地方的值覆盖为一个很大的数字，例如`0x12345678`，并假设是将其覆盖到`0x0804a028`处，由于一般情况下是小端存储，那么我们希望以如下方式覆盖：

```
0x0804A028 \x78
0x0804A029 \x56
0x0804A02a \x34
0x0804A02b \x12
```

也就是，我们要对这四个地方的值进行单字节覆盖。例子`payload`：

```python
p32(0x0804A028)+p32(0x0804A029)+p32(0x0804A02a)+p32(0x0804A02b)+pad1+'%6$n'+pad2+'%7$n'+pad3+'%8$n'+pad4+'%9$n'
```

### pwntools中生成覆盖内存payload

`pwntools`中内置了一个方法可以在有格式化字符串漏洞的情况下生成对指定地址的值进行覆盖的方法。

```python
payload = fmtstr_payload(7, {puts_got:system_addr})
sh.sendline(payload)
# 确定了是覆盖第7个参数，这段代码会将puts_got地址处的值覆盖为system_addr
```

## hijack GOT

实际上就是通过格式化字符串漏洞修改`got`表来利用

## hijack ret_addr

实际上是覆盖栈上的返回值。但由于程序运行时，我们不能知道栈上的地址，因此难以将其返回地址进行覆盖，但要知道栈上的第一个值是`rbp/ebp`的值，栈上的第二个值是函数的返回地址，栈上的第二个值的地址和栈上存放的第一个值的偏移是固定的，gdb调试一下计算出该偏移，因此只需要泄露出栈上的第一个值，通过偏移计算即可得到函数返回地址的地址。覆盖该地址即可。