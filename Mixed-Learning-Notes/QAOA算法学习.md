**<center><font face="微软雅黑" color=black size=10>QAOA算法学习</font></center>**
代码（作者：Microsoft）：

https://github.com/microsoft/Quantum/tree/main/samples/simulation/qaoa

翻译与整理：B20200342吕征宇 @北京科技大学 2020/11/15

# 问题提出并制定约束条件

先说明一下，这就是个组合优化问题（中的travelling salesman problem，Max Cut问题也是一个组合优化问题）。

这个问题属于NP-Hard问题，精确算法（暴力搜索）需要$O(n!)$的时间，而动态编程则需要$O(n^{2}2^{n})$的时间；最小匹配法（The Algorithm of Christofides and Serdyukov）可以在$O(n^{3})$的时间内完成，而优化了迭代之后，使用Pairwise exchange的方法则可以达到$O(n log(n))$的时间复杂度（这些优化虽然可解了，但是还是在极大节点的空域中只能得到近似解）。

而我们知道，由于量子的天然并行性，可以使计算机处于天然的并行状态下，随着量子位的增加，算力呈现指数级增加，因此在这类问题上面可以有一个很大的加速，在未来能够控制量子精度的，使其足够高时候可以得到比现在的近似算法更精确的解，并且拥有更快的计算速率；因此我们使用量子加速便是为了这个目的进行的。

依据 http://quantumalgorithmzoo.org/traveling_santa/ 中描述的圣诞老人问题，将验证函数定义在了经典计算范畴，总共有两个，CalculatedCost计算代价，而IsSatisfactory来判定是否满足题设中抽取出来的数学条件。
## 函数 CalculatedCost
### 概述
计算给定结果的总成本。

这是经典计算方法而非量子方法。

总成本计算公式所示：
$$C=\sum^{5}_{j=0}C_{j}x_{j}$$
#### 输入
##### segmentCosts
路径的成本数组
##### usedSegments
路径的使用状况数组

#### Output
##### finalCost
给定路径的成本结果
#### 代码
```
function CalculatedCost(segmentCosts : Double[], usedSegments : Bool[]) : Double {
    mutable finalCost = 0.0;
    for ((cost, segment) in Zipped(segmentCosts, usedSegments)) {
        set finalCost += segment ? cost | 0.0;
    }
    return finalCost;
}
```

## 函数 IsSatisfactory
### 概述
最终检查以确定使用的段是否满足我们已知的约束。此函数用于考虑一个包含6个线段和三个有效连通路径的图。

这是经典计算方法而非量子方法。

这个方法要通过以下4个约束条件，来判断路径是否满足题设条件：
$$\sum^{5}_{j=0}x_{j}=4$$
$$x_{0}=x_{2}$$
$$x_{1}=x_{3}$$
$$x_{4}=x_{5}$$
#### 输入
##### numSegments
图中的分段数
##### usedSegments
使用了哪些段的数组
#### Output
##### output
是否满足条件(布尔值)
#### 代码
```
function IsSatisfactory(numSegments: Int, usedSegments : Bool[]) : Bool {
    EqualityFactI(numSegments, 6, 
        "Currently, IsSatisfactory only supports constraints for 6 segments."
    );
    mutable hammingWeight = 0;
    for (segment in usedSegments) {
        set hammingWeight += segment ? 1 | 0;
    }
    if (hammingWeight != 4 
        or usedSegments[0] != usedSegments[2] 
        or usedSegments[1] != usedSegments[3] 
        or usedSegments[4] != usedSegments[5]) {
        return false;
    }
    return true;
}
```
# 问题转化
通过已经抽取出来的数学条件，作者将问题转化为量子计算中优化问题，是将一些变量映射到时间域上进行优化。
## 函数 PerformQAOA
### 概述
对这个 Ising Hamiltonian执行QAOA算法，这个是个总的流程，可以看作整个量子电路的循环运行。
#### 输入
##### numSegments
图中的总边数
##### weights
将Hamiltonian参数（“权重”）作为一个数组，其中每个元素对应于qubit状态$j$的参数$h_j$
##### couplings
将哈密顿耦合参数实例化为一个数组，其中每个元素对应于量子位状态$i$和$j$之间的参数$j_{ij}$
##### timeX
Pauli-X算子的时间演化
##### timeZ
Pauli-Z算子的时间演化
#### 代码
```
operation PerformQAOA(
        numSegments : Int, 
        weights : Double[], 
        couplings : Double[], 
        timeX : Double[], 
        timeZ : Double[]
) : Bool[] {
    EqualityFactI(Length(timeX), Length(timeZ), "timeZ and timeX are not the same length");

    // 运行 QAOA 电路
    mutable result = new Bool[numSegments];
    using (x = Qubit[numSegments]) {
        // 将改6个量子进行组合，并且初始化为均匀概率
        ApplyToEach(H, x); 
        for ((tz, tx) in Zipped(timeZ, timeX)) {
            // 执行 Exp(-i H_C tz)
            ApplyInstanceHamiltonian(numSegments, tz, weights, couplings, x); 
            // 执行 Exp(-i H_0 tx)
            ApplyDriverHamiltonian(tx, x); 
        }
        //计算基础上的度量
        return ResultArrayAsBoolArray(MultiM(x)); 
    }
}
```
## 函数  ApplyDriverHamiltonian
### 简介
此操作将X旋转应用于每个量子比特。我们可以把它看作是通过应用Hamiltonian对所有X旋转求和得到的时间演化。因为在量子过程中，优化搜索相当于对量子进行干涉的动态演化过程。
#### 数学描述
Driver Hamiltonian的定义：
对于时间t而言，有
$$H = - \sum_{i} X_{i} $$
#### 输入
##### 时间
X旋转演化的时间
##### 目标
目标量子寄存器
#### 代码
```
operation ApplyDriverHamiltonian(time: Double, target: Qubit[]) : Unit is Adj + Ctl {
    ApplyToEachCA(Rx(-2.0 * time, _), target);
}
```
## 函数  ApplyInstanceHamiltonian
### 概述
根据这个例子，Hamiltonian适用于旋转。我们可以把它看作是Ising Hamiltonian引起的时间t的哈密顿时间演化。对$J_{ij}$缩放的所有Pauli-Z运算$Z_i$和$Z_j$的所有连通对上的Ising Hamiltonian进行求和再加上所有$h_i$缩放的$Z_i$的和。这个便是问题2中最后确定的好的代价函数。
#### 数学描述
Ising Hamiltonian 可以表示为:
$$H_{C}=\sum_{ij} J_{ij} Z_i Z_j + \sum_i h_i Z_i$$
#### 输入
##### time
演化的时间点.
##### weights
旅行圣诞老人问题的Ising约束场（“权重”编码）。
##### coupling
旅行圣诞老人问题的Ising对（“惩罚”编码）。
#### target
Ising Hamiltonian中的旋转值进行编码的量子位寄存器。
#### 代码
```
operation ApplyInstanceHamiltonian(
    numSegments : Int,
    time : Double, 
    weights : Double[], 
    coupling : Double[],
    target : Qubit[]
) : Unit {
    using (auxiliary = Qubit()) {
        for ((h, qubit) in Zipped(weights, target)) {
            Rz(2.0 * time * h, qubit);
        }
        for (i in 0..5) {
            for (j in i + 1..5) {
                within {
                    CNOT(target[i], auxiliary);
                    CNOT(target[j], auxiliary);
                } apply {
                    Rz(2.0 * time * coupling[numSegments * i + j], auxiliary);
                }
            }
        }
    }
}
```
## 函数 HamiltonianWeights
### 概述
根据给定的代价和惩罚计算Hamiltonian参数。
$$h_{j}=4p−\frac{1}{2}C, 其中p=20$$
该函数计算结果如下：
$$h_0=77.65$$
$$h_1=75.455$$
$$h_2=75.485$$
$$h_3=77.15$$
$$h_4=75.99$$
$$h_5=79.145$$
#### 输入
##### segmentCosts
每条边的代价
##### penalty
对不符合约束条件的情况进行处罚
#### Output
##### weights
Hamiltonian参数（“权重”）作为一个数组，其中每个元素对应于量子比特状态$j$的参数$h_j$
##### numSegments
图中描述可能路径的段数
#### 代码
```
function HamiltonianWeights(
    segmentCosts : Double[], 
    penalty : Double, 
    numSegments : Int
) : Double[] {
    mutable weights = new Double[numSegments];
    for (i in 0..numSegments - 1) {
        set weights w/= i <- 4.0 * penalty - 0.5 * segmentCosts[i];
    }
    return weights;
}
```
## 函数 HamiltonianCouplings
### 概述
依据给定的罚函数计算Hamiltonian对参数：基于给定的代价和惩罚计算Hamiltonian耦合参数，$J_{ij}$的大多数元素都等于2倍惩罚，因此将所有元素设置为该值，然后覆盖异常。这个是目前实现的6量子位的例子。
令其他的$J_{i<j}=2p$以及$J_{02}=J_{13}=J_{45}=p$，作者此时使用$p=20$。该函数计算可以得到以下的结果：

```math
J_{02}=J_{13}=J_{45}=20

J_{01}=J_{03}=J_{04}=J_{12}=J_{14}=J_{15}=J_{23}=J_{24}=J_{25}=J_{34}=J_{35}=40
```
#### 输入
##### penalty
对不符合约束条件的情况进行惩罚
##### numSegments
图中描述可能路径的段数
#### Output
##### coupling
哈密顿耦合参数作为一个数组，其中每个元素对应于量子比特状态$i$和$j$之间的一个参数$J_{ij}$。
#### 代码
```
function HamiltonianCouplings(penalty : Double, numSegments : Int) : Double[] {
    EqualityFactI(numSegments, 6, 
        "Currently, HamiltonianCouplings only supports given constraints for 6 segments."
    );
    return ConstantArray(numSegments * numSegments, 2.0 * penalty)
        w/ 2 <- penalty
        w/ 9 <- penalty
        w/ 29 <- penalty;
}
```
# 运行函数
## 主函数    RunQAOATrials
### 简介
运行QAOA，在6个量子比特上进行给定数量的试验。此示例基于
关于圣诞老人的旅行问题：http://quantumalgorithmzoo.org/traveling_santa/.
报告旅行圣诞老人问题的最佳行程，以及多少次运行结果得到答案。这通常会在大约71%的时间内返回最佳解决方案。
#### 输入
##### 运行次数
QAOA算法尝试的次数.
#### 代码
```
@EntryPoint()
operation RunQAOATrials(numTrials : Int) : Unit {
    let penalty = 20.0;
    let segmentCosts = [4.70, 9.09, 9.03, 5.70, 8.02, 1.71];
    let timeX = [0.619193, 0.742566, 0.060035, -1.568955, 0.045490];
    let timeZ = [3.182203, -1.139045, 0.221082, 0.537753, -0.417222];
    let limit = 1E-6;
    let numSegments = 6;
    mutable bestCost = 100.0 * penalty;
    mutable bestItinerary = [false, false, false, false, false];
    mutable successNumber = 0;
    let weights = HamiltonianWeights(segmentCosts, penalty, numSegments);
    let couplings = HamiltonianCouplings(penalty, numSegments);
    for (trial in 0..numTrials) {
        let result = PerformQAOA(
            numSegments, 
            weights, 
            couplings, 
            timeX, 
            timeZ
        );
        let cost = CalculatedCost(segmentCosts, result);
        let sat = IsSatisfactory(numSegments, result);
        Message($"result = {result}, cost = {cost}, satisfactory = {sat}");
        if (sat) {
            if (cost < bestCost - limit) {
                // New best cost found - update
                set bestCost = cost;
                set bestItinerary = result;
                set successNumber = 1;
            } elif (AbsD(cost - bestCost) < limit) {
                set successNumber += 1;
            }
        }
    }
    let runPercentage = IntAsDouble(successNumber) * 100.0 / IntAsDouble(numTrials);
    Message("Simulation is complete\n");
    Message($"Best itinerary found: {bestItinerary}, cost = {bestCost}");
    Message($"{runPercentage}% of runs found the best itinerary\n");
}
```