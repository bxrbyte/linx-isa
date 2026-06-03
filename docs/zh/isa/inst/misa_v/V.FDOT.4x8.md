# V.FDOT.4x8

## 说明

浮点向量点积4×8（*Vector Floating-point Dot Product 4×8*）<br>
对SrcL和SrcR中同一个lane的8组4-bit浮点值分为低4组和高4组，每组内4对值分别相乘后再累加，并分别加上SrcD中对应lane低16-bit和高16-bit的累加初值，两个结果分别写入目的寄存器的低16-bit和高16-bit。与V.DOT.4x8逻辑相同，但操作数为浮点类型。

## 汇编语法

```asm
    v.fdot.4x8 SrcL<.reuse>.{T}, SrcR<.reuse>.{T}, SrcD<.reuse>.{T}, ->RegDst.{W}
```

## 汇编符号

- **SrcL**：左源乘数向量，可以索引的寄存器类型请见[向量指令介绍](../../blockIntro/vecinstrs/instIntro.md)。
- **SrcR**：右源乘数向量，可以索引的寄存器类型请见[向量指令介绍](../../blockIntro/vecinstrs/instIntro.md)。
- **SrcD**：累加初值寄存器，每个lane的低16-bit和高16-bit分别提供两组累加初始值。
- **reuse**：当源寄存器为向量寄存器时可增加本后缀，用于指示当前指令提交后本寄存器不允许被释放。如无此标识，则表示允许硬件释放本寄存器。
- **T**：指定操作数的数据类型，仅支持fs（32-bit宽输入，每个lane内打包8个4-bit浮点值）。
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
for (laneid = 0; laneid < lanenum; laneid++) {
    if (pmask[laneid] == 1) {
        bits(32) srcL = V[m, 32, laneid];
        bits(32) srcR = V[n, 32, laneid];
        bits(32) acc  = V[acc, 32, laneid];

        bits(16) sum_lo = acc[15:0];
        bits(16) sum_hi = acc[31:16];

        // 低 16-bit 区域：4 组 4-bit 乘加
        for (k = 0; k < 4; k++) {
            bits(4) xk = srcL[4*k+3 : 4*k];
            bits(4) yk = srcR[4*k+3 : 4*k];
            sum_lo += (dsttype)xk * (dsttype)yk;
        }

        // 高 16-bit 区域：4 组 4-bit 乘加
        for (k = 0; k < 4; k++) {
            bits(4) xk = srcL[4*k+19 : 4*k+16];
            bits(4) yk = srcR[4*k+19 : 4*k+16];
            sum_hi += (dsttype)xk * (dsttype)yk;
        }

        bits(32) packed = {sum_hi, sum_lo};
        V[d, 32, laneid] = packed;
    } else {
        V[d, dstwidth, laneid] = 0;
    }
}
```

## 备注

本指令属于[超长指令扩展](../../instset/longInstrs.md)，可用于向量数据块或访存数据块中。
点积运算中，中间乘法结果应扩展到更高精度后再累加，以避免精度损失。
对于更大规模的矩阵运算（≥16×16×16），请使用CUBE运算指令。
