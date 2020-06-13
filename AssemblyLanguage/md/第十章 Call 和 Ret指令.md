# 第十章 Call 和 Ret指令

`call`和`ret`指令都是转移指令，他们同时修改IP，或同时修改CS和IP。经常被用于**子程序的设计**。

## ret 和 retf

指令的格式: 单独出现，没有任何操作数。

`ret`指令通过栈中的数据，修改IP，从而实现近转移。

`retf`指令通过栈中的数据，修改CS和IP，从而实现远转移。

### ret指令的执行步骤

-   `ip = (ss*16 + sp)` 
-   `sp = sp + 2`

也就是相当于进行了`pop ip`的操作。

### retf指令的执行步骤

-   `ip = (ss*16 + sp)`
-   `sp = sp + 2`
-   `cs = (ss*16 + sp)`
-   `sp = sp + 2`

也就是相当于进行了`pop ip, pop cs`的操作。

## call指令

指令格式：call 标号

功能：转移到对应的标号处执行指令。

### call 指令执行的步骤

-   sp = sp -2
-   (ss*16 + sp) = ip
-   ip = ip + 16位位移

CPU在执行`call 标号`相当于在执行以下的汇编指令。

```asm
push ip
jmp near ptr 标号
```

使用`call far ptr 标号`可以实现段间转移，此时目的地址在指令中。

在执行`call far ptr 标号`相当于在执行

```asm
push cs
push ip
jmp far ptr 标号
```

也可以使用`call 16位reg`的格式，相对于执行

```asm
push ip
jmp 16位reg
```

转移地址还可以在内存当中。

如果我们想要实现段内转移，此时需要1个字的内存空间，可以使用`call word ptr 内存单元`，相当于执行。

```asm
push ip
jmp word ptr 内存单元
```

实现段间转移可以使用`call dword ptr`，相当于执行。

```asm
push ip
push cs
jmp dword ptr 内存单元 ;(高位为cs， 低位为ip)
```

## 配合使用call和ret

我们可以使用`call`和`ret`来实现子程序的机制。（有点儿相当于C语言当中的函数）。子程序的框架如下。

```
标号：
		指令
		ret
```

具有子程序的源程序的框架如下：

```asm
assume cs:code 

code segment
start:
		...
		...
		call sub1 ; 调用子程序1
		...
		...
		
		mov ax, 4c00h
		int 21h
		
sub1:
		...
		call sub2 ; 调用子程序2
		...
		ret ; 子程序1返回
		
sub2:
		...
		...
		ret ; 子程序2返回
code ends
end start
```

其实这就类似这样的C语言代码。

```c
void sub1() {
		...;
		sub2();				// call sub2
   ...;
		return;      // ret
}

void sub2() {
    ...;
    return;				// ret
}

int main() {
    ...;
    sub1();				// call sub1
    
    return;				// mov ax, 4c00h  int 21h
}
```

## mul 指令

`mul`是乘法指令，有一下地方需要注意。

-   两个相乘的数：要么都是8位，要么都是16位。如果是8位，一个默认放在al中，另一个放在8位寄存器或者内存字节单元中。如果是16位，一个默认放在ax中，另一个放在16位寄存器或者内存字单元中。
-   结果：如果是8位乘法，结果默认放在ax中。如果是16位乘法，结果高位默认在dx中存放，低位在ax中存放。

指令格式： `mul reg`或者`mul 内存单元`

例：

```asm
mul bx
mul byte ptr ds:[0]
mul word ptr [bx+si+8]
```

1) 计算100*10

两个数都小于255，可以做8位乘法。

```asm
mov al, 100
mov bl, 10
mul bl
```

结果： ax = 1000（03e8h）

2) 计算100*10000

需要进行16位乘法

```asm
mov ax, 10000
mov bx, 100
mul bx
```

结果： ax = 4240h, dx = 000fh。         f4240h = 1000000

## 模块化程序设计

计算data段中第一组数据的3次方，结果放在后面一组dword单元中。

```asm
assume cs:code, ds:data, ss:stack

data segment
		dw 1, 2, 3, 4, 5, 6, 7, 8
		dd 0, 0, 0, 0, 0, 0, 0, 0
data ends

stack segment stack
		dw 16 dup (0)
stack ends

code segment
start:
		mov ax, stack
		mov ss, ax
		mov sp, 10h
		
		mov ax, data
		mov ds, ax
		
		mov si, 0 ;  ds:[si] 指向第一组数据
		mov di, 16 ; ds:[di] 指向第二组数据
		
		mov cx, 8
cal:
		mov bx, [si]
   	call cube
   	mov [di], ax ; 低位放入ax
   	mov [di].2, dx ; 高位放入dx
   	add si, 2
   	add di, 4
		loop cal
   	
   	mov ax, 4c00h
		int 21h
   	


cube: ; 计算立方的子程序(参数存放在bx寄存器中，结果存放在ax和dx中)
		mov ax, bx
		mul bx ; 其实这里很有瑕疵，我们怎么知道平方没有超过16位呢？？
		mul bx
		ret

code ends
end start
```

## 多个参数的传送

上面我们使用的只有一个参数，那就是bx寄存器，但是有时候我们需要的不只是一个参数，这时候我们就需要额外使用其他的多项式。

例如：将给定的字符串变成全部大写。

我们需要两个参数，第一个参数是字符串的首地址，第一个参数是字符串的长度。

子程序的实现如下：

```asm
capital:
		and byte ptr [si], 1101111b ; 将 ds:[si]指向字符串的首地址
		inc si ; 指向下一个位置的字符
		loop capital ; 这里的循环就是表示，讲cx作为字符串的长度，很妙！
		ret
```

整个程序的应用如下。

```asm
assume cs:code, ds:data

data segment
	db 'conversation'
data ends

code segment
start:
	mov ax, data
	mov ds, ax
	mov si, 0
	mov cx, 12
	call capital
	
	mov ax, 4c00h
	int 21h

capital:
	and byte ptr [si], 11011111b
	inc si
	loop capital
	ret
	
code ends
end start
```

## 寄存器的冲突

设计一个程序，功能：将一个全是字母，以0结尾的字符串，转换为大写。

这个字符串可以使用如下的方式进行定义。

```asm
db 'conversation', 0
```

此时我们就不需要使用字符串的长度这个参数了，可以使用`jcxz`来检测0。

```asm
capital:
	mov cl, [si]
	mov ch, 0 ; 将一个8位的数据赋值给16位寄存器cx
	jcxz ok ; 如果cx为0，直接返回就行了
	and byte ptr [si], 11011111b ; 还是和之前一样的操作
	inc si
	jmp short capital ; 这里不再使用loop，因为cx不再决定循环的次数。
ok:
	ret
```

讲data段中的字符串全部转化为大写。

```asm
assume cs:code, ds:data

data segment
	db 'word', 0
	db 'unix', 0
	db 'wind', 0
	db 'good', 0
data ends

code segment
start:
	mov ax, data
	mov ds, ax
	mov si, 0
	mov cx, 4 ; 使用到了cx
upper:
	call capital
	inc si
	loop upper

	mov ax, 4c00h
	int 21h

capital:
	mov cl, [si]
	mov ch, 0 ; 使用到了cx
	jcxz ok
	and byte ptr [si], 11011111b
	inc si
	jmp short capital
ok:
	ret
	
code ends
end start
```

不过这个程序有点儿问题，内层和外层都使用到了cx这个寄存器。之前我们也遇到了这个问题，解决的方法是使用栈的方式，这里也是一样的。一般来说，我们在使用子程序的时候，**第一步要做的事情就是将子程序中使用到的寄存器入栈，最后返回之前要做的是将使用将寄存器按顺序出栈（和入栈的顺序相反）。**这样我们就能够保证，在子程序中使用到的寄存器不会影响到外面的寄存器。

### 子程序的框架

为了避免寄存器冲突的问题，子程序一般使用如下的框架。

```asm
子程序开始：
	使用到的寄存器入栈
	子程序内容
	使用到的寄存器出栈
	返回
```

改进之后的子程序`capital`如下。

```asm
capital:
	push cx
	push si
	
change:
	mov cl, [si]
	mov ch, 0
	jcxz ok
	and byte ptr [si], 11011111b
	inc si
	jmp short change
	
ok:
	pop si
	pop cx
	ret
```

改进之后的整体代码如下。

```asm
assume cs:code, ds:data, ss:stack

data segment
	db 'word', 0
	db 'unix', 0
	db 'wind', 0
	db 'good', 0
data ends

stack segment stack
	db 16 dup (0)
stack ends

code segment
start:
	mov ax, data
	mov ds, ax
	
	mov ax, stack
	mov ss, ax
	mov sp, 10h
	
	mov si, 0
	mov cx, 4 ; 使用到了cx
upper:
	call capital
	add si, 5 ; 在子程序中修改的si，不会影响到外部的si了。
	loop upper

	mov ax, 4c00h
	int 21h

capital:
	push cx
	push si
	
change:
	mov cl, [si]
	mov ch, 0 ; 使用到了cx
	jcxz ok
	and byte ptr [si], 11011111b
	inc si
	jmp short change ;这里不能在jmp short capital了，只能入栈一次
ok:
	pop si
	pop cx
	ret
	
code ends
end start
```

## 编写子程序

### 显示字符串

-   名称：`show_ptr`
-   功能：在指定的位置，用制定的颜色，显示一个使用0结束的字符串。
-   参数：
    -   `dh` => 行号 （0-24）
    -   `dl` => 列号 （0-79）
    -   `cl` => 颜色
    -   `ds:si `指向字符串的首地址

```asm
show_ptr:
    push cx
    push dx
    push si

    push es
    push bx
    push ax ; 这里我们额外使用到了es, ax, bx寄存器

    mov al, 160 ; 一行占用160个字符
    mul dh ; 8位数的乘法，结果放在ax中

    push dx ; 下面做加法需要使用到dx
    mov dh, 0 ; 讲dl变成dx
    add ax, dx ; ；因为 add ax, dl 是非法的
    add ax, dx ;确定列的位置
    pop dx

    mov bx, 0b800h
    mov es, bx ;显存开始的位置
    mov bx, ax ; 将开始显示的位置传送给bx寄存器

show:
    push cx ; 下面临时使用到了cx
    mov cl, [si]
    mov ch, 0
    jcxz ok ; 如果此时真的跳转到了ok， 那么下面的pop cx将无法执行
    pop cx

    mov ch, ds:[si] 
    mov es:[bx], ch ; 偶数地址放字符
    mov es:[bx+1], cl ; 奇数地址放颜色
    inc si ; 字符向后移动
    add bx, 2 ;显示的位置也向后移动
    jmp short show ; 跳转到show重复显示

ok:
    pop cx ; 一个push cx，两个pop cx

    pop ax
    pop bx
    pop es

    pop si
    pop dx
    pop cx
    ret
```

完整程序代码示例:

```asm
assume cs:code, ds:data, ss:stack

data segment
    db 'Welcome to masm', 0
data ends


stack segment stack
    db 128 dup (0) ; 栈空间千万不要申请太小！！否则栈空间会溢出，我之前设置为64字节，结果栈空间溢出覆盖了数据段中的数据，我还一脸懵逼呢！！
stack ends


code segment
start:
    mov ax, data
    mov ds, ax

    mov ax, stack
    mov ss, ax
    mov sp, 128

    mov dh, 10
    mov dl, 3
    mov cl, 2
    mov si, 0
    call show_ptr

    mov ax, 4c00h
    int 21h

show_ptr:
    push cx
    push dx
    push si

    push es
    push bx
    push ax ; 这里我们额外使用到了es, ax, bx寄存器

    mov al, 160 ; 一行占用160个字符
    mul dh ; 8位数的乘法，结果放在ax中

    push dx ; 下面做加法需要使用到dx
    mov dh, 0 ; 讲dl变成dx
    add ax, dx ; ；因为 add ax, dl 是非法的
    add ax, dx ;确定列的位置
    pop dx

    mov bx, 0b800h
    mov es, bx ;显存开始的位置
    mov bx, ax ; 将开始显示的位置传送给bx寄存器

show:
    push cx ; 下面临时使用到了cx
    mov cl, [si]
    mov ch, 0
    jcxz ok ; 如果此时真的跳转到了ok， 那么下面的pop cx将无法执行
    pop cx

    mov ch, ds:[si] 
    mov es:[bx], ch ; 偶数地址放字符
    ; 注意上面的代码不能写成 mov byte ptr es:[bx], [si]
    ; 需要借助一个寄存器讲ds段和es段的数据进行传送。
    
    mov es:[bx+1], cl ; 奇数地址放颜色
    
    inc si ; 字符向后移动
    add bx, 2 ;显示的位置也向后移动
    jmp short show ; 跳转到show重复显示

ok:
    pop cx ; 一个push cx，两个pop cx

    pop ax
    pop bx
    pop es

    pop si
    pop dx
    pop cx
    ret

code ends
end start
```

### 解决除法溢出的问题

```asm
mov bh, 1
mov ax, 1000
div bh
```

此时进行的是八位数的除法，商为1000，余为0。但是商1000在al中无法存放，这就是除法溢出现象。

```asm
mov ax, 1000h
mov dx, 1
mov bx, 1
div bx
```

此时进行的16位的除法，商为11000，余为0，但是商11000在ax中无法存放，这也是除法溢出现象。

可以通过相应的程序处理可能产生的除法溢出。

-   名称：`divdw`

-   功能：进行不会产生溢出的除法运算，被除数为dword，除数为word，结果为dword

-   参数

    -   `ax` => dword数据的低16位 L
    -   `dx` => dword数据的高16位 H
    -   `cx` => 除数 N 

-   返回

    -   `ax` => 结果的低16位

    -   `dx` => 结果的高16位

    -   `cx` => 余数

- 公式

    `X/N = int(H/N)*65536 + ((H%N)*65536 + L)/N` 注：(65536 = 10000h)

    这个公式将可能产生除法溢出的`X/N`变成右面的几个不会产生除法溢出。右面的除法都是可以使用`div`指令进行运算的。

```asm
divdw:
	; ax: L bx: H, cx: N
	
	push bx
	push ax
	
	mov ax, dx
	mov dx, 0
	div cx ; ax = int(H/N) dx = H%N

	mov bx, ax ; bx = int(H/N) 结果的高八位
	pop ax
	; ax = L, dx = H%N
	div cx ; ax = ((H%N)*65536 + L)/N 结果的低八位 dx = ((H%N)*65536 + L)%N 余数
	mov cx, dx ; cx中为余数
	mov dx, bx
	pop bx
	ret
```

这个其实有点绕，但是看清楚还是非常简单的。

例：计算1000000/10 也就是(f4240h/0ah)

```asm
assume cs:code, ss:stack

stack segment stack
    db 128 dup (0)
stack ends

code segment
start:
    mov ax, 4240h
    mov dx, 000fh
    mov cx, 0ah
    call divdw

    mov ax, 4c00h
    int 21h

divdw:
    ; ax: L bx: H, cx: N

    push bx
    push ax

    mov ax, dx
    mov dx, 0
    div cx ; ax = int(H/N) dx = H%N

    mov bx, ax ; bx = int(H/N) 结果的高八位
    pop ax
    ; ax = L, dx = H%N
    div cx ; ax = ((H%N)*65536 + L)/N 结果的低八位 dx = ((H%N)*65536 + L)%N 余数
    mov cx, dx ; cx中为余数
    mov dx, bx
    pop bx
    ret
code ends
end start
```

运行结果：

dx = 0001h, ax = 86a0h, cx = 0

### 数值显示

将data段的数据以十进制的形式显示出来。

```asm
data segment
	dw 123, 12660, 1, 8, 3, 38
data ends
```

显示字符串的子程序，`show_ptr`我们已经写好了，现在我们需要做的就是如何讲计算机底层的二进制数字转换成十进制形式的字符串。

-   名称： `dtoc`
-   功能：将word型数据表示为十进制数字的字符串，字符串以0为结尾符。
-   参数
    -   `ax` ：word型数据
    -   `ds:si` 指向字符串的首地址

```asm
dtoc:
	push ax
	push si
	
	push bx
	push cx
	push dx
	mov bx, 10
	
to10Char:
	mov cx, ax
	jcxz ok ; ax此时已经为0了，已经得到了所有的十进制的值
	; 下面的除法为32位除以16位，主要是为了防止除法溢出以及让商还在ax中
	mov dx, 0
	div bx ; ax/10 余数在dx中，商在ax中
	add dl, 30h ; dl为对应的十进制字符
	mov byte ptr [si], dl
	inc si ; 移动到下一个位置
	jmp short to10Char
	
ok:
	mov byte ptr [si], 0 ; 在某位添加0
	; 此时数据是倒序的，我们要将其变成正的，还要写一个reverse_str
	; 但是根据现有的知识，比较麻烦，先暂且不弄了。
	
	pop dx
	pop cx
	pop bx
	
	pop si
	pop ax

	ret
	
```

完整程序示例：

```asm
assume cs:code, ds:data, ss:stack

data segment
    db 10 dup (0)
data ends

stack segment stack
    db 128 dup (0)
stack ends

code segment
start:
    mov ax, data
    mov ds, ax

    mov ax, stack
    mov ss, ax
    mov sp, 128

    mov ax, 12666
    mov si, 0
    call dtoc

    mov dh, 10
    mov dl, 3
    mov cl, 2
    call show_str

    mov ax, 4c00h
    int 21h

dtoc:
    push ax
    push si

    push bx
    push cx
    push dx
    mov bx, 10

to10Char:
    mov cx, ax
    jcxz ok2 ; ax此时已经为0了，已经得到了所有的十进制的值
    ; 下面的除法为32位除以16位，主要是为了防止除法溢出以及让商还在ax中
    mov dx, 0
    div bx ; ax/10 余数在dx中，商在ax中
    add dl, 30h ; dl为对应的十进制字符
    mov byte ptr [si], dl
    inc si ; 移动到下一个位置
    jmp short to10Char

ok2:
    mov byte ptr [si], 0 ; 在某位添加0

    pop dx
    pop cx
    pop bx

    pop si
    pop ax

    ret

show_str:
    push cx
    push dx
    push si

    push es
    push bx
    push ax ; 这里我们额外使用到了es, ax, bx寄存器

    mov al, 160 ; 一行占用160个字符
    mul dh ; 8位数的乘法，结果放在ax中

    push dx ; 下面做加法需要使用到dx
    mov dh, 0 ; 讲dl变成dx
    add ax, dx ; ；因为 add ax, dl 是非法的
    add ax, dx ;确定列的位置
    pop dx

    mov bx, 0b800h
    mov es, bx ;显存开始的位置
    mov bx, ax ; 将开始显示的位置传送给bx寄存器

show:
    push cx ; 下面临时使用到了cx
    mov cl, [si]
    mov ch, 0
    jcxz ok ; 如果此时真的跳转到了ok， 那么下面的pop cx将无法执行
    pop cx

    mov ch, ds:[si] 
    mov es:[bx], ch ; 偶数地址放字符
    mov es:[bx+1], cl ; 奇数地址放颜色
    inc si ; 字符向后移动
    add bx, 2 ;显示的位置也向后移动
    jmp short show ; 跳转到show重复显示

ok:
    pop cx ; 一个push cx，两个pop cx

    pop ax
    pop bx
    pop es

    pop si
    pop dx
    pop cx
    ret

code ends
end start
```