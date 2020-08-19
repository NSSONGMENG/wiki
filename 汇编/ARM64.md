
# ARM64常用的汇编指令
ARM指令集是以32位二进制编码方式给出的，大部分的指令编码定义了第一、第二和目的操作数、影响的标志位以及每条指令所对应的不同功能实现。

28--31位是条件码；21--24位为操作码；12--19位寄存器编号

# 寄存器
ARM架构不同于Intel架构，不存在8086中涉及到的CS，DS，ES，SS四个段地址寄存器。

## 通用寄存器
ARM64有31个通用寄存器，按64位使用时以`X`开头（例如：X0、X1等），按32位使用时以`W`开头。

- X0  用于存放函数的返回值
- X30 为`LR`，保存函数的返回地址，即`ret`（return）后的下一条待执行指令
- FP（X29） 栈底指针寄存器，用于保存栈的地址（栈底指针）

## SP（X31）
栈顶指针寄存器，任意时刻都指向栈顶指针

## PC
指令指针寄存器，用于保存下一条指令地址（类似8086中的`ip`）

## 浮点数寄存器
浮点数的存储和运算存在特殊性，专门提供31个浮点数寄存器用来处理浮点数。

- 64位D0--D31
- 32位S0--S31

## 向量寄存器
为支持向量运算
128位，V0--V31

---

### 传送指令
#### MOV   X1, X0
将寄存器x0的值传送到x1

(MOV Xd|SP, Xn|SP		Move (extended register): alias for ADD Xd|SP,Xn|SP,#0, but only when one or other of the registers is SP. In other cases the ORR Xd,XZR,Xn instruction is used.)

#### LDR    X5，[X6，#0x08]
将`(X6)+0x08`作为偏移地址指向的双字数据传送到X5寄存器

(LDR Xt, addr		Load Register (extended): loads a doubleword from memory addressed by addr to Xt.)

#### STR X0, [SP, #0x08]
X0寄存器的数据传送到`(SP) + 0x08`作为偏移地址指向的存储空间

(STR Xt, addr		Store Register (extended): stores doubleword from Xt to memory addressed by addr.)

#### LDP  x29, x30, [sp, #0x10]
`ARM64去除了栈操作的push和pop指令`

将`(sp) + 0x10`指向的栈内存中的数据分别写入x29, x30中

(LDP Xt1, Xt2, addr		Load Pair Registers (extended): loads two doublewords from memory addressed by addr to Xt1 and Xt2

#### STP  x29, x30, [sp, #0x10]
将两个双字数据写入`(sp) + 0x10`指向的栈内存空间

(STP Xt1, Xt2, addr 	Store Pair Registers (extended): stores two doublewords from Xt1 and Xt2 to memory addressed by addr.)

---
### 算术运算指令
#### ADD   X0，X1，X2
寄存器X1和X2的值相加后传送到X0

(ADD Xd, Xn, Xm{, ashift #imm}		Add (extended register): Xd = Xn + ashift(Xm, imm).)

#### SUB    X0，X1，X2
寄存器X1和X2的值相减后传送到X0

(SUB Xd, Xn, Xm{, ashift #imm}		Subtract (extended register): Xd = Xn - ashift(Xm, imm).)

---
### 逻辑运算指令
#### AND    X0，X0，#0xF
X0的值与0xF相位与后的值传送到X0

(AND Xd, Xn, Xm{, lshift #imm}		Bitwise AND (extended register): Xd = Xn AND lshift(Xm, imm).)

#### ORR    X0，X0，#9
X0的值与9相位或后的值传送到X0

(ORR Xd, Xn, Xm{, lshift #imm}		Bitwise inclusive OR (extended register): Xd = Xn OR lshift(Xm, imm).)

#### EOR    X0，X0，#0xF
X0的值与0xF相异或后的值传送到X0

(EOR Xd|SP, Xn, #bimm64		Bitwise exclusive OR (extended immediate): Xd|SP = Xn EOR bimm64.)

---
## 转移指令
#### CBZ  
Compare and Branch Zero，如果结果为零（Zero）就转移（只能跳到后面的指令）

(CBZ Xn, label Compare and Branch Zero (extended): conditionally jumps to label if Xn is equal to zero.)

#### CBNZ
比较，如果结果非零（Non Zero）就转移（只能跳到后面的指令）

(CBNZ Xn, label 		Compare and Branch Not Zero (extended): conditionally jumps to label if Xn is not equal to zero.)

#### CMP
比较指令，相当于SUBS，影响程序标志寄存器的零标志位（ZF）和进位标志位（CF）

#### B/BL
绝对跳转指令，后跟`.EQ`、`.LT`等条件，返回地址保存到LR（X30）

(BL label		Branch and Link: unconditionally jumps to pc-relative label, writing the address of the next sequential instruction to register X30.)

```
BL 标号：跳转到标号处执行
B.LT 标号：比较结果是小于（less than），执行标号，否则不跳转；
B.LE 标号：比较结果是小于等于（less than or equal to），执行标号，否则不跳转；
B.GT 标号：比较结果是大于（greater than），执行标号，否则不跳转；
B.GE 标号：比较结果是大于等于（greater than or equal to），执行标号，否则不跳转；
B.EQ 标号：比较结果是等于，执行标号，否则不跳转；
B.HI 标号：比较结果是无符号大于，执行标号，否则不跳转；
```


#### RET
子程序返回指令，返回地址默认保存在LR（X30）

(`RET {Xm}` 		Return: jumps to register Xm, with a hint to the CPU that this is a subroutine return. An assembler shall default to register X30 if Xm is omitted.)
