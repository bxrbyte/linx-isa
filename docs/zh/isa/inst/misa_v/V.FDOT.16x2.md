# V.FDOT.16x2

## 说明

浮点向量点积16×2（*Vector Floating-point Dot Product 16×2*）<br>
对SrcL和SrcR中同一个lane的两组16-bit浮点值分别相乘（每个32-bit lane中包含高16-bit和低16-bit各一对），并将每2个连续lane的4个乘积累加起来，加上SrcD中最小lane的累加初值，结果广播（broadcast）到这2个lane中。与V.DOT.16x2逻辑相同，但操作数为浮点类型。

## 汇编语法

```asm
    v.fdot.16x2 SrcL<.reuse>.{T}, SrcR<.reuse>.{T}, SrcD<.reuse>.{T}, ->RegDst.{W}
```

## 汇编符号

- **SrcL**：左源乘数向量，可以索引的寄存器类型请见[向量指令介绍](../../blockIntro/vecinstrs/instIntro.md)。
- **SrcR**：右源乘数向量，可以索引的寄存器类型请见[向量指令介绍](../../blockIntro/vecinstrs/instIntro.md)。
- **SrcD**：累加初值寄存器，提供各组归约前的初始值。取值来自每组2个lane中最小lane号对应的寄存器值。
- **reuse**：当源寄存器为向量寄存器时可增加本后缀，用于指示当前指令提交后本寄存器不允许被释放。如无此标识，则表示允许硬件释放本寄存器。
- **T**：指定操作数的数据类型，仅支持fs（32-bit宽输入，每个lane内打包2个16-bit浮点值）。
- **->**：用于指示目的寄存器。
- **RegDst**：目的寄存器。
- **W**：目的寄存器位宽固定为32-bit（.w）。

## 编码格式

（待补充）

## 执行方式

- 解码源寄存器域：[DecodeFP](../LibPseudoCode.md#locationM)
- 解码输出参数：[DecodeDst](../LibPseudoCode.md#locationN)
- 通用寄存器读写：[V\[\]](../LibPseudoCode.md#locationB)

```c
integer {m, srcwidth} = DecodeINT(SrcL);
integer {n, srcwidth} = DecodeINT(SrcR);
integer {acc, accwidth} = DecodeINT(SrcD);
integer {d, dstwidth} = DecodeDst(RegDst);

bits(64) pmask = P;
for (gid = 0; gid < lanenum / 2; gid++) {
    integer base_lane = gid * 2;
    bits(64) sum = V[acc, accwidth, base_lane];

    for (j = 0; j < 2; j++) {
        integer laneid = base_lane + j;
        if (pmask[laneid] == 1) {
            bits(32) srcL = V[m, 32, laneid];
            bits(32) srcR = V[n, 32, laneid];

            bits(16) x1 = srcL[15:0];
            bits(16) x2 = srcL[31:16];
            bits(16) y1 = srcR[15:0];
            bits(16) y2 = srcR[31:16];

            sum += (dsttype)x1 * (dsttype)y1 + (dsttype)x2 * (dsttype)y2;
        }
    }

    for (j = 0; j < 2; j++) {
        integer laneid = base_lane + j;
        if (pmask[laneid] == 1)
            V[d, dstwidth, laneid] = sum;
        else
            V[d, dstwidth, laneid] = 0;
    }
}
```

## 备注

本指令属于[超长指令扩展](../../instset/longInstrs.md)，可用于向量数据块或访存数据块中。
点积运算中，中间乘法结果应扩展到更高精度后再累加，以避免精度损失。
对于更大规模的矩阵运算（≥16×16×16），请使用CUBE运算指令。
