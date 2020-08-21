---
title: "Kỹ thuật Dịch ngược: Nhân để chia"
author: Huy
date: 2019-12-01 8:48:22 +1345
tags: [reverse engineer]
---

Trong bài viết này, mình sẽ giúp các bạn decompile một vài phương thức tối ưu của compiler.

![Image for post](https://miro.medium.com/max/850/0*kmssB7j1LbCF5h71.png)

Khi dịch ngược một chương trình đã được tối ưu hóa thường tốn rất nhiều thời gian. Bằng cách tìm hiểu phương thức tối ưu của các compiler, chúng ta có thể đẩy nhanh tiến độ của mình và tiết kiệm được kha khá thời gian.

# Thay thế phép chia bằng phép nhân

*Đừng lo, không có gì cao siêu trong phần này đâu.*

Trong khi đang dịch ngược một con rootkit, mình đã phát hiện một đoạn mã assembly trông có vẻ “lạ”:

```assembly
mov     ecx, dword ptr [X]        
mov     rax, 0x3333333333333400          
mul     rcx
```

Mỗi khi nhìn thấy những đoạn mã như thế này, mình thường tự hỏi rằng: Liệu có phải mấy ông dev viết cái thứ này không? Và sau khi tìm hiểu đoạn mã này làm gì, mình đã nhận ra đây là một phương thức tối ưu của compiler — Đoạn mã này đơn giản là thực hiện một phép chia mà không sử dụng ‘div’ ([division instruction](http://www.c-jump.com/CIS77/MLabs/M11arithmetic/M11_0080_div_instruction.htm)). Phép chia là một lệnh rất “đắt đỏ” và “tốn kém” cho CPU — thậm chí còn hơn cả phép nhân luôn. Bây giờ chúng ta hãy tìm hiểu compiler đã tối ưu hóa điều này như thế nào nhé.

Giả sử ta muốn chia X cho 5. Trong toán học, (X/5) = (X*(1/5)). Từ đó ai cũng có thể suy ra rằng chúng ta có thể đảo ngược 5 thành 1/5 rồi sử dụng phép nhân là đã có kết quả của phép chia ban đầu. Nhưng điều thắc mắc là: *1/5 là một số thực, làm sao ta có thể biểu diễn nó mà không sử dụng dấu thập phân?*

Ta có thể sử dụng lệnh [mul](https://www.aldeid.com/wiki/X86-assembly/Instructions/mul): Lệnh dùng để thực hiện phép nhân giữa thanh ghi rax và một toán hạng (operand). Nếu các toán hạng là 64-bit, kết quả của phép nhân có thể là 128-bit. Vì vậy sau khi tính, ‘mul’ trữ kết quả vào 2 thanh ghi (register):

- rax — phần 64-bit “dưới” của kết quả
- rdx — phần 64-but “trên” của kết quả

Đây là pseudo-code của lệnh ‘mul’:

```c++
void mul(uint64_t operand) { 
    __int128 result = (rax * operand);
    // take the lower 64 bits and store in rax
    rax = (result & 0xffffffffffffffff);

    // store the upper 64 bits and store in rdx
    rdx = (result >> 64) & 0xffffffffffffffff;                
}
```

Vậy ta sử dụng ‘mul’ như thế nào để thực hiện phép toán (X*(1/5))?

Giờ mình sẽ biểu diễn số thập phân như thế này: thanh ghi rax sẽ giữ toàn bộ phần số nguyên và thanh ghi rdx sẽ giữ phần thập phân.

Ở phần này, 2⁶⁴ sẽ đại diện cho giá trị 1:

```c++
fractional_part = 0
whole_number    = 1

full_representation: 0x 0000000000000001.0000000000000000
```

Để viết được 1/5, chúng ta phải chia 2⁶⁴ cho 5.0:
$$
2^{64} / 5.0 =  0x3333333333333400
$$
*Lưu ý khi thực hiện những phép tính này, hãy để ý việc làm tròn số.*

Okay, `0x3333333333333400` là giá trị ta cần tìm rồi. Bây giờ ta sẽ thử chia 10 cho 5 nhé:

```
0x0000000000000000 0x000000000000000A (10)
                  *
0x0000000000000000 0x3333333333333400 (0.2)
                  =
0x0000000000000002 0x0000000000000000 (2)
```

Để có được kết quả là 2, ta phải dịch (shift) kết quả sang phải 64 bits. Lí do toán hạng đầu tiên (10) không bị dịch là vì ta đã dịch toán hạng thứ hai rồi.

Tiếp theo, hãy thử xài cách trên để tối ưu hóa phép chia như sau:

```c
//
// Real calculation:
//
Result = (X / 5.0)

//
// Calculation Using Multiplication:
//
Result = (X * (1/5.0))

//
// Calculation using our decimal representation: 
//
Result = (X * 0x3333333333333400) >> 64


//
// Calulation using assembly:
//
__asm {

	// Store (2^64 / 5.0) in rcx
	mov rcx, 0x3333333333333400 

	// Store X in rax.
	mov rax, qword ptr [X] 

	// Multiply.
	// The result of the multiplication is stored in 	rdx, which stores the higher 64 bits.
	mul rcx
}
```

Phần assembly code này cơ bản chính là đoạn mã “kì lạ” mà mình đã nói đến ở đầu bài viết này đấy ;)

Sử dụng mánh này ta đã có thể chia mà không cần xài ‘div’ rồi.

# Ví dụ thực nghiệm

Hãy nhìn vào đoạn assembly sau:

```assembly
mov     rcx, qword ptr [X]
mov     rax, 0xCCCCCCCCCCCCCCCD
mul     rcx
mov     rdi, rdx
shr     rdi, 4
```

Làm sao để dịch ngược đoạn này đây? Đầu tiên ta phải biết được `0xCCCCCCCCCCCCCCCD` là gì đã. Mình sẽ đảo ngược nó như thế này:

```python
>>> 0xCCCCCCCCCCCCCCCD / float(2**64)
0.8
```

Giờ ta đã biết được lệnh ‘mul’ đầu tiên sẽ nhân X với 0.8 (4/5) và lưu kết quả vào thanh rdx. Sau đó, có lệnh dịch phải 4 bit — có nghĩa là chia cho (2⁴) = 16.

Phép tính tổng quát:

```c
Result = (X * 4/5) / 16
```

Ta có thể đổi lại như thế này:

```c
(X * 4/5 * 1/16)
```

Sau đó ghép 2 phép nhân lại:

```c
(X * 4/80) -> (X * 1/20)
```

Vậy là ta đã biết đoạn mã trên đơn giản là chia X cho 20.

# Summary

Hy vọng là bài viết này hữu ích. Mình sẽ tiếp tục dăng những bài liên quan đến RE, có thể là cả mấy bài pwn nữa nếu mình không lười hihi.

Thanks for reading!