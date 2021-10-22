# Assembly's Three Part

## 内中断



#### 除法溢出，int 0

可能是版本的原因，当前 WIN10 上的 DOS 环境，发生除法溢出（ 0 号中断），并未显示 Divide Overflow。

```assembly
mov ax, 1000h
mov dx, 1
mov bx, 1
div bx
```

显示的是 ```  F000:1060  FE38  ??? [BX+SI]```

查看中断向量表  ``` -d 0000:0000 f ```  显示的是  ``` 60 10 00 F0  .. ..```

然后 ``` -u F000:1060```  查看之。





### 12.11 单步中断

标志寄存器 TF，IF 用来控制单步中断。



### 12.12 响应中断的特殊情况

特殊情况之栈的操作：

```assembly
assume cs:code

datasg segment
    db 'conversation',0
datasg ends

code segment
			
begin:  
    
    mov ax, 1000h
    mov ss, ax
    mov ax, 100
    mov sp, 100h

    mov ax, 2000h
    mov ss, ax
    mov sp, 200h

    mov ax, 4c00h
    int 21h
    
code ends
end begin
```





#### 实验 12：编写 0 号中断的处理程序，在屏幕中间展示 "divide error!" ，然后返回到 DOS

代码如下：

```assembly
assume cs:code

code segment
			
begin:  
    
    call copy_int0_into_interrupt
    call set_interrupt_in0
    call test_overflow
    
    mov ax, 4c00h
    int 21h

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 测试除法溢出的情况--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

test_overflow:
    push ax
    push bx
    pushf

    mov ax, 1000h
    mov bh, 1
    div bh

    popf
    pop bx
    pop ax

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 测试除法溢出的情况--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_int0_into_interrupt:
    ; 将 do0 程序复制到 0000:200h
    ; 0000:200h ~ 0000:03ffh 的中断向量表的空间是空的
    ; 这里面有一个非常重要的细节，在 debug 模式中 0000:200h
    ; 需要写成 -d 0000:0200 , 如果写成 -d 0000:200 就错了
    ; 写到这个区域的内存，如果修改了代码，需要再次启动，否则会有干扰

    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset do0
    mov di, 200h
    mov cx, offset do0end - offset do0
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置除法溢出的中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt_in0:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[0*4], 200h
    mov word ptr es:[0*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置除法溢出的中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 除法溢出的错误展示--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

do0: 
    ; 这里不能设置 pushf，因为中断程序会自动设置
    push ax
    push ds
    push si
    push es
    push di
    push cx
    
    jmp short do0start
    db "divide error!",0    ; 将中断程序需要显示的字符串写入内存

do0start:
    mov ax, cs
    mov ds, ax
    mov si, 208h    ; 设置 ds:si 指向字符串
                    ; push 占用一个字节，jmp short 占用两个字节 

    mov ax, 0b800h
    mov es, ax
    mov di, 12*160+36*2    ; 设置 es:di 指向显存空间的中间位置

    mov ch, 0
s:  
    mov cl, [si]
    jcxz str_end
    mov es:[di], cl
    mov byte ptr es:[di+1], 24h
    inc si
    add di, 2
    loop s

str_end:

    pop cx
    pop di
    pop es
    pop si
    pop ds
    pop ax

    ;iret    ; iret 为啥没有使程序正常返回(P239) ??
    mov ax, 4c00h    ; 如果在这里写上关闭程序,也没有真正的关闭，不知为何??
    int 21h

do0end:
    nop

    
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 除法溢出的错误展示--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    
code ends
end begin
```



这个练习，体现了这一章的所有的知识点。执行中断程序，是因为 CS:IP 指向了中断程序，这一点贯彻了汇编的核

心思想（CS:IP 指向程序执行的地方），从未改变。控制 CS:IP 的指向，是CPU 的默认设置，是中断程序接收了中

断信号，去中断向量表中寻找，然后自动设置之，程序无法控制。但是程序可以设置中断向量表的指向，从而可以

按照人为的意愿来设置中断程序。







## 第 13 章 int 指令

 int 指令基本统一了内中断的所有情况嘛，除法溢出就是 int 0. 当然单步中断，暂且视为特例。

代码验证如下：

```
assume cs:code

code segment
			
begin:  
    
    call copy_int0_into_interrupt
    call set_interrupt_in0
    int 0
    ;call test_overflow
    
    mov ax, 4c00h
    int 21h

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 测试除法溢出的情况--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

test_overflow:
    push ax
    push bx
    pushf

    mov ax, 1000h
    mov bh, 1
    div bh

    popf
    pop bx
    pop ax

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 测试除法溢出的情况--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_int0_into_interrupt:
    ; 将 do0 程序复制到 0000:200h
    ; 0000:200h ~ 0000:03ffh 的中断向量表的空间是空的
    ; 这里面有一个非常重要的细节，在 debug 模式中 0000:200h
    ; 需要写成 -d 0000:0200 , 如果写成 -d 0000:200 就错了
    ; 写到这个区域的内存，如果修改了代码，需要再次启动，否则会有干扰

    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset do0
    mov di, 200h
    mov cx, offset do0end - offset do0
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置除法溢出的中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt_in0:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[0*4], 200h
    mov word ptr es:[0*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置除法溢出的中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 除法溢出的错误展示--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

do0: 
    ; 这里不能设置 pushf，因为中断程序会自动设置
    push ax
    push ds
    push si
    push es
    push di
    push cx
    
    jmp short do0start
    db "divide error!",0    ; 将中断程序需要显示的字符串写入内存

do0start:
    mov ax, cs
    mov ds, ax
    mov si, 208h    ; 设置 ds:si 指向字符串
                    ; push 占用一个字节，jmp short 占用两个字节 

    mov ax, 0b800h
    mov es, ax
    mov di, 12*160+36*2    ; 设置 es:di 指向显存空间的中间位置

    mov ch, 0
s:  
    mov cl, [si]
    jcxz str_end
    mov es:[di], cl
    mov byte ptr es:[di+1], 24h
    inc si
    add di, 2
    loop s

str_end:

    pop cx
    pop di
    pop es
    pop si
    pop ds
    pop ax

    ;iret    ; iret 为啥没有使程序正常返回(P239) ??
    mov ax, 4c00h    ; 如果在这里写上关闭程序,也没有真正的关闭，不知为何??
    int 21h

do0end:
    nop

    
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 除法溢出的错误展示--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    
code ends
end begin   
```

 int 0 和调用 call test_overflow 结果一致。（我的 DOS 原生的除法溢出中断程序没有正常执行，所以用自己写的）





### 13.2 编写供应用程序调用的中断例程

#### 求 2*3456^2， 参数：(ax) = 要计算的数据，返回值：dx, ax 中存放结果的高 16 位和低 16 位

代码如下：

```assembly
assume cs:code

code segment
			
begin:  
    
    call copy_into_interrupt
    call set_interrupt
    int 7ch
    add ax, ax
    adc dx, dx
    
    mov ax, 4c00h
    int 21h

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset real_program
    mov di, 200h
    mov cx, offset real_program_end - offset real_program
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[7ch*4], 200h
    mov word ptr es:[7ch*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码：计算平方--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

real_program: 
    ; 这里不方便 push 用到的寄存器，因为寄存器用来保存值了
    mov ax, 3456
    mul ax
    iret    ; 这里的 iret 可以正常执行，说明我的理解是正确的(P252)

real_program_end:
    nop
    
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码：计算平方--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    
code ends
end begin   
```



#### 编写，安装中断 7ch 的中断例程，将 data 段中的字符串转化为大写

```assembly
datasg segment
    db 'conversation',0
datasg ends
```



代码如下：

```assembly
assume cs:code

datasg segment
    db 'conversation',0
datasg ends

code segment
			
begin:  
    
    call copy_into_interrupt
    call set_interrupt
    int 7ch    ; 调用指定的中断

    mov ax, 4c00h
    int 21h

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset real_program
    mov di, 200h
    mov cx, offset real_program_end - offset real_program
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[7ch*4], 200h
    mov word ptr es:[7ch*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码：转化为大写--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

real_program: 
    push ax
    push ds
    push bx
    push cx

    mov ax, datasg
    mov ds, ax
    mov bx, 0

    mov ch, 0
turn_start:
    mov cl, ds:[bx]
    jcxz turn_end
    and byte ptr ds:[bx], 11011111b     ; 等同于以下两条命令 
                                        ; and al, 11011111b
                                        ; mov ds:[bx], al
    inc bx
    loop turn_start

turn_end:
    pop cx
    pop bx
    pop ds
    pop ax
    iret

real_program_end:
    nop

    
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码：转化为大写--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    
code ends
end begin
```



### 13.3 对 int，iret 和栈的深入理解

在屏幕中间显示 80 个 “ ! ”，编写 int 7ch 程序来代替 loop 功能：

```assembly
assume cs:code

datasg segment
    db 16 dup (0)
datasg ends

code segment
begin:  


    call set_interrupt    ; 设置中断向量表
    call copy_into_interrupt    ; 将中断程序复制到中断向量表指定的位置
                                ; 开始可以使用 int 7ch 程序
    call show_screen    ; 在屏幕中间显示 80 个 “ ! ”

    mov ax, 4c00h
    int 21h


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 屏幕展示--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

show_screen:

    push ax
    push es
    push di
    push ds

    mov ax, 0b800h
    mov es, ax
    mov di, 160*12

    mov ax, datasg
    mov ds, ax
    mov ax, offset s - offset se
    mov ds:[0], ax    ; 变量（差值）保存在内存空间中，而不用寄存器来保存和来传值，这样的程序更为结构化
    mov cx, 80

    s:
        mov byte ptr es:[di], '!'
        add di, 2
        int 7ch
        ;loop s

    se: 
        nop

    pop ds
    pop di
    pop es
    pop ax

    ret
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 屏幕展示--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 7ch --start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

lp: 
    push bp
    push ax
    push ds

    mov ax, datasg
    mov ds, ax
    mov ax, ds:[0]

    mov bp, sp
    dec cx
    jcxz lp_ret
    add ss:[bp+6], ax    ; 6 = 3 * 2，栈操作的最小单位是字单元
                         ; 但是栈指针的指向还是符合内存单位的标注或是查找的

lp_ret:
    pop ds
    pop ax
    pop bp
    iret
lp_end:
    nop

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 7ch --end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[7ch*4], 200h
    mov word ptr es:[7ch*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset lp
    mov di, 200h
    mov cx, offset lp_end - offset lp
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

code ends
end begin
```





### 13.6 BIOS 中断例程应用



```assembly
assume cs:code

code segment
begin:  


    mov ah, 2    ; 置光标
    mov bh, 0    ; 第 0 页
    mov dh, 10    ; dh 中放行号
    mov dl, 9    ; dl 中放列号(如果单独执行上面这一段，光标所在的位置不对；
                 ; 如果继续执行了下面的这一段中断程序，那么光标的位置和展示的字符都是对的)
    int 10h    ; 调用 BIOS 提供的中断例程

    mov ah, 9     ; 在光标位置显示字符
    mov al, 'a'    ; 字符
    mov bl, 11001010b    ; 颜色属性
    mov bh, 0    ; 第 0 页
    mov cx, 3    ; 字符重复个数
    int 10h    ; 调用 BIOS 提供的中断例程

    
    mov ax, 4c00h
    int 21h


code ends
end begin
```



#### 实验 13 编写，应用中断例程

一：编写并安装 int 7ch 中断例程，功能为显示一个用 0 结束的字符串，中断例程安装在 0:200 处。

参数： (dh) = 行号， (dl) = 列号， (cl) = 颜色，ds:si 指向字符串首地址。

```assembly
assume cs:code

datasg segment
    db 'conversation',0
datasg ends

code segment
			
begin:  
    
    call copy_into_interrupt
    call set_interrupt

    mov dh, 10
    mov dl, 10
    mov cl, 2
    mov ax, datasg
    mov ds, ax
    mov si, 0

    int 7ch    ; 调用指定的中断

    mov ax, 4c00h
    int 21h


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码：--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

real_program: 
    push ax
    push bx
    push dx
    push di
    push es
    push ds

    mov ax, 0b800h
    mov es, ax
    mov al, dh    ; dh 中保存着行号
    add al, 3    ; g 命令之后会向上移动 3 行
    mov ah, 160    ; 一行 80 个字符，每个字符需要两个字节来保存
    mul ah    ; 如果是 8 位乘法，默认的一个放在 AL 中
              ; 结果默认放在 AX 中
    mov di, ax    ; 将第 8 行的计算结果赋值给 di
    mov al, dl
    mov ah, 2
    mul ah    ; 计算列号
    add di, ax

turn_start:
    mov al, ds:[si]
    mov bl, 0
    cmp al, bl
    je turn_end

    mov al, ds:[si]
    mov es:[di], al    ; 显存中保存值
    mov es:[di+1], cl    ; 显存中保存颜色
    inc si
    add di, 2

    jmp turn_start

turn_end:
    pop ds
    pop es
    pop di
    pop dx
    pop bx
    pop ax
    iret

real_program_end:
    nop

    
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码：--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset real_program
    mov di, 200h
    mov cx, offset real_program_end - offset real_program
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[7ch*4], 200h
    mov word ptr es:[7ch*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    
    
code ends
end begin
```



二：编写并安装 int 7ch 中断例程，功能为完成 loop 指令的功能。

参数： (cx) = 循环次数， (bx) = 位移。

```assembly
assume cs:code

code segment
begin:  
	
    call set_interrupt    ; 设置中断向量表
    call copy_into_interrupt    ; 将中断程序复制到中断向量表指定的位置
                                ; 开始可以使用 int 7ch 程序
    mov ax, 0b800h
    mov es, ax
    mov di, 160*12

    mov bx, offset s - offset se    ; 传递位移
    mov cx, 80    ; 传递循环次数
    s:
        mov byte ptr es:[di], '!'
        add di, 2
        int 7ch
    se: 
        nop

    mov ax, 4c00h
    int 21h


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 7ch --start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

lp: 
    push bp
    push ds

    mov bp, sp
    dec cx
    jcxz lp_ret
    add ss:[bp+4], bx    ; 4 = 2 * 2，栈操作的最小单位是字单元
                         ; 但是栈指针的指向还是符合内存单位的标注或是查找的

lp_ret:
    pop ds
    pop bp
    iret
lp_end:
    nop

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 7ch --end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax
    mov word ptr es:[7ch*4], 200h
    mov word ptr es:[7ch*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset lp
    mov di, 200h
    mov cx, offset lp_end - offset lp
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

code ends
end begin
```



三：下面的程序，分别在屏幕的第 2， 4， 6， 8 行显示 4 句英文诗

```assembly
assume cs:code

code segment
    s1: db 'Good,better,best','$'
    s2: db 'Never let it rest','$'
    s3: db 'Till good is better','$'
    s4: db 'And better,best.','$'
    s: dw offset s1,offset s2,offset s3,offset s4
    row: db 2,4,6,8

start:  
    mov ax, cs
    mov ds, ax
    mov bx, offset s
    mov si, offset row
    mov cx, 4

ok:
    mov bh, 0
    mov dh, [si]    ; 光标停留的位置
    mov dl, 0
    mov ah, 2
    int 10h

    mov dx, [bx]    ; dx 指向字符串首地址
    mov ah, 9
    int 21h

    inc si
    add bx, 2
    loop ok

    mov ax, 4c00h
    int 21h
	
    
code ends
end start
```



### 第 14 章 端口

CPU 可以直接读写以下 3 个地方的数据：

1. CPU 内部的寄存器
2. 内存单元
3. 端口 



#### 14.1 端口的读写

注意，在 in 和 out 指令中，只能使用 ax 或者 al 来存放从端口中读入的数据或要发送到端口中的数据。



#### 14.2 CMOS RAM 芯片

监测点 14.1 

1. 编程，读取 CMOS RAM 的 2 号单元的内容

   ```assembly
   mov al, 2    ; 使用 al 来保存将要发送到端口中数据
   out 70h, al    ; 向地址端口 70h 发送 al 中的数据
   in al, 71h    ; 向数据端口 71h 读取内容存放在 al 中
   ```

   

2. 编程，向 CMOS RAM 的 2 号单元写入 0

   ```assembly
   mov al, 2    ; 使用 al 来保存将要使用的数据
   out 70h, al    ; 向地址端口 70h 发送 al 中的数据
   mov al, 0    ; 使用 al 来保存将要发送到端口中数据
   out 71h, al    ; 向数据端口 71h 写入存放在 al 中的数据
   ```



#### 14.3 shl 和 shr 指令

检测点 14.2：编程，用加法和移位指令计算 (ax) = (ax) * 10。

```assembly
mov bx, ax
shl bx, 1
mov cl, 3
shl ax, cl
add ax, bx
```





#### 14.4 CMOS RAM 中存储的时间信息

编程，在屏幕中间显示当前的月份：

```assembly
assume cs:code

code segment
begin:  

    mov al, 8
    out 70h, al
    in al, 71h

    mov ah, al
    mov cl, 4
    shr ah, cl
    and al, 00001111b

    add ah, 30h
    add al, 30h

    mov bx, 0b800h
    mov es, bx
    mov byte ptr es:[160*12 + 40*2], ah
    mov byte ptr es:[160*12 + 40*2 + 1], 11001010b ;颜色
    mov byte ptr es:[160*12 + 40*2 + 2], al
    mov byte ptr es:[160*12 + 40*2 + 3], 11001010b ;颜色
    
    mov ax, 4c00h
    int 21h


code ends
end begin
```





#### 实验 14 访问 CMOS RAM

编程， 以 ”年/月/日 时:分:秒“ 的格式，显示当前的日期和时间。

方法一：

```assembly
assume cs:code

datasg segment
    db '11/11/11 11:11:11$' ;预设字符串
datasg ends

portsg segment
    db 9,8,7,4,2,0 ;端口时间地址列表
portsg ends


code segment
begin:  
    mov cx, 6
    mov ax, datasg
    mov ds, ax
    mov ax, portsg
    mov es, ax
    mov di, 0
    mov si, 0
s:
    mov al, es:[si]
    out 70h, al
    in al, 71h

    mov ah, al
    shr al, 1
    shr al, 1
    shr al, 1
    shr al, 1    ; 连续写四个 shr al, 1 是不想占用寄存器 cx。
    and ah, 00001111b    ; 高低位调转，例如展示'21'，在内存中 '2' 是低位。

    add ah, 30h
    add al, 30h
    mov ds:[di], ax

    add di, 3
    inc si
    loop s
    
    mov ah, 2    ; 置光标
    mov bh, 0    ; 第 0 页
    mov dh, 12    ; dh 中放行号
    mov dl, 30    ; dl 中放列号
    int 10h ;

    mov dx, 0    ; ds:dx 指向字符串的首地址 datasg:0
    mov ah, 9    ; 在光标位置显示字符串
    int 21h    ; DOS 为程序员提供了很多的中断例程都包含在 int 21h 中断例程中


    mov ax, 4c00h
    int 21h


code ends
end begin
```





### 第 15 章 外中断

外中断和内中断大同小异，内中断的中断类型码在 CPU 内部产生，外中断的中断类型码来自 CPU 外部。



### 15.4 编写 int 9 中断例程

编程： 在屏幕中间依次显示 “a~z”，并可以看清

```assembly
assume cs:code

code segment
begin:  
    mov ax, 0b800h
    mov es, ax
    mov ah, 'a'
s:
    mov es:[160*12 + 40*2], ah
    call delay
    inc ah
    cmp ah, 'z'
    jna s

    mov ax, 4c00h
    int 21h

delay:
    push ax
    push dx

    mov dx, 10h
    mov ax, 0
s1: 
    sub ax, 1
    sbb dx, 0
    cmp ax, 0
    jne s1
    cmp dx, 0
    jne s1

    pop dx
    pop ax
    ret

code ends
end begin
```



编程： 在屏幕中间依次显示 “a~z”，并可以看清，如果是 Esc 的扫描码，改变显示的颜色后返回

```assembly
assume cs:code

stack segment
    db 128 dup (0)
stack ends

data segment
    dw 0, 0
data ends


code segment
begin:  
    mov ax, stack
    mov ss, ax
    mov sp, 128

    mov ax, data
    mov ds, ax

    mov ax, 0
    mov es, ax

    push es:[9*4]
    pop ds:[0]
    push es:[9*4+2]
    pop ds:[2]    ; 将原来的 int 9 中断例程的入口地址保存在 ds:0, ds:2 单元中

    cli    ; 如果在执行设置 int9 中断例程的段地址和偏移地址的指令之间发生了键盘中断
           ; CPU 将转去一个错误的地址
    mov word ptr es:[9*4], offset int9
    mov es:[9*4+2], cs    ; 在中断向量表中设置新的 int 9 中断例程的入口地址
    sti    ; 恢复标志位 IF=1 

    mov ax, 0b800h
    mov es, ax
    mov ah, 'a'
s:
    mov es:[160*12 + 40*2], ah
    call delay
    inc ah
    cmp ah, 'z'
    jna s

    mov ax, 4c00h
    int 21h

delay:
    push ax
    push dx

    mov dx, 10h
    mov ax, 0
s1: 
    sub ax, 1
    sbb dx, 0
    cmp ax, 0
    jne s1
    cmp dx, 0
    jne s1

    pop dx
    pop ax
    ret

;-------------------------------------------------------------
;新的 int 9 中断例程--start
;-------------------------------------------------------------

int9: 
    push ax
    push bx
    push es

    in al, 60h

    pushf    ; 对 int 指令进行模拟的准备
    call dword ptr ds:[0]    ; 对 int 指令进行模拟，调用原来的 int 9 中断例程

    cmp al, 1
    jne int9ret

    mov ax, 0b800h
    mov es, ax
    inc byte ptr es:[160*12+40*2+1]    ; 将属性值加一，改变颜色

int9ret:
    pop es
    pop bx
    pop ax
    iret


;-------------------------------------------------------------
;新的 int 9 中断例程--end
;-------------------------------------------------------------

code ends
end begin
```



### 15.5 安装新的 int 9 中断例程

编程：在 DOS 下，按 F1 键后改变当前屏幕的显示颜色，其他的键照常处理

```assembly
assume cs:code

stack segment
    db 128 dup (0)
stack ends

data segment
    dw 0, 0
data ends


code segment
begin:  
    mov ax, stack
    mov ss, ax
    mov sp, 128

    mov ax, data
    mov ds, ax

    mov ax, 0
    mov es, ax

    push es:[9*4]
    pop ds:[0]
    push es:[9*4+2]
    pop ds:[2]    ; 将原来的 int 9 中断例程的入口地址保存在 ds:0, ds:2 单元中

    cli    ; 如果在执行设置 int9 中断例程的段地址和偏移地址的指令之间发生了键盘中断
        ; CPU 将转去一个错误的地址
    mov word ptr es:[9*4], offset int9
    mov es:[9*4+2], cs    ; 在中断向量表中设置新的 int 9 中断例程的入口地址
    sti    ; 恢复标志位 IF=1 

    
    mov ax, 1111h    ; 缓冲代码，留作测试。
    mov ax, 4c00h
    int 21h

;-------------------------------------------------------------
;----新的 int 9 中断例程--start
;-------------------------------------------------------------

int9: 
    push ax
    push bx
    push cx
    push es


    in al, 60h
    pushf    ; 对 int 指令进行模拟的准备，保存标志位寄存器
    call dword ptr ds:[0]    ; 对 int 指令进行模拟，调用原来的 int 9 中断例程

    cmp al, 3bh    ; F1 的扫描码为 3bh, B 的扫描码为 30h.
    jne int9ret

    mov ax, 0b800h
    mov es, ax
    mov bx, 1
    mov cx, 2000
s:    ; 问题应该发生在这一段，如果换成其它代码，这里还能有些效果
      ; 或许是 loop 等功能有些问题
    inc byte ptr es:[bx]
    add bx, 2
    loop s
    
int9ret:
    pop es
    pop cx
    pop bx
    pop ax
    iret


;-------------------------------------------------------------
;----新的 int 9 中断例程--end
;-------------------------------------------------------------

code ends
end begin
```

以上程序不仅无法正常显示，即使正确，还有一个错误，中断例程不能保存在安装程序中，因为安装程序返回后地址将丢失。

正确的程序如下所示，按 F1 键将可以一直改变 DOS 屏幕的颜色

```assembly
assume cs:code

code segment
begin:  
	
    call save_int9    ; 保存原来的 int9 程序地址，等待调用
    call set_interrupt    ; 设置中断向量表
    call copy_into_interrupt    ; 将中断程序复制到中断向量表指定的位置
    
    mov ax, 4c00h
    int 21h


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 --start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

lp: 
    push ax
    push bx
    push es

    in al, 60h
    pushf    ; 对 int 指令进行模拟的准备，保存标志位寄存器
    mov bx, 0
    mov es, bx
    call dword ptr es:[200h]    ; 对 int 指令进行模拟，调用原来的 int 9 中断例程

    
    cmp al, 3bh    ; F1 的扫描码为 3bh
    jne int9ret

    mov ax, 0b800h
    mov es, ax
    mov bx, 1
    mov cx, 2000
s:
    inc byte ptr es:[bx]
    add bx, 2
    loop s
    
int9ret:
    pop es
    pop bx
    pop ax
    iret

lp_end:
    nop

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 --end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将 int9 程序换一个地址保存--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

save_int9:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax

    push es:[9*4]
    pop es:[200h]
    push es:[9*4+2]
    pop es:[202h]
    
    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将 int9 程序换一个地址保存--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax

    mov word ptr es:[9*4], 204h    ; 这里稍作改变，保存在 204h 的地方
    mov word ptr es:[9*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset lp
    mov di, 204h
    mov cx, offset lp_end - offset lp
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

code ends
end begin
```



### 实验 15  安装新的 int 9 中断例程

编程：安装一个新的 int 9 中断例程，功能：在 DOS 下，按下 “A” 键后，除非不再松开，如果松开，就显示满屏幕的 “A”，其他的键照常处理

```assembly
assume cs:code

code segment
begin:  
	
    call save_int9    ; 保存原来的 int9 程序地址，等待调用
    call set_interrupt    ; 设置中断向量表
    call copy_into_interrupt    ; 将中断程序复制到中断向量表指定的位置
    
    mov ax, 4c00h
    int 21h


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 --start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

lp: 
    push ax
    push bx
    push es

    in al, 60h    ; 键盘的按下和松开都会触发中断，接受此类中断信息都是 in 指令
    pushf    ; 对 int 指令进行模拟的准备，保存标志位寄存器
    mov bx, 0
    mov es, bx
    call dword ptr es:[200h]    ; 对 int 指令进行模拟，调用原来的 int 9 中断例程

    
    cmp al, 9Eh    ; A 的扫描码为 1Eh, 断码 = 通码 + 80h
    jne int9ret

    mov ax, 0b800h
    mov es, ax
    mov bx, 0
    mov cx, 2000
s:
    mov byte ptr es:[bx], 'A'
    add bx, 2
    loop s
    
int9ret:
    pop es
    pop bx
    pop ax
    iret

lp_end:
    nop

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 真实运行的一段代码程序 --end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将 int9 程序换一个地址保存--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

save_int9:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax

    push es:[9*4]
    pop es:[200h]
    push es:[9*4+2]
    pop es:[202h]
    
    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将 int9 程序换一个地址保存--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

set_interrupt:
    push ax
    push es
    pushf
    
    mov ax, 0
    mov es, ax

    mov word ptr es:[9*4], 204h    ; 这里稍作改变，保存在 204h 的地方
    mov word ptr es:[9*4+2], 0

    popf
    pop es
    pop ax
    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 设置中断向量表--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--start
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

copy_into_interrupt:
    
    push ax
    push ds
    push es
    push si
    push di
    push cx
    pushf

    mov ax, cs
    mov ds, ax
    mov ax, 0
    mov es, ax

    mov si, offset lp
    mov di, 204h
    mov cx, offset lp_end - offset lp
    cld
    rep movsb

    popf
    pop cx
    pop di
    pop si
    pop es
    pop ds
    pop ax

    ret

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; 将中断程序复制到中断向量表指定的位置--end
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

code ends
end begin
```



这类代码的封装，自己还是挺满意的。实验 15 和 上一个编程代码（F1 改变屏幕的颜色）只需要稍作修改即可运行，调用简单，封装完善，分层明显。





## 第 16 章 直接定址表



### 16.1 描述了单元长度的标号

不仅有地址，并且描述了单元长度

#### 监测点 16.1

```
assume cs:code

code segment
    a dw 1, 2, 3, 4, 5, 6, 7, 8
    b dd 0

start:
    mov si, 0
    mov cx, 8
s:
    mov ax, a[si]
    add word ptr b[0], ax
    adc word ptr b[2], 0
    add si, 2
    loop s

    mov ax, 4c00h
    int 21h


code ends
end start
```



### 16.2 在其他段中使用数据标号

assume 指令：如果想在代码段中直接使用数据标号访问数据，则需要用伪指令 assume 将标号所在的段和一个段寄存器联系起来。否则编译器在编译的时候，无法确定标号的段地址在哪一个寄存器中。与此同时，还需要在实际的代码程序中指定。

offset 操作符：功能为取得某一标号的偏移地址。

seg 操作符：功能为取得某一标号的段地址。



#### 监测点 16.2

```assembly
assume cs:code,es:data

data segment
    a db 1, 2, 3, 4, 5, 6, 7, 8
    b dw 0
data ends

code segment
start:
    mov ax, data
    mov es, ax
    mov si, 0
    mov cx, 8
s:
    mov al, a[si]
    mov ah, 0
    add b[0], ax
    inc si
    loop s

    mov ax, 4c00h
    int 21h


code ends
end start
```





### 16.3 直接定址表

