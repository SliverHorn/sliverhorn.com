---
title: "错题本"
date: 2022-04-10T17:17:17+17:17
draft: true
---

<!--more-->

1. 设说明为 `int x;` 与 `!x` 等价的表达式是

 A. `x != 0`     

B. `x == 0`    

C. `x != 1`

D.  `x == 1`

> B

2. 设 `int k=0; if (k=3) printf("####"); else printf("&&&&");` 输出结果是 

 A. `####`   

B. `&&&&`    

C. `####&&&&`   

D. 无输出结果

> `k = 3` 为真，所以选 A

3. 程序输出结果是 

   ```c
   #include <stdio.h>
   
   long f(char *s) {
       char *p = s;
       while (*p != '\0') p++;
       return (p - s);
   }
   
   int main() {
       printf("%ld\n", f("abcdef"));
   }
   ```

    A. 0       B. 6      C. 7        D. 8

   > B

4. 设char s[ ]="ABCD", *p=s; printf("%d\n", *(p+4)); 的输出是

   A. 68       

   B. 0      

   C. 字符 'Ｄ'的地址  

   D. 不确定值

> B

5. 以下叙述中不正确的是 

A.在C中，函数中的自动变量可以赋初值，每调用一次，赋一次初值。

B.在C中，在调用函数时，实际参数和对应形参在类型上只需赋值兼容。

C.在C中，静态变量如果不初始化，值为不定值。

D.在C中，函数形参是局部变量。

> C	

6. `015 & 15 | 0xa + 1` 值为

> 015 & 15 : 
>
> - 015 为八进制，转成十进制为 13 ，13 转二进制为 1101
> - 15 为 十进制， 转成二进制为 1111
> - 1101 & 1111 结果为 1101
>
> 1101 & 0xa + 1 :
>
> - 0xa 为十六进制， 转成十进制为10 
> - 0xa + 1 结果为 11
> - 11 转成二进制为 1011
> - 1101 | 1011 结果为 1111 ，1111 转成十进制为 15

7. 阅读下述程序，写出执行结果

```c
int main() {
    int a = 10, y = 0;
    do {
        a += 2;
        y += a;
        if (y > 50)
            break;
    } while (a = 14);
    printf("a=%d, y=%d\n", a, y);
}
```

>- 第一次循环
>  - 第4行：a = 12
>  - 第5行：y = 12
>  - 第8行：a = 14
>- 第二次循环
>  - 第4行：a = 16
>  - 第5行：y = 28
>  - 第8行：a = 14
>- 第三次循环
>  - 第4行：a = 16
>  - 第5行：y = 44
>  - 第8行：a = 14
>- 第四次循环
>  - 第4行：a = 16
>  - 第5行：y = 60
>  - 第7行：退出循环
>
>程序执行结果为 `a=16, y=60`

8. 阅读下述程序，写出执行结果

```c
#include <stdio.h>

int main() {
    int i;
    for (i = 4; i >= 2; i--) {
        switch (i) {
            case 0:
                printf("%4s", "ABC ");
            case 1:
                printf("%4s", "DEF ");
            case 2:
                printf("%4s", "GHI ");
                break;
            case 3:
                printf("%4s", "JKL ");
            default:
                printf("%4s", "MNO ");
        }
    }
    printf("\n");
}
```

> MNO JKL MNO GHI 

9. 打印出所有的“水仙花数”，所谓“水仙花数”是指一个三位数，其各位数字立方和等于该数本身。例如，153是一水仙花数，因为 $153=1^3+5^3+3^3$

```c
#include <stdio.h>

// 水仙花数
int main() {
    for (int i = 100; i <= 999; i++) {
        // 百位
        int b = i / 100;
        // 十位
        int s = i / 10 - b * 10;
        // 个位
        int g = i % 10;
        if (i == (g * g * g + s * s * s + b * b * b)) {
            printf("%d是水仙花数\n", i);
        }
    }
}
```

10. 写一段C程序，该程序从键盘读入一浮点数给x ,再计算$1+x+\frac{x^2}{2!}+\frac{x^3}{3!}+...+\frac{x^{10}}{10!}$的值并输出。

```c
#include <stdio.h>
#include <math.h>

int JieCheng(int x) {
    if (x < 0) {
        return 0;
    }
    int y = 1;
    for (int i = 1; i <= x; i++) {
        y *= i;
    }
    return y;
}

int main() {
    double x = 0;
    double sum = 1;
    scanf("%lf", &x);
    for (int i = 1; i <= 10; i++) {
        sum += pow(x, i) / JieCheng(1);
    }
    printf("%lf", sum);
    return 0;
}
```

11. 自己编写一个函数Inverse实现字符串逆序存放的功能，在主函数中，从键盘任意输入一个字符串（可以有空格），调用函数Inverse实现字符串逆序存放，然后打印逆序存放后的字符串。

```c
#include <stdio.h>
#include <string.h>
#define MaxSize 10000

void Inverse(char *str) {
    char *p = str + strlen(str) - 1;
    char t;
    while (str < p) {
        t = *p;
        *p-- = *str;
        *str++ = t;
    }
}

int main() {
    char str[MaxSize];
    scanf("%s", str);
    printf("反转之前字符串为%s\n", str);
    Inverse(str);
    printf("反转之后字符串为%s\n", str);
}
```

