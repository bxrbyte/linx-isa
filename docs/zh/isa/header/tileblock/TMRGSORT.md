# TMRGSORT

## 说明

**数据块归并排序（Tile Merge Sort）**

`TMRGSORT` 对一个或多个已排序列表执行归并排序，支持单列表多分段归并和多列表归并两种模式。

实现伪代码示意如下：
```pseudocode
// 归并排序操作
for r in 0..(ValidRow-1):                              // 遍历所有行
  // 多段归并：将 src 按 blockLen 分段，段内已有序
  // 归并各段到 dst
  dst[r, :] = KWayMerge([src[r, k*blockLen..(k+1)*blockLen-1] for k in 0..3])
  // 多列表归并：将多个已排序列表归并
  dst[r, :] = Merge([src0[r,:], src1[r,:], src2[r,:], src3[r,:]])
```

---

## 汇编语法

```asm
    TMRGSORT <LB0:ValidCol, LB1:ValidRow, DataType, BlockLen>, SrcTile<.reuse>, ->DstTile<Size>
```

多列表形式：
```asm
    TMRGSORT <LB0:ValidCol, LB1:ValidRow, DataType>, SrcTile0<.reuse>, SrcTile1<.reuse>, SrcTile2<.reuse>, SrcTile3<.reuse>, ->DstTile<Size>, ExeNum<Size>
```

## 汇编符号

- **ValidCol**：输出 Tile 中有效元素的列数。该参数可以通过以下 3 种形式配置到 LB0 寄存器中：
    - **reg**：通过全局寄存器 [GGPR](../../register/common/ggpr.md) 设置。
    - **imm**: 使用立即数设置。
    - **reg+imm**：通过全局寄存器加立即数的形式设置。
- **ValidRow**：输出 Tile 中有效元素的行数（可缺省，默认值：`1`）。该参数配置到 LB1 寄存器中，配置方式同上。
- **DataType**：输入/输出 Tile 元素的数据格式，支持 `FP32`、`FP16`。
- **BlockLen**（单列表形式）：每个已排序分段的长度，必须为 64 的倍数。
- **SrcTile**：输入 Tile 寄存器（单列表形式），支持 `T`/`U`/`M`/`N` 队列输入（参见：[Tile 寄存器](../../register/common/tilereg.md)）。
- **SrcTile0~3**：4 个已排序输入 Tile 寄存器（多列表形式）。
- **reuse**（后缀）：指示当前指令提交后保留寄存器（若无此标识，允许硬件自动释放）。
- **DstTile**：输出 Tile 寄存器，归并后的有序结果。支持 `T`/`U`/`M`/`N` 队列输出。
- **ExeNum**：执行列表数输出 Tile（多列表形式，可选），输出实际参与归并的列表数量。
- **Size**：输出 Tile 寄存器的空间大小（有效范围参见：[Tile 寄存器](../../register/common/tilereg.md)）。

---

## 编码格式

该 TileOp 模版块编码为以下指令：

**单列表形式**：
- [BSTART.TEPL](../../blockIntro/tepl_block/header.md) `TMRGSORT, DataType`
- [B.DATR](../../header/B.DATR.md) `BlockLen`
- [B.DIM](../../header/B.DIM.md) `reg, imm, ->LB0`   （注：*ValidCol*）
- [B.DIM](../../header/B.DIM.md) `reg, imm, ->LB1`   （注：*ValidRow*）
- [B.IOT](../../header/B.IOT.md) `SrcTile<.reuse>, last, ->DstTile<Size>`

**多列表形式**：
- [BSTART.TEPL](../../blockIntro/tepl_block/header.md) `TMRGSORT_4LIST, DataType`
- [B.DIM](../../header/B.DIM.md) `reg, imm, ->LB0`   （注：*ValidCol*）
- [B.DIM](../../header/B.DIM.md) `reg, imm, ->LB1`   （注：*ValidRow*）
- [B.IOT](../../header/B.IOT.md) `SrcTile0<.reuse>, SrcTile1<.reuse>, SrcTile2<.reuse>, SrcTile3<.reuse>, ->DstTile<Size>`
- [B.IOT](../../header/B.IOT.md) `last, ->ExeNum<Size>`   （注：*可选*）

## 约束条件

- **单列表模式**：
    - `ValidCol` 必须为 `blockLen × 4` 的倍数。
    - 重复次数 `repeatTimes = ValidCol / (blockLen × 4)` 必须在 `[1, 255]` 范围内。
    - `blockLen` 必须为 64 的倍数。
- **多列表模式**：
    - 4 个源 Tile 的 `DataType` 和 `ValidRow` 必须一致。
    - 每个源 Tile 内部已按值有序。
- **数据类型**：仅支持 `FP32`、`FP16`。
- **存储布局**：必须是行主序（RowMajor）。
- **尺寸范围**：Tile 的行列/有效行列等参数大小均必须小于等于 16 bit。

---

## 汇编示例

单列表形式：
```asm
    TMRGSORT <LB0:256, LB1:4, FP16, 64>, T#1.reuse, ->T<2KB>
```

1. **操作内容**
    - 将 `T#1` Tile 的每行 256 个元素（= 4 段 × 64 元素）归并排序
    - 输出：结果存入 `T` 队列 Tile 寄存器
2. **单列表归并**
    - `blockLen=64`：每段 64 个已排序元素，共 4 段

---

## 备注

此指令是 TileOp 模版块，软件只定义块头。
