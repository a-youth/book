# 字符串操作函数

头文件: `<string.h>`
> s 与 t 为 `char *` 类型 , c 与 n 为 `int` 类型

- strcat(s, t)      将t指向的字符串连接到s指向的字符串的末尾
- strncat(s, t, n)  将t指向的字符串中前n个字符连接到s指向的字符串的末尾
- strcmp(s, t)      根据s指向的字符串小于(s<t), 等于(s==t) 或大于(s>t), t指向的字符串的不同情况, 分别返回负整数,
                    0, 或正整数
- strcpy(s,t)       将t指向的字符串复制到s指向的位置
- strncpy(s, t, n)  将t指向的字符串中前n个字符复制到s指向的位置
- strlen(s)         返回s指向的字符串的长度
- strchr(s,c)       在s指向的字符串中查找C, 若找到则返回指向他第一次出现的位置的指针,否则返回null,
- strrchr(s,c)      在s指向的字符串中查找c, 若找到, 则返回指向他最后一次出现位置的指针, 否则返回null