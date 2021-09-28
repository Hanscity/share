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
    call show_screen    ; 屏幕显示

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

