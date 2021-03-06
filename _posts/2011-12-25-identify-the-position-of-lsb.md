---
layout: post
title: Identify the position of the LSB 
tags:  algorithm
---


前段时间看到一段代码用于计算一个32位二进制数的最低为一的位的位置,一直没想到怎么证明它的正确性.今天终于有点头绪了,但仍不知道对不对.暂且记在这里.


{% highlight c %}

#define    u      99
int ntz8a( unsigned x )
{
    static char table[64] ={
      32, 0, 1,12,  2, 6, u,13, 3, u, 7, u,  u, u, u,14,  
      10, 4, u, u,  8, u, u,25,  u, u, u, u,  u,21,27,15,
      31,11, 5, u,  u, u, u, u,  9, u, u,24,  u, u,20,26,  
      30, u, u, u,  u,23, u,19, 29, u,22,18, 28,17,16, u
    };
    
    x = ( x & -x );
    x = ( x << 4 ) + x;    
    x = ( x << 6 ) + x;                                  
    x = ( x << 16 ) - x;     
    
    return table[ x>>26 ];
}

{% endhighlight %}

这段代码由David Seal于1994年发表，原始版本是arm的汇编:

{% highlight c %}
  ; Operand in R0, register R1 is free, R2 addresses a byte table 'Table' 
    RSB    R1,R0,#0            ;Standard trick to isolate bottom bit in R1, 
    AND    R1,R1,R0            ; or produce zero in R1 if R0 = 0. 
    ORR    R1,R1,R1,LSL #4    ;If R1=X with 0 or 1 bits set, R0 = X * &11 
    ORR    R1,R1,R1,LSL #6    ;R0 = X * &451 
    RSB    R1,R1,R1,LSL #16    ;R1 = X * &0450FBAF 
    LDRB    R1,[R2,R1,LSR #26]  ;Look up table entry indexed by top 6 bits of R1 
  ; Result in R1
{% endhighlight %}

用同余的思路证明：
令 K = x&-x；  则 K = 2^n;   n+1是x的32位二进制表示中最低的为1那位的位置

例如：   x=00111110001110010110001011000000；-x的二进制表示为x的各位取反加一。
所以    -x=11000001110001101001110101000000；所以x&-x的二进制表示为
      x&-x=00000000000000000000000001000000;  2^6;
      
因此任何32位数经过x & -x都转换为一个2^n形式的数。\(0 = < n < 32 \)我们需要证明对任何两个n1与n2对应的2^n1与2^n2经过算法中的一系列运算后不会发生碰撞。
证明如下：  \(注意所有运算结果被限制在32位内。即相当于每次运算结果对2^32求余\)

               令x = 2^k  (  0 = < k < 32 )
               
               则 x = ( x << 4 ) + x相当于
               x=[(16*x )mod 2^32 + x mod 2^32 ] mod 2^32;
               由于 (A mod K + B mod K)= (A+B)mod K 可得：
               x=(17*x) mod 2^32;
               
               同理 x=(x<<6)+x可表示为
               
               x={[(17*x mod 2^32 )*64] mod 2^32 +17*x mod 2^32 } mod 2^32
               
               由同余等式 [ (A mod K)* (B mod K)]= (A * B)mod K 可得：第二次移位累加后
               
               x=(17*65*x)mod 2^32；
               
               同理，经过第三次移位累加后
               
               x=(17*65*65535*x) mod 2^32      17*65*65535 = 72416175 
              
               最后一步用来得出索引 index = x>>26
               
               对任意K1与K2
               
               X1 = (72416175*K1)mod 2^32 = 72416175*K1 - P1 * 2^32 = A1 * 2^26 + R1；  
               X2 = (72416175*K2)mod 2^32 = 72416175*K1 - P2 * 2^32 = A2 * 2^26 + R2；
               
               2^32 ≡  2^26 ≡ 0 mod 2^26 
               所以 Xi mod 2^26 = Ri = 72416175*Ki % 2^26
               
               X1 >> 26 = (72416175*K1*2^-26) mod 2^6; 
               X2 >> 26 = (72416175*K2*2^-26) mod 2^6;
              
               令Ki*2^-26 = Pi;
               
               则 
               2^-26  <= Pi < 2^6;
               
               若  X1 >> 26 = X2 >> 26 则
               
               72416175*P1 ≡  72416175*P2 mod 2^6
               
               即           2^6 | 72416175*(P1-P2)
               
               但是 2^6 与72516175互素;2^6也不能整除P1-P2;
               
               故   2^6 不能整除72416175*(P1-P2)
               
               即 X 1>> 26 ≠ X2 >> 26;
               命题得证
             
如何求查找表:

取71246175 * K的26~32位构成一个0~64之间的数即可.设这个数为index; K = 2^n ( 0 =< n < 32 )
72416175的二进制为: 00000100010100001111101110101111
\

        71246175  *  K :                                  位32~26      index    k
        |000001|00010100001111101110101111                 000001        1      2^0
        |000010|00101000011111011101011110                 000010        2      2^1
        |000100|01010000111110111010111100                 000100        4      2^2
        |001000|10100001111101110101111000                 001000        8      2^3
        |010001|01000011111011101011110000                 010001        17     2^4
        |100010|10000111110111010111100000                 100010        34     2^5
        |000101|00001111101110101111000000                 000101        5      2^6
        |001010|00011111011101011110000000                 001010        10     2^7
        |010100|00111110111010111100000000                 010100        20     2^8
        |101000|01111101110101111000000000                 101000        40     2^9
        |010000|11111011101011110000000000                 010000        16     2^10
        |100001|11110111010111100000000000                 100001        33     2^11
        |000011|11101110101111000000000000                 000011        3      2^12
        |000111|11011101011110000000000000                 000111        7      2^13
        |001111|10111010111100000000000000                 001111        15     2^14
        |011111|01110101111000000000000000                 011111        31     2^15
        |111110|11101011110000000000000000                 111110        62     2^16
        |111101|11010111100000000000000000                 111101        61     2^17
        |111011|10101111000000000000000000                 111011        59     2^18
        |110111|01011110000000000000000000                 110111        55     2^19
        |101110|10111100000000000000000000                 101110        46     2^20
        |011101|01111000000000000000000000                 011101        29     2^21  
        |111010|11110000000000000000000000                 111010        58     2^22
        |110101|11100000000000000000000000                 110101        53     2^23
        |101011|11000000000000000000000000                 101011        43     2^24
        |010111|10000000000000000000000000                 010111        23     2^25
        |101111|00000000000000000000000000                 101111        47     2^26
        |011110|00000000000000000000000000                 011110        30     2^27
        |111100|00000000000000000000000000                 111100        60     2^28
        |111000|00000000000000000000000000                 111000        56     2^29
        |110000|00000000000000000000000000                 110000        48     2^30
        |100000|00000000000000000000000000                 100000        32     2^31

呵呵,这是不是很像一个Hash 算法~~~ 
