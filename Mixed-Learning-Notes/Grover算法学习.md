**<center><font face="微软雅黑" size=10>Grover算法学习</font></center>**
代码（作者：Microsoft）：[https://github.com/microsoft/Quantum/tree/main/samples/algorithms/sudoku-grover]

翻译与整理：B20200342吕征宇 @北京科技大学 2020/11/18

# 问题描述

数独是一个图的着色问题，图的边必须连接不同颜色的节点。在我们的例子中，图节点是拼图方块，颜色是数独数字。图边是阻止正方形具有相同值的约束。为了减少所需的量子比特数，我们只对空方块使用量子比特。
我们使用两种数据结构定义谜题：

- 连接空方块的边的列表

- 对拼图中初始数字（起始数字）的空方块的约束列表
    该代码适用于9x9数独游戏和4x4数独游戏。
    此描述将使用4x4拼图，以使其更易于理解。
    4x4谜题的答案是0到3，而不是1到4。
    这是因为我们可以用2个量子比特来编码0-3。

但是，同样的规则也适用：

- 数字0到3在每行、每列和2x2子方格中只能出现一次。

```
    As an example              has solution
    _________________          _________________
    |   | 1 |   | 3 |          | 0 | 1 | 2 | 3 |  
    -----------------          -----------------
    | 2 |   |   | 1 |          | 2 | 3 | 0 | 1 |  
    -----------------          -----------------
    |   |   | 3 | 0 |          | 1 | 2 | 3 | 0 |  
    -----------------          -----------------
    | 3 |   | 1 |   |          | 3 | 0 | 1 | 2 |  
    -----------------          -----------------
   
```

在上面的示例中，最上面行的边/约束是：

```
   _________
  | ______   \                   _____   
  || __   \   \                  | __  \                        __
 _|||__\___\_ _\__         ______||__\___\__          _________|___\__ 
 |   | 1 |   | 3 |         |   | 1 |   | 3 |         |   | 1 |   | 3 | 
 -----------------         -----------------         -----------------
```

对于上面的行，空方块有索引

```
_________________
| 0 |   | 1 |   |
-----------------
```

对于此行，空正方形列表只有一个条目：
空正方形数=（0,1），即空正方形0不能与空正方形1具有相同的数字。
这些空方块对起始数字的限制是：
起始数字约束=（0,1）（0,3）（1,1）（1,3）
这是一个列表（空方块，不能是数字）。
i、 e.空正方形0不能有值1或3，空正方形1不能有值1或3。

# 主函数

用Grover算法解决数独难题。

### 输入

- numVertices

    空白方块数。

- size

    数独的大小。4个用于4x4格，9个用于9x9格。

- emptySquareEdges

    传统的边传递给图着色算法，在我们的例子中，是空的拼图方块。这些边定义空单元格之间的任何“同一行”、“同一列”、“同一子网格”关系。

- startingNumberConstraints

    当我们开始的时候，由于数字已经在数独中，对空方块的限制。

### 输出

一个包含结果和每个空正方形的数字数组的元组。

### 提示

以下4x4数独的输入和输出是：

```
    -----------------
    |   | 1 | 2 | 3 |         <--- empty square #0
    -----------------
    | 2 |   | 0 | 1 |         <--- empty square #1
    -----------------
    | 1 | 2 | 3 | 0 |
    -----------------
    | 3 |   | 1 | 2 |         <--- empty square #2
    -----------------
```

emptySquareEdges = [(1, 0),(2, 1)] 空格 #0 不能够和 #1 有相同的颜色或者数字，空格 #1 不能够和 #2 有相同的颜色或者数字。

startingNumberConstraints = [(0, 2),(0, 1),(0, 3),(1, 1),(1, 2),(1, 0),(2, 1),(2, 2),(2, 3)]

空格 #0 不能够和 2,1,3 有相同的颜色或者数字或者在同一个2x2的方格中，空格 #1 不能够和 1,2,0 有相同的颜色或者数字或者在同一个2x2的方格中.

Results = [0,3,0] ，即空格 #0 = 0, 空格 #1 = 3, 空格 #2 = 0.

### 代码

```javascript
operation SolvePuzzle(numVertices : Int, size : Int, emptySquareEdges : (Int, Int)[], 
    startingNumberConstraints: (Int, Int)[]) : (Bool, Int[]) {
    // 尺寸=4x4网格
    let bitsPerColor = size == 9 ? 4 | 2;
    mutable oracle = ApplyVertexColoringOracle(numVertices, bitsPerColor, emptySquareEdges, startingNumberConstraints, _, _);
    if (size == 9) {
        //虽然我们可以在9x9上使用ApplyVertexColoringOracle，
        //但我们必须对每种颜色添加限制，以禁止颜色8到15。
        //这可以通过将这些添加到startNumberConstraints来实现。
        //但是，这并不能很好地适应模拟器，因此我们使用
        //ApplyVertexColoringOracle4Bit9Color，它内置了9种颜色限制。
        set oracle = ApplyVertexColoringOracle4Bit9Color(numVertices, emptySquareEdges, startingNumberConstraints, _, _);
    } elif (size != 4) {
        fail $"Cannot set size {size}: only a grid size of 4x4 or 9x9 is supported";
    }
    let numIterations = NIterations(bitsPerColor * numVertices);
    Message($"Running Quantum test with #Vertex = {numVertices}");
    Message($"   Bits Per Color = {bitsPerColor}");
    Message($"   emptySquareEdges = {emptySquareEdges}");
    Message($"   startingNumberConstraints = {startingNumberConstraints}");
    Message($"   Estimated #iterations needed = {numIterations}");
    Message($"   Size of Sudoku grid = {size}x{size}");
    let coloring = FindColorsWithGrover(numVertices, bitsPerColor, numIterations, oracle);

    Message($"Got Sudoku solution: {coloring}");
    if (IsSudokuSolutionValid(size, emptySquareEdges, startingNumberConstraints, coloring)) {
        Message($"Got valid Sudoku solution: {coloring}");
        return (true, coloring);
    } else {
        Message($"Got invalid Sudoku solution: {coloring}");
        return (false, coloring);
    }
}
```

# Oracle准备

## 函数 ApplyColorEqualityOracle

N位颜色等价oracle（无需额外的量子位）

### 输入

- color0

    第一个颜色。

- color1
  
    第二个颜色。

### 输出

两个颜色一样，则输出为1。

### 代码

```javascript
operation ApplyColorEqualityOracle (color0 : Qubit[], color1 : Qubit[], target : Qubit) : Unit is Adj+Ctl {
    within {
        for ((q0, q1) in Zip(color0, color1)) {
            //q1<=q0 XOR q1
            CNOT(q0, q1);
        }
    } apply {
        //如果观测出来全为0，则表示两个寄存器中的值相等
        (ControlledOnInt(0, X))(color1, target);
    }
}
```

## 函数 ApplyVertexColoringOracle

验证顶点着色的`Oracle`，包括非量子位顶点的颜色约束。

### 输入

- numVertices

    图中顶点的数目。

- bitsPerColor

    每种颜色的位数，例如每种颜色2位，允许4种颜色。

- edges

    （Vertex#，Vertex#）数组，指定不同颜色的顶点。

- startingColorConstraints

    指定顶点不允许的颜色的数组（Vertex#，Color）。

### 输出

如果控制寄存器状态与位掩码`bits`相对应，则对目标寄存器应用`oracle`操作。

### 代码

```javascript
operation ApplyVertexColoringOracle (numVertices : Int, bitsPerColor : Int, edges : (Int, Int)[],  
    startingColorConstraints : (Int, Int)[], 
    colorsRegister : Qubit[], 
    target : Qubit) : Unit is Adj+Ctl {
    let nEdges = Length(edges);
    let nStartingColorConstraints = Length(startingColorConstraints);
    // 我们正在寻找一种解决方案：
    // (a) 没有两端颜色相同的边 
    // (b) 没有颜色违反起始颜色约束的顶点
    using ((edgeConflictQubits, startingColorConflictQubits) = (Qubit[nEdges], Qubit[nStartingColorConstraints])) {
        within {
            ConstrainByEdgeAndStartingColors(colorsRegister, edges, startingColorConstraints, edgeConflictQubits, startingColorConflictQubits, bitsPerColor);
        } apply {
            // 如果没有冲突（所有量子位都处于0状态），则顶点着色是有效的。
            (ControlledOnInt(0, X))(edgeConflictQubits + startingColorConflictQubits, target);
        }
    }
}
//按边和起始颜色约束进行约束
operation ConstrainByEdgeAndStartingColors (colorsRegister : Qubit[], edges : (Int, Int)[], startingColorConstraints : (Int, Int)[],
    edgeConflictQubits : Qubit[], startingColorConflictQubits : Qubit[], bitsPerColor: Int): Unit is Adj+Ctl {
    for (((start, end), conflictQubit) in Zip(edges, edgeConflictQubits)) {
        // 检查边的端点是否具有不同的颜色：
        // 应用ColorEqualityOracle_Nbit `oracle`；
        // 如果颜色相同，结果为1，表示冲突
        ApplyColorEqualityOracle(
            colorsRegister[start * bitsPerColor .. (start + 1) * bitsPerColor - 1], colorsRegister[end * bitsPerColor .. (end + 1) * bitsPerColor - 1], conflictQubit);
    }
    for (((cell, value), conflictQubit) in
        Zip(startingColorConstraints, startingColorConflictQubits)) {
        // 检查单元格是否与起始颜色冲突。
        (ControlledOnInt(value, X))(colorsRegister[
            cell * bitsPerColor .. (cell + 1) * bitsPerColor - 1], conflictQubit);
    }

}

```

### 举个例子

这是个 4x4 的数独谜题：

``` 
    -----------------
    |   |   | 2 | 3 |
    -----------------
    |   |   | 0 | 1 |
    -----------------
    | 1 | 2 | 3 | 0 |
    -----------------
    | 3 | 0 | 1 | 2 |
    -----------------
```

挑战是用0到3的数字填充空正方形，这些数字在行、列和左上角2x2平方中是唯一的。这是一个图着色问题，其中颜色为0到3，空单元格是顶点。顶点可以定义为：

```
    -----------------
    | 0 | 1 |   |   |
    -----------------
    | 2 | 3 |   |   |
    -----------------
    |   |   |   |   |
    -----------------
    |   |   |   |   |
    -----------------

图可以表示为这个

 0---1
 | X |
 1---2
```

i.e. 每个顶点都与每个顶点相连其他。另外，我们要求：
- 顶点0和1没有得到颜色2和3。
- 顶点2和3没有得到颜色3和0。
- 顶点0和2不会得到颜色1和3。
- 顶点1和3没有得到颜色2和0。

这将导致边（顶点的颜色不能相同）：

`edges = [(1, 0),(2, 0),(3, 0),(3, 1),(3, 2)]`

这意味着顶点1不能与顶点0具有相同的颜色等。

并且 startingColorConstraints = [(0, 1),(0, 3),(0, 2),(1, 2),(1, 0),(1, 3),(2, 1),(2, 3),(2, 0),(3, 2),(3, 0),(3, 1)]

这就是说：

- 顶点0不允许有值1、3、2
- 顶点1不允许有值2、0、3
- 顶点2不允许有值1、3、0
- 顶点3不允许有值2、0、1

有效的图着色解决方案是：[0,1,2,3]，即vextex 0的颜色为0，顶点1的颜色为1，以此类推。

## 函数 ApplyVertexColoringOracle4Bit9Color

Oracle用于验证顶点着色，包括来自非量子位顶点的颜色约束。这与pplyVertexColoringOracle相同，但硬编码为每种颜色4位，并且限制颜色限制为0到8。

### 输入

- numVertices

    图中顶点的数目。

- edges

    （Vertex#，Vertex#）数组，指定不同颜色的顶点。

- startingColorConstraints

    指定顶点不允许的颜色的数组（Vertex#，Color）。

- colorsRegister

    颜色寄存器。

- target

    行动的目标。

### 输出

如果控制寄存器状态与位掩码`bits`相对应，则对目标寄存器应用`oracle`操作。

### 举例子

考虑下面的9x9数独游戏：

```
   -------------------------------------
   |   | 6 | 2 | 7 | 8 | 3 | 4 | 0 | 1 |
   -------------------------------------
   | 8 |   | 1 | 6 | 2 | 4 | 3 | 7 | 5 |
   -------------------------------------
   | 7 | 3 | 4 | 5 | 0 | 1 | 8 | 6 | 2 |
   -------------------------------------
   | 6 | 8 | 7 | 1 | 5 | 0 | 2 | 4 | 3 |
   -------------------------------------
   | 4 | 1 | 5 | 3 | 6 | 2 | 7 | 8 | 0 |
   -------------------------------------
   | 0 | 2 | 3 | 4 | 7 | 8 | 1 | 5 | 6 |
   -------------------------------------
   | 3 | 5 | 8 | 0 | 1 | 7 | 6 | 2 | 4 |
   -------------------------------------
   | 1 | 7 | 6 | 2 | 4 | 5 | 0 | 3 | 8 |
   -------------------------------------
   | 2 | 4 | 0 | 8 | 3 | 6 | 5 | 1 | 7 |
   -------------------------------------
```

挑战是用行、列和左上3x3正方形中唯一的0到8填充空方块。这是一个图的着色问题，颜色是0到8，空单元格是顶点。顶点可以定义为

```  
    -----------------
    | 0 |   |   |   | ...
    -----------------
    |   | 1 |   |   | ...
    -----------------
    |   |   |   |   | ...
    ...
```
图可以表示为
```
    0---1 
```

此外，我们还要求

  - 顶点0不能具有值6,2,7,8,3,4,0,1（行约束）或值8,7,6,4,0,3,1,2（列约束）

  - 顶点1不能值8,1,6,2,4,3,7,5（行约束）或值6,3,8,1,2,5,7,4（列约束）
  
这将导致边（顶点的颜色不能相同）

边=[（1，0）]这意味着顶点1不能与顶点0具有相同的颜色

并且startingColorConstraints=[（0，8），（0，7），（0，6），（0，4），（0，0），（0，3），（0，1），（0，2），（1，6），（1，3），（1，8），（1，1），（1，2），（1，5），（1，7），（1，4）]
找到的颜色必须介于0到8之间，这需要每个颜色4位。
有效的图着色解决方案是：[5,0]，即vextex 0的颜色为5，顶点1的颜色为0。

### 代码

```javascript
operation ApplyVertexColoringOracle4Bit9Color (numVertices : Int, edges : (Int, Int)[],  startingColorConstraints : (Int, Int)[], colorsRegister : Qubit[], target : Qubit) : Unit is Adj+Ctl {
    let nEdges = Length(edges);
    let bitsPerColor = 4; // 颜色用4位表示，有9个颜色
    let nStartingColorConstraints = Length(startingColorConstraints);
    //我们正在寻找一种解决方案：
    //（a）两端无相同颜色的边缘
    //（b）没有顶点的颜色违反起始颜色约束。
    using ((edgeConflictQubits, startingColorConflictQubits, vertexColorConflictQubits) = 
        (Qubit[nEdges], Qubit[nStartingColorConstraints], Qubit[numVertices])) {
        within {
            ConstrainByEdgeAndStartingColors(colorsRegister, edges, startingColorConstraints, 
                edgeConflictQubits, startingColorConflictQubits, bitsPerColor);
            let zippedColorAndConfictQubit = Zip(
                Partitioned(ConstantArray(numVertices, bitsPerColor), colorsRegister),
                vertexColorConflictQubits);
            for ((color, conflictQubit) in zippedColorAndConfictQubit) {
                // 只允许0到8之间的颜色，即如果位3=1，则位2..0必须为000。
                using (tempQubit = Qubit()) {
                    within {
                        ApplyOrOracle(color[0 .. 2], tempQubit);
                    } apply{
                        // 颜色的最高有效位与或最低有效位之比。
                        // 如果颜色>8，则将conflictQubit设置为1。
                        CCNOT(color[3], tempQubit, conflictQubit);
                    }
                }
            }
        } apply {
            // 如果没有冲突（所有量子位都处于0状态），则顶点着色是有效的。
            (ControlledOnInt(0, X))(edgeConflictQubits + startingColorConflictQubits + vertexColorConflictQubits, target);
        }
    }
}
```

## 函数 ApplyOrOracle

用于查询寄存器中任意数量的量子位的OR oracle。

### 输入

- queryRegister

    要查询的Qubit寄存器。

- target

    用于存储oracle结果的目标量子位。

### 代码

```javascript
operation ApplyOrOracle (queryRegister : Qubit[], target : Qubit) : Unit is Adj {
    // x₀ ∨ x₁ = ¬ (¬x₀ ∧ ¬x₁)
    // 首先，如果两个量子位都处于| 0⟩状态，则翻转目标。
    (ControlledOnInt(0, X))(queryRegister, target);
    // 然后再次翻转目标取反。
    X(target);
}
```

# 用Grover的搜索寻找解

## 函数 FindColorsWithGrover

用Grover的搜索寻找顶点着色。

### 输入

- numVertices

    图中顶点的数目。

- bitsPerColor

    每种颜色的位数。

- maxIterations

    估计所需的最大迭代次数。

- oracle

    用于寻找解的`oracle`

### 输出

提供每个顶点颜色的Int数组。

### 提示

SolveSATWithGrover Kata中的原始实现请查看[https://github.com/microsoft/QuantumKatas/tree/main/SolveSATWithGrover]。

### 代码

```javascript
operation FindColorsWithGrover (numVertices : Int, bitsPerColor : Int, maxIterations : Int,oracle : ((Qubit[], Qubit) => Unit is Adj)) : Int[] {
    // 此任务类似于SolveSATWithGrover kata的2.2版本，
    // 但正确解决方案的百分比可能更高。
    mutable coloring = new Int[numVertices];

    // 注意，着色寄存器的量子比特数是顶点数的两倍
    //（每个顶点的比特数或量子比特数）。
    using ((register, output) = (Qubit[bitsPerColor * numVertices], Qubit())) {
        mutable correct = false;
        mutable iter = 1;
        // 尝试一次迭代，如果失败，则再次尝试一次迭代并重复，
        // 直到达到最大迭代次数。
        repeat {
            Message($"Trying search with {iter} iterations...");
            ApplyGroversAlgorithmLoop(register, oracle, iter);
            let res = MultiM(register);
            // 若要检查结果是否正确，请在测量后对寄存器加辅助项应用oracle。
            oracle(register, output);
            if (MResetZ(output) == One) {
                set correct = true;
                // 读出颜色。
                set coloring = MeasureColoring(bitsPerColor, register);
            }
            ResetAll(register);
        } until (correct or iter > maxIterations)  
        fixup {
            set iter += 1;
        }
        if (not correct) {
            fail "Failed to find a coloring.";
        }
    }
    return coloring;
}
```


## 函数 ApplyPhaseOrOracle

Grover算法迭代循环

### 输入

- oracle

    `Oracle`将标记有效的解决方案。

### 提示

SolveSATWithGrover Kata中的原始实现请查看[https://github.com/microsoft/QuantumKatas/tree/main/SolveSATWithGrover]。

### 代码

```javascript
operation ApplyPhaseOracle (oracle : ((Qubit[], Qubit) => Unit is Adj), register : Qubit[]) : Unit is Adj {

    using (target = Qubit()) {
        within {
            // 将目标置于|-⟩状态。
            X(target);
            H(target);
        } apply {
            // 应用标记oracle；由于目标处于|-⟩状态，
            // 如果寄存器满足oracle条件，则翻转目标将对状态应用-1因子。
            oracle(register, target);
        }
        //我们把目标放回|0⟩这样我们就可以把它放回去了。
    }
}
```

# 循环迭代

## 函数 NIterations

估计求解所需的迭代次数。

### 输入

- nQubits

    正在使用的量子位数。

### 评述

对于振幅放大问题，只有一个正确的解决方案，这是正确的，但需要在有多个解决方案时进行调整。

### 代码

```javascript
function NIterations(nQubits : Int) : Int {
    let nItems = 1 <<< nQubits; // 2^numQubits
    // 计算迭代次数
    let angle = ArcSin(1. / Sqrt(IntAsDouble(nItems)));
    let nIterations = Round(0.25 * PI() / angle - 0.5);
    return nIterations;
}
```

## 函数 ApplyGroversAlgorithmLoop

使用Grover算法迭代循环.

### 输入

- register

    量子比特的寄存器。

- oracle

    `量子黑箱`定义了解决方案。

- iterations

    要尝试的迭代次数。

### 输出

单一实现Grover搜索算法。

### 代码

```javascript
operation ApplyGroversAlgorithmLoop (register : Qubit[], 
    oracle : ((Qubit[], Qubit) => Unit is Adj), iterations : Int) : Unit {
    let applyPhaseOracle = ApplyPhaseOracle(oracle, _);
    ApplyToEach(H, register);
    for (_ in 1 .. iterations) {
        applyPhaseOracle(register);
        within {
            ApplyToEachA(H, register);
            ApplyToEachA(X, register);
        } apply {
            Controlled Z(Most(register), Tail(register));
        }
    }
}
```

# 测量寄存器

## 函数 MeasureColor

测量量程寄存器的内容并将其转换为颜色（整数）。

### 输入

- register

    要测量的量子位寄存器。

### 代码

```javascript
operation MeasureColor (register : Qubit[]) : Int {
    return MeasureInteger(LittleEndian(register));
}
```

## 函数 MeasureColoring

从寄存器中读取颜色。

### 输入

- bitsPerColor

    每种颜色的位数。

- register

    要测量的量子位寄存器。

### 代码

```javascript
operation MeasureColoring (bitsPerColor : Int, register : Qubit[]) : Int[] {
    let numVertices = Length(register) / bitsPerColor;
    let colorPartitions = Partitioned(ConstantArray(numVertices - 1, bitsPerColor), register);
    return ForEach(MeasureColor, colorPartitions);
}
```

# 经典计算用以验证

## 函数 IsSudokuSolutionValid

检验填在空白位置的数字是否符合数独规则——满足行列的约束。

### 输入

- size

    数独大小。

- edges

    传统的边传递给图着色算法，在我们的例子中，是空的拼图方块。

    这些边定义空单元格之间的任何“同一行”、“同一列”、“同一子网格”关系。

- startingNumberConstraints

    当我们开始的时候，由于数字已经在拼图中，对空格子的限制。

- colors

    每个空格的整数数组，即数独的解。

### 输出

如果符合数独规则，那么返回`真`。

### 代码

```javascript
function IsSudokuSolutionValid (size : Int, edges : (Int, Int)[],startingNumberConstraints : (Int, Int)[], colors : Int[]) : Bool 
{
    if (Any(GreaterThanOrEqualI(_, size), colors)) { return false; }
    if (Any(EqualI, edges)) { return false; }
    for ((index, startingNumber) in startingNumberConstraints) {
        if (colors[index] == startingNumber) {
            return false;
        }
    }
    return true;
}
```



