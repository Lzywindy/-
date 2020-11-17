

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

i.e. every vertex is connected to each other.
Additionally, we require that:

   - vertices 0 and 1 do not get colors 2 and 3.
   - vertices 2 and 3 do not get colors 3 and 0.
   - vertices 0 and 2 do not get colors 1 and 3.
   - vertices 1 and 3 do not get colors 2 and 0.

This results in edges (vertices that can not be same color):
`edges = [(1, 0),(2, 0),(3, 0),(3, 1),(3, 2)]`

This is saying that vertex 1 can not have same color as vertex 0 etc.

and startingColorConstraints = [(0, 1),(0, 3),(0, 2),(1, 2),(1, 0),(1, 3),(2, 1),(2, 3),(2, 0),(3, 2),(3, 0),(3, 1)]

This is saying that:

- vertex 0 is not allowed to have values 1,3,2
- vertex 1 is not allowed to have values 2,0,3
- vertex 2 is not allowed to have values 1,3,0
- vertex 3 is not allowed to have values 2,0,1

A valid graph coloring solution is: [0,1,2,3] i.e. vextex 0 has color 0, vertex 1 has color 1 etc.

# Summary
Oracle for verifying vertex coloring, including color constraints 
from non qubit vertices. This is the same as ApplyVertexColoringOracle, 
but hardcoded to 4 bits per color and restriction that colors are 
limited to 0 to 8.

# Input
## numVertices
The number of vertices in the graph.
## edges
The array of (Vertex#,Vertex#) specifying the Vertices that can not 
be the same color.
## startingColorConstraints
The array of (Vertex#,Color) specifying the dissallowed colors for vertices.
## colorsRegister
The color register.
## target
The target of the operation.

# Output
A unitary operation that applies `oracle` on the target register if the control 
register state corresponds to the bit mask `bits`.

# Example
Consider the following 9x9 Sudoku puzzle:
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
The challenge is to fill the empty squares with numbers 0 to 8
that are unique in row, column and the top left 3x3 square
This is a graph coloring problem where the colors are 0 to 8
and the empty cells are the vertices. The vertices can be defined as
```  
    -----------------
    | 0 |   |   |   | ...
    -----------------
    |   | 1 |   |   | ...
    -----------------
    |   |   |   |   | ...
    ...
```
The graph is
```
    0---1 
```
Additionally, we also require that 
   - vertex 0 can not have value 6,2,7,8,3,4,0,1 (row constraint)
                        or value 8,7,6,4,0,3,1,2 (col constraint)
   - vertex 1 can not value 8,1,6,2,4,3,7,5 (row constraint)
                   or value 6,3,8,1,2,5,7,4 (col constraint)
This results in edges (vertices that can not be same color)
edges = [(1, 0)]
This is saying that vertex 1 can not have same color as vertex 0
and startingColorConstraints = [(0, 8),(0, 7),(0, 6),(0, 4),(0, 0),(0, 3),
 (0, 1),(0, 2),(1, 6),(1, 3),(1, 8),(1, 1),(1, 2),(1, 5),(1, 7),(1, 4)]
The colors found must be from 0 to 8, which requires 4 bits per color.
A valid graph coloring solution is: [5,0]
i.e. vextex 0 has color 5, vertex 1 has color 0.
### 代码

```javascript
operation ApplyVertexColoringOracle4Bit9Color (numVertices : Int, edges : (Int, Int)[],  startingColorConstraints : (Int, Int)[], colorsRegister : Qubit[], target : Qubit) : Unit is Adj+Ctl {
    let nEdges = Length(edges);
    let bitsPerColor = 4; // 4 bits per color
    let nStartingColorConstraints = Length(startingColorConstraints);
    // we are looking for a solution that:
    // (a) has no edge with same color at both ends and 
    // (b) has no Vertex with a color that violates the starting color constraints.
    using ((edgeConflictQubits, startingColorConflictQubits, vertexColorConflictQubits) = 
        (Qubit[nEdges], Qubit[nStartingColorConstraints], Qubit[numVertices])) {
        within {
            ConstrainByEdgeAndStartingColors(colorsRegister, edges, startingColorConstraints, 
                edgeConflictQubits, startingColorConflictQubits, bitsPerColor);
            let zippedColorAndConfictQubit = Zip(
                Partitioned(ConstantArray(numVertices, bitsPerColor), colorsRegister),
                vertexColorConflictQubits);
            for ((color, conflictQubit) in zippedColorAndConfictQubit) {
                // Only allow colors from 0 to 8 i.e. if bit #3 = 1, then bits 2..0 must be 000.
                using (tempQubit = Qubit()) {
                    within {
                        ApplyOrOracle(color[0 .. 2], tempQubit);
                    } apply{
                        // AND color's most significant bit with OR of least significant bits. 
                        // This will set conflictQubit to 1 if color > 8.
                        CCNOT(color[3], tempQubit, conflictQubit);
                    }
                }
            }
        } apply {
            // If there are no conflicts (all qubits are in 0 state), the vertex coloring is valid.
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

## 函数 ApplyPhaseOracle

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

# Summary

Solve a Sudoku puzzle using Grover's algorithm.

# Description

Sudoku is a graph coloring problem where graph edges must connect nodes of different colors. In our case, graph nodes are puzzle squares and colors are the Sudoku numbers. Graph edges are the constraints preventing squares from having the same values. To reduce the number of qubits needed, we only use qubits for empty squares.
We define the puzzle using 2 data structures:

  - A list of edges connecting empty squares
  - A list of constraints on empty squares to the initial numbers 
    in the puzzle (starting numbers)
The code works for both 9x9 Sudoku puzzles, and 4x4 Sudoku puzzles. 
This description will use a 4x4 puzzle to make it easier to understand.
The 4x4 puzzle is solved with number 0 to 3 instead of 1 to 4. 
This is because we can encode 0-3 with 2 qubits.
However, the same rules apply:
   - The numbers 0 to 3 may only appear once per row, column and 2x2 sub squares.

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

In the above example, the edges/constraints for the top row are:

```
   _________
  | ______   \                   _____   
  || __   \   \                  | __  \                        __
 _|||__\___\_ _\__         ______||__\___\__          _________|___\__ 
 |   | 1 |   | 3 |         |   | 1 |   | 3 |         |   | 1 |   | 3 | 
 -----------------         -----------------         -----------------

For the row above, the empty squares have indexes
_________________
| 0 |   | 1 |   |
-----------------
```

For this row the list of emptySquareEdges has only 1 entry:

emptySquareEdges = (0,1) i.e. empty square 0 can't have the same number as empty square 1.

The constraints on these empty squares to the starting numbers are:

startingNumberConstraints = (0,1)  (0,3)  (1,1)  (1,3)

This is a list of (empty square #, number it can't be).
i.e. empty square 0 can't have value 1 or 3, and empty square #1 can't have values 1 or 3.

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
    // for size = 4x4 grid
    let bitsPerColor = size == 9 ? 4 | 2;
    mutable oracle = ApplyVertexColoringOracle(numVertices, bitsPerColor, emptySquareEdges, startingNumberConstraints, _, _);
    if (size == 9) {
        // Although we could use ApplyVertexColoringOracle for 9x9, we would
        // have to add restrictions on each color to not allow colors 8 
        // thru 15. This could be achieved by adding these to 
        // startNumberConstraints. However, this did not scale well with 
        // the simulator, and so instead we use 
        // ApplyVertexColoringOracle4Bit9Color which has the 9 color 
        // restriction built in.
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