Tagged Pointer顾名思义the pointer be tagged，即`被标记的指针`，可以理解为被标记的特殊指针。
它伴随着iPhone 5s的64位A7处理器出现。为了解决一些较小数据的创建和访问效率，以及内存占用问题，如`NSString`、`NSNumber`、`NSDate`。

32位CPU下一个指针占用32 bit（4个字节）的内存空间，64位CPU下一个指针占用64 bit（8个字节）的内存空间。即仅CPU位数的变化，会导致指针占用内存的翻倍。

## 关于NSNumber
32 bit的存储空间能保存的最大数字是2^32=2147483648，64位能保存的最大数字即(2^32)^2，能覆盖绝大部分使用场景。因此如果将数字保存在指针空间内部，并对指针做个特殊标记，即一个8位的指针里保存了一个有效的数字形成了一个NSNumber对象。

32位下NSNumber对象内存模型，假设数字小于2^32
> NSNumber指针 + 值 + isa指针 = 4 + 4 + 4 = 12字节

64位非Tagged pointer下的内存模型，假设数字小于2^32
> NSNumber指针 + 值 + isa指针 = 8 + 8 + 8 = 24字节

Tagged pointer下的内存模型
> NSNumber标记 + NSNumber数据 = 8字节

```Objective-C
NSNumber * num = @1;
NSNumber * num1 = @2;

NSNumber * num2 = @0xfffffffffffff; // 13个16位数字
NSNumber * num3 = @0xffffffffffffff;// 14个16位数字

NSLog(@"%p --- %@ ", num, num);
NSLog(@"%p --- %@ ", num1, num1);
NSLog(@"%p --- %@ ", num2, num2);
NSLog(@"%p --- %@", num3, num3);

// 2020-07-30 14:11:43.378731+0800 Test[83181:5340955] 0x4b34d65709cfa21f --- 1
// 2020-07-30 14:11:43.378815+0800 Test[83181:5340955] 0x4b34d65709cfa11f --- 2
// 2020-07-30 14:11:43.378865+0800 Test[83181:5340955] 0x44cb29a8f6305c0f --- 4503599627370495
// 2020-07-30 14:11:43.378911+0800 Test[83181:5340955] 0x1006419f0 --- 72057594037927935
```
由打印的结果可知，64位CPU中保留了3 * 4 = 12 bit作为NSNumber的标记位。

---

## 关于NSSTring
一个ASCII码占用8位即1字节，通常情况下64位CPU一个指针的地址空间内最多可以保存8个ASCII编码的字符，然而我们发现实际上却可以超过8个。

```Objective-C
NSString * str = [NSString stringWithFormat:@"a"];
NSString * str1 = [NSString stringWithFormat:@"mmmmmmmm"];

NSString * str2 = [NSString stringWithFormat:@"ooooooooooo"]; // 11个字符
NSString * str3 = [NSString stringWithFormat:@"abcdefghig"];  // 10个字符

NSLog(@"%p --- %@", str, str);
NSLog(@"%p --- %@", str1, str1);
NSLog(@"%p --- %@", str2, str2);
NSLog(@"%p --- %@", str3, str3);

// 2020-07-30 14:22:56.688467+0800 Test[83610:5352001] 0xf4c009c12d1f6f91 --- a
// 2020-07-30 14:22:56.688603+0800 Test[83610:5352001] 0xf4d86847357e8801 --- mmmmmmmm
// 2020-07-30 14:22:56.688700+0800 Test[83610:5352001] 0xf8a311071c936d31 --- ooooooooooo
// 2020-07-30 14:22:56.688809+0800 Test[83610:5352001] 0x100728660 --- abcdefghig
```
实际对字符的编码方式并非只有ASCII，还有6 bit和5 bit的编码方式，针对最常用的字符如e,i,l,o,t等使用5 bit编码方式，其他较不常用的使用6 bit和ASCII的编码方式。

针对string的Tagged pointer保留了一个字节的标记为，实际上能保存数据的位数有8 * 7 = 56位，因此最多能保存11个5 bit编码的字符，或最多9个6 bit编码的字符。

[参考资料](https://www.programmersought.com/article/4849908018/)
