# Assembly's Two Part


## 第 7 章 更灵活的定位内存地址的方法


### 7.1 and 和 or 指令

- and 指令：逻辑与指令，按位进行与运算（两个同时为真，才为真；反之，有一个为假，就为假）

  ```
  mov al,01100011B
  and al,00111011B
  ;al =  00100011B
  ```

  

- or 指令：逻辑或指令，按位进行或运算（有一个为真，即是真；反之同时为假，就为假）

  ```
  mov al,01100011B
  or al, 00111011B
  ;al =  01111011B
  ```

  

#### 扩展：PHP 位运算的应用

- 应用背景：用户三百万，用户字段，当时接近 100个，还在不断扩展；并发比较高，用户信息会同步 redis；在一些特定的场景下，

  会非常的方便数据库的扩展，会节省较多的缓存。

```php
<?php


class Binary
{


    public static $flag_shumei = 1; // 0001
    public static $flag_driver = 2; // 0010

    //判断这个标志位是否被设置
    public function isBitSetFlag($flag,$val)
    {
        return (($val & $flag) === $flag);
    }

    /**
     * Notes: 设置标志位
     * User: ${USER}
     * Date: ${DATE}
     * Time: ${TIME}
     * @param $flag
     * @param $val
     * @param $bool
     * @return mixed
     *
     */
    public function setBitflag($flag,$val,$bool)
    {
        if($bool){
            $val |= $flag;
        }else{
            $flag ^= $flag;
            $val &= $flag;
        }

        return $val;
    }

  
    //初始化二进制的值
    public function initValueBinary()
    {
        return 0;
    }

}





class Activity
{

    /**
     * Notes: 设置二进制位
     * User: ${USER}
     * Date: ${DATE}
     * Time: ${TIME}
     * @param $data
     * bool,如果设置为开启，则为true ；如果设置为关闭，则为 false;
     */
    public function userBinaryCharacter($data)
    {

        $uid = $data['uid'];
        $type = $data['type'];
        $bool = $data['bool'];
        ## 更新和写入
        $model = new UserBinaryCharacter();
        $res = $model->find()->where(['uid'=>$uid])->asArray()->one();
        $cur_time = date('Y-m-d H:i:s');
        if($res){
            $binary_one = $res['binary_one'];
            //设置二进制位
            $binary_one = UserDAO::getInstance()->setBitflag($type,$binary_one,$bool);
            $data = [
                'binary_one' => $binary_one,
                'update_time' => $cur_time
            ];
            //数据库的更新操作
            if($model::updateAll($data, ['uid'=>$uid])){
                //更新redis
                UserDAO::getInstance()->updateRedisUserInfo($uid,$data);
            };

        }else{
            $binary_one = UserDAO::getInstance()->initValueBinary();
            $binary_one = UserDAO::getInstance()->setBitflag($type,$binary_one,$bool);
            //数据库的写入操作
            $model->uid = $uid;
            $model->binary_one = $binary_one;
            $model->create_time = $cur_time;
            $model->update_time = $cur_time;
            if($model->save()){
                //更新redis
                $data = [
                    'binary_one' => $binary_one,
                    'update_time' => $cur_time
                ];
                UserDAO::getInstance()->updateRedisUserInfo($uid,$data);
            };
        }

    }

    public function judgeUserBinaryCharacter($data)
    {

        $uid = $data['uid'];
        $type = $data['type'];
        $user_info = UserDAO::getInstance()->getUserInfoByUid($uid);
        $bool = false;
        if(isset($user_info['binary_one']) && (UserDAO::getInstance()->isBitSetFlag($type,$user_info['binary_one']))){
            $bool = true;
        }
        return $bool;
    }
  
  
  
}





?>
```



### 7.2 关于 ASCII 码

- 计算机只保存二进制，人类比较喜欢字符串。例如，宝马雕车香满路，一夜鱼龙舞，多么的具象；例如 "a",保存在计算机中是 61H,

  输出到显存，显示部分将 61H 转为 “a” 展示在屏幕上。故，ASCII码更像是一个翻译家，是一个沟通的桥梁。

- 世界上有很多的编码方案，计算机系统中，通常采用 ACSCII 码。（这里的计算机系统应该比较老哈～）



#### 扩展：字符串和编码

> https://www.liaoxuefeng.com/wiki/1016959663602400/1017075323632896

> 由于计算机是美国人发明的，因此，最早只有127个字符被编码到计算机里，也就是大小写英文字母、数字和一些符号，这个编码表被称为`ASCII`编码
>
> 但是要处理中文显然一个字节是不够的，至少需要两个字节，而且还不能和ASCII编码冲突，所以，中国制定了`GB2312`编码，用来把中文编进去。
>
> 你可以想得到的是，全世界有上百种语言，日本把日文编到`Shift_JIS`里，韩国把韩文编到`Euc-kr`里，各国有各国的标准，就会不可避免地出现冲突，结果就是，在多语言混合的文本中，显示出来会有乱码。
>
> 因此，Unicode字符集应运而生。Unicode把所有语言都统一到一套编码里，这样就不会再有乱码问题了。
>
> Unicode标准也在不断发展，但最常用的是UCS-16编码，用两个字节表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）。现代操作系统和大多数编程语言都直接支持Unicode。
>
> 在计算机内存中，统一使用Unicode编码，当需要保存到硬盘或者需要传输的时候，就转换为UTF-8编码。
>
> ...





### 7.3 以字符形式给出的数据

- 代码示下：

  ```assembly
  assume cs:code,ds:data
  
  data segment
      db 'unIX'    ;在内存单元中，一个字符串占用一个内存单元
      db 'foRK'
  data ends
  
  code segment
  			
  start:  
      mov ax,data
      mov ds, ax    ;仅仅是查看，我为啥写这个赋值呢？方便明确的查看段地址，来 debug 数据哈
                    ;例如："ds=0B2D",所以程序从 0B3D 段开始~ 难道这就是 PSP，暂不深究啦~
      
      mov al,'a'
      mov bl,'b'
  
      mov ax,4c00h
      int 21h
  
  
  code ends
  
  
  end start
  ```



### 7.4 大小写转换的问题

- 如下所示：将 data 段中的第一个字符串 “BaSiC” 转化为大写，第二个字符串 “iNfOrMaTioN” 转化为小写：

  ```assembly
  assume cs:code,ds:data
  
  data segment
      db 'BaSiC'
      db 'iNfOrMaTioN'
  data ends
  
  code segment
  			
  start:  
      
      mov ax,4c00h
      int 21h
  
  
  code ends
  
  
  
  end start
  ```

  

- 代码示下：

  ```assembly
  assume cs:code,ds:data
    
      data segment
          db 'BaSiC'
          db 'iNfOrMaTioN'
      data ends
    
      code segment
    			
      start:  
          
          mov ax,data
          mov ds,ax
          mov bx,0
  
          mov cx,5
      s1: 
          mov al,[bx]
          and al,11011111B
          mov [bx],al
          inc bx
      loop s1
  
          mov cx,0bh
      s2:
          mov al,[bx]
          or al,00100000B
          mov [bx],al
          inc bx
      loop s2
  
          mov ax,4c00h
          int 21h
      
    
    code ends
    
    
    
    end start
  ```

  

### 7.5 [bx+idata]

```
;mov ax,[bx+200] 的含义
;数学化的描述为： (ax) = ((ds)*16 + (bx) + 200)

;还可以写成以下三种格式：
; 经测试，这三种格式，在 debug 和 源程序中，都是可以的
; debug 中，-a 输入命令，所有的数据默认就是 16 进制，暂认为没有其它进制
; 源程序中，默认就是十进制，带上 h 表示 16 进制

mov ax,[bx+0c8h]
mov ax,0c8h[bx]
mov ax,[bx].0c8h
```



### 7.6 用 [bx+idata] 的方式进行数组的处理

- 如下所示：将 data 段中的第一个字符串 “BaSiC” 转化为大写，第二个字符串 “MinIx” 转化为小写：

  ```assembly
  assume cs:code,ds:data
  
  data segment
      db 'BaSiC'
      db 'MinIx'
  data ends
  
  code segment
  			
  start:  
      
      mov ax,4c00h
      int 21h
  
  
  code ends
  
  
  end start
  ```

- 根据上面的例子，稍加修改，代码如下：

  ```
  assume cs:code,ds:data
    
      data segment
          db 'BaSiC'
          db 'MinIX'
      data ends
    
      code segment
    			
      start:  
          
          mov ax,data
          mov ds,ax
          mov bx,0
  
          mov cx,5h
      s1: 
          mov al,[bx]
          and al,11011111B
          mov [bx],al
          inc bx
      loop s1
  
          mov cx,5h
      s2:
          mov al,[bx]
          or al,00100000B
          mov [bx],al
          inc bx
      loop s2
  
          mov ax,4c00h
          int 21h
      
    
    code ends
    
    
    end start
  ```

  

- 用 [bx+idata] 的方式，优化如下：

  ```assembly
  assume cs:code,ds:data
    
      data segment
          db 'BaSiC'
          db 'MinIX'
      data ends
    
      code segment
    			
      start:  
          
          mov ax,data
          mov ds,ax
          mov bx,0
  
          mov cx,5h
      s1: 
          mov al,0[bx]
          and al,11011111B
          mov 0[bx],al
  
          mov al,5[bx]
          or al,00100000B
          mov 5[bx],al
  
          inc bx
      loop s1
  
      
  
          mov ax,4c00h
          int 21h
      
    
    code ends
    
    
    end start
  ```

- C 语言的原生实现

     ```c
     #include <stdio.h>
       
     int main()
     {
     
             char a[] = "BaSiC";
             char b[] = "MinIX";
     
             int i=0;
             do{
                     a[i]=a[i] & 0b11011111;
                     b[i]=b[i] | 0b00100000;
                     i++;
             }while(i<5);
     
             printf("%s",a);
             printf("%s",b);
     
     }
     
     ```



- Php 语言的原生实现

  ```php
  <?php
  
  
          $a = 'BaSic';
          $b = 'MinIX';
  
          do{
                  $a[$i] &= 0b11011111;
                  $b[$i] |= 0b00100000;
                  $i++;
  
          }while($i<5);
  
  
          var_dump($a);
          var_dump($b);
  
  ```

  ```
  Fatal error: Uncaught Error: Cannot use assign-op operators with string offsets in /home/ch/php/test001.php:8
  Stack trace:
  #0 {main}
    thrown in /home/ch/php/test001.php on line 8
  
  ```

 

#### [bx+idata] 和高级语言数组的关系

```
C 语言:    a[i],b[i]
汇编语言:   0[bx],5[bx] 
```

- 可以看出，[bx+idata] 的方式为高级语言实现数组提供了便利机制。



### 7.7 SI 和 DI

- si 和 di 是 8086CPU 中和 bx 功能相近的寄存器，si 和 di 不能够分为两个 8 位寄存器来使用

#### 问题 7.2 ：用 si 和 di 实现将字符串 'welcome to masm!' 复制到它后面的数据区中

```assembly
assume cs:code,ds:data
  
    data segment
        db 'welcome to masm!'
        db '................'
    data ends
  
    code segment
  			
    start:  
        
        
        mov ax,4c00h
        int 21h
    
  
  code ends
  
  
  end start
```



- 解答：

```assembly
assume cs:code,ds:data
  
    data segment
        db 'welcome to masm!'
        db '................'
    data ends
  
    code segment
  			
    start:  
        mov ax,data
        mov ds,ax
        mov di,0h

        mov cx,8h
    s1: 
        mov ax,[di]
        mov 10h[di],ax
        add di,2
    loop s1
        
        mov ax,4c00h
        int 21h
    
  
  code ends
  
  
  end start
```



### 7.8 [bx+si] 和 [bx+di]

- 指令 mov ax,[bx+si] 的含义如下：(ax) = ((ds)*16 +(bx)+(di)) ; 也可以表示为 

  ```assembly
  mov ax,[bx][si]
  ```

  

- 一个特别小的注意点分享哈，有时特别容易懵

```asseembly
;Debug 查看内存，结果如下：
;2000:0000 BE 00 06 00 00 00 ...
;((2000*16) + 0000) 等于 00BE 还是  00EB ？
; 结果是 00BE 哈
; 在内存中 00 相对 BE 是高位，在寄存器中，将寄存器表达出来， B 相对于 E 是高位哦~
```



### 7.9 [bx+si+idata] 和 [bx+di+idata]

- 指令 mov ax,[bx+si+idata] 的含义如下：(ax) = ((ds)*16+(bx)+(si)+(idata));也可以表示为：

  ```assembly
  mov ax,[bx+200+si]
  mov ax,[200+bx+si]
  mov ax,200[bx][si]
  mov ax,[bx].200[si]
  mov ax,[bx][si].200
  ```

  

### 7.10 不同的寻址方式的灵活应用

- 通过一个问题的系列来体会 CPU 提供多种寻址方式的用意

#### 编程，将 datasg 段中的每个单词的头一个字母改为大写字母

```assembly
assume cs:codesg,ds:datasg
  
    datasg segment
        db '1. file         '
        db '2. edit         '
        db '2. search       '
        db '2. view         '
        db '2. options      '
        db '2. help         '
    datasg ends
  
    codesg segment
  			
    start:  
        

        mov ax,4c00h
        int 21h
    
  
  codesg ends
  
  
  end start
```



- Answer

```assembly
assume cs:codesg,ds:datasg
  
    datasg segment
        db '1. file         '
        db '2. edit         '
        db '2. search       '
        db '2. view         '
        db '2. options      '
        db '2. help         '
    datasg ends
  
    codesg segment
  			
    start:  
        mov ax,datasg
        mov ds,ax
        mov bx,0h
        mov cx,6h
        
    s1:
        mov ah,3[bx]
        and ah,11011111B
        mov 3[bx],ah
        add bx,10h
    loop s1        

        mov ax,4c00h
        int 21h
    
  
  codesg ends
  
  
  end start
```





#### 编程，将 datasg 段中每个单词改为大写字母

```assembly

assume cs:codesg,ds:datasg
  
    datasg segment
        db 'ibm             '
        db 'dec             '
        db 'dos             '
        db 'vax             '
    datasg ends
  
    codesg segment
  			
    start:  
        mov ax,datasg
        mov ds,ax
        mov bx,0h
        
        mov si,0h
        mov cx,3h
        mov ds:[40h],cx

    s0:
        mov cx,4h
    s1:
        mov ah,[bx+si]
        and ah,11011111B
        mov [bx+si],ah
        add bx,10h
    loop s1        

        inc si
        mov cx,ds:[40h]
        sub cx,si
        inc cx

    loop s0



        mov ax,4c00h
        int 21h
    

  codesg ends
  end start
```

