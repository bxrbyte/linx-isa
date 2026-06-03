# V.FDOT

## 说明

浮点向量点积（*Vector Floating-point Dot Product*）<br>
将SrcL和SrcR中同一个lane的浮点值相乘后，再将每4个连续lane的乘积累加起来，加上SrcD中最小lane的累加初值，结果广播（broadcast）到这4个lane中。与V.DOT逻辑相同，但操作数为浮点类型。

## 汇编语法

```asm
    v.fdot SrcL<.reuse>.{T}, SrcR<.reuse>.{T}, SrcD<.reuse>.{T}, ->RegDst.{W}
```

## 汇编符号

- **SrcL**：左源乘数向量，可以索引的寄存器类型请见[向量指令介绍](../../blockIntro/vecinstrs/instIntro.md)。
- **SrcR**：右源乘数向量，可以索引的寄存器类型请见[向量指令介绍](../../blockIntro/vecinstrs/instIntro.md)。
- **SrcD**：累加初值寄存器，提供各组归约前的初始值。取值来自每组4个lane中最小lane号对应的寄存器值。
- **reuse**：当源寄存器为向量寄存器时可增加本后缀，用于指示当前指令提交后本寄存器不允许被释放。如无此标识，则表示允许硬件释放本寄存器。
- **T**：指定操作数的数据类型，可选类型包括fb, fh, fs等8/16/32-bit浮点型。
- **->**：用于指示目的寄存器。
- **RegDst**：目的寄存器。
- **W**：目的寄存器位宽，至少是源操作数的两倍（如fh×fh累加结果至少为fs宽度）。

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
for (gid = 0; gid < lanenum / 4; gid++) {
    integer base_lane = gid * 4;
    bits(64) sum = V[acc, accwidth, base_lane];

    for (j = 0; j < 4; j++) {
        integer laneid = base_lane + j;
        if (pmask[laneid] == 1) {
            bits(64) opL = V[m, srcwidth, laneid];
            bits(64) opR = V[n, srcwidth, laneid];
            sum += (dsttype)opL * (dsttype)opR;
        }
    }

    for (j = 0; j < 4; j++) {
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
