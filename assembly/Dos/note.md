# NOTE




## 指令

```assembly
mov 
add
inc
sub
dec
mul
div

push    ; push，pop 的操作单位是 字，且可以直接操作内存，普通寄存器，段寄存器
pop
loop 
jcxz
jmp
call
ret
retf

adc
sbb
cmp
je
jne
jb
jnb
ja
jna
movsb
movsw
rep

pushf
popf

iret




```



## 小知识点



```assembly
db    ; data byte, 8 字节
dw    ; data word, 字型数据，16 字节
dd    ; double data word, 双字型数据， 32 字节
nop    ; nop 的机器码占一个字节。（P176）

mul    ; 如果是 16 位，一个默认在 AX 中，另一个放在 16 位 reg 或者内存子单元中。
       ; 结果高位默认在 DX 中存放，低位在 AX 中存放。(P199)
  
```





