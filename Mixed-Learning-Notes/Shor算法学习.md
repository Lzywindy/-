**<center><font face="微软雅黑" size=10>Shor算法学习</font></center>**

代码（作者：Microsoft）：[https://github.com/microsoft/Quantum/tree/main/samples/algorithms/integer-factorization]

翻译与整理：B20200342吕征宇 @北京科技大学 2020/11/15

# 背景
来源于[https://baike.baidu.com/item/RSA%E7%AE%97%E6%B3%95/263310]

RSA公开密钥密码体制是一种使用不同的加密密钥与解密密钥，“由已知加密密钥推导出解密密钥在计算上是不可行的”密码体制。
在公开密钥密码体制中，加密密钥（即公开密钥）PK是公开信息，而解密密钥（即秘密密钥）SK是需要保密的。加密算法E和解密算法D也都是公开的。虽然解密密钥SK是由公开密钥PK决定的，但却不能根据PK计算出SK。

正是基于这种理论，1978年出现了著名的RSA算法，它通常是先生成一对RSA密钥，其中之一是保密密钥，由用户保存；另一个为公开密钥，可对外公开，甚至可在网络服务器中注册。为提高保密强度，RSA密钥至少为500位长，一般推荐使用1024位。这就使加密的计算量很大。为减少计算量，在传送信息时，常采用传统加密方法与公开密钥加密方法相结合的方式，即信息采用改进的DES或IDEA对话密钥加密，然后使用RSA密钥加密对话密钥和信息摘要。

RSA的安全性依赖于大数分解，但是否等同于大数分解一直未能得到理论上的证明，也并没有从理论上证明破译。RSA的难度与大数分解难度等价。因为没有证明破解RSA就一定需要做大数分解。假设存在一种无须分解大数的算法，那它肯定可以修改成为大数分解算法，即RSA的重大缺陷是无法从理论上把握它的保密性能如何，而且密码学界多数人士倾向于因子分解不是NPC问题。

通常求解大数分解（通常被认为是NP问题，但是现在有人证明其实是个P问题）最快需要 $O(e^{1.9(log\ N)^{\frac{1}{3}}(log\ log\ N)^{\frac{2}{3}}})$这种时间量级。而目前出现的Shor算法则是需要花费 $O((log\ N)^{3})$的时间，实现了指数级别的加速（最快的因数分解算法：普通数域筛选法）。

但从这个角度来说，量子计算并没有使得计算能从NP问题变成P问题。最关键的是现在的量子计算技术还处于发展中的萌芽期，来解决RSA至少需要1024来解决，然而现在量子计算机能够达到拥有1000位的计算比特已经很不错了（只有顶尖的实验室才有）。因此RSA的安全性并没有受到过大的威胁。

# 算法的原理
来源于[https://zh.wikipedia.org/wiki/%E7%A7%80%E7%88%BE%E6%BC%94%E7%AE%97%E6%B3%95]

## 解释

此算法包含两个部分。算法的第一部分是将因数分解问题转成查找一个函数的周期，而且这部分可以用传统方式实现。第二部分则是使用量子傅里叶变换来搜索这个函数的周期，而且这一部分是量子加速这整个算法的理由。

Shor算法包含两个部分：

1. 一个以传统电脑运作的简化算法，将因数分解简化成搜索阶问题。
2. 一个量子算法，解决搜索阶问题。

### 从周期得到因数

小于 $N$ 且互质于 $N$ 的整数组成一个有限大，且对乘法同余 $N$ 的群。

我们有一个属于这个群的整数 $a$。

既然这个群是有限的，$a$ 必有一个有限大的阶 $r$ , 也就是最小的正整数令 $a^{r}\equiv 1\ mod\ N$，因此可知，$N$是 $a^{r}− 1$的因数。先假设我们有能力获得$r$，而且$r$是偶数。

则 $a^{r}− 1=(a^{\frac{r}{2}}+1)(a^{\frac{r}{2}}-1)\equiv 0\ mod\ N => N|(a^{\frac{r}{2}}+1)(a^{\frac{r}{2}}-1)$。

$r$ 是令$a^{r}\equiv 1$最小的正整数，所以$(a^{\frac{r}{2}}-1)$必定不能整除于N。若$(a^{\frac{r}{2}}+1)$也不整除于N的话，则N必定与$(a^{\frac{r}{2}}+1)$或者$(a^{\frac{r}{2}}-1)$有一个非显然的公因数。

### 找寻周期
  
Shor的周期查找算法非常倚赖量子计算机同时计算许多状态的能力。物理学家称呼这个特性为这一些状态的"叠加"。在计算函数f的周期时，我们会同时估计这个函数的所有点。

不过，量子物理不允许我们直接获取这一些信息。测量会令观测结果塌陷到其中一个可能的结果，并摧毁其他可能性的存在。解决这些麻烦的问题，采用在这里我们使用量子傅里叶变换来达成。

因此Shor在这里必须解决三个"实现"的问题。这一些问题都必须要有很快的实现，或者说他们可以用 $log\ N$ 的多项式个数这么多量子门来实现。

1. 制造状态的叠加。
2. 以量子变换实现函数f。这一步需要多一倍的量子位进行辅助。
3. 进行量子傅里叶变换。这一步之后，观察结果会逼近周期r。

另一种解释Shor算法的方式是将之视为是量子相位估计算法的一种变形。从实现的角度来说，量子部分就在估算周期和相位上去了，而留给经典的计算来说，只是做了一个验证而已。

# 主函数执行部分

此示例包含实现Shor整数因式分解的量子算法的Q#代码。基于Stephane Beauregard的一篇论文，基本的模块化算法是在相位编码中实现的，他给出了一个量子电路需要 $2n+3$ 个量子位和$O(n^{3}log(n))$的基本量子门来进行对$n$比特数的因式分解。

## 主函数  FactorSemiprimeInteger

使用 Shor 算法去分解参数 $N$。
主要原理就是分解一个$N$，使得 $N=p \cdot q$。
找到$p > 1$ 、 $q > 1$，然后完成分解。

### 输入

- number  

    数字$N$：一个待分解半素数

- useRobustPhaseEstimation

    如果设置为true，则使用 Microsoft.Quantum.Characterization.RobustPhaseEstimation 以及 Microsoft.Quantum.Characterization.QuantumPhaseEstimation, 否则则使用其他的方法 

### 输出

- 一对数

    满足 $p > 1$ 、 $q > 1$ 与 $p \cdot q = N$

### 代码

```Javascript
operation FactorSemiprimeInteger(number : Int, useRobustPhaseEstimation : Bool) : (Int, Int) {
    //如果提供的数字是偶数，首先检查最简单的情况
    if (number % 2 == 0) {
        Message("An even number has been given; 2 is a factor.");
        return (number / 2, 2);
    }
    //如果我们找到了这些可变因素，那么这些可变变量将跟踪这些因素是什么。
    mutable foundFactors = false;
    //因子的默认值是 (1,1).
    mutable factors = (1, 1);
    repeat {
        //下一步试着把一个数的余素数猜成'number'
        //在区间[1,number-1]中获取一个随机整数
        let generator = DrawRandomInt(1, number - 1);
        //检查随机整数是否真的使用 Microsoft.Quantum.Math.IsCoprimeI。
        //如果为真，则使用量子算法进行周期查找。
        if (IsCoprimeI(generator, number)) {
            //使用 Microsoft.Quantum.Intrinsic.Message打印信息
            //表明我们正在用量子方法进行处理
            Message($"Estimating period of {generator}");
            //调用`generator` mod 'number'的量子周期查找算法。
            //这里我们可以选择使用哪种相位估计算法。
            let period = EstimatePeriod(generator, number, useRobustPhaseEstimation);
            //如果辗转相除成功，则设置标志和因数。
            set (foundFactors, factors) = MaybeFactorsFromPeriod(number, generator, period);
        }
        //在这个例子中，我们意外地猜到了除数。
        else {
            //使用Microsoft.Quantum.Math.GreatestCommonDivisorI
            //来发现最大公约数
            let gcd = GreatestCommonDivisorI(number, generator);
            //别忘了告诉用户我们很幸运，没有做任何量子运算。
            //通过Microsoft.Quantum.Intrinsic.Message.发送消息
            Message($"We have guessed a divisor of {number} to be {gcd} by accident.");
            //将标志“foundFactors”设置为true，表示我们成功地找到了因子。
            set foundFactors = true;
            set factors = (gcd, number / gcd);
        }
    }
    until (foundFactors)
    fixup {
        Message("The estimated period did not yield a valid factor, trying again.");
    }
    // 返回分解结果
    return factors;
}
```

# 量子计算部分

## 函数 ApplyOrderFindingOracle

将`target`转化为无符号小端存储的整数 $k$ 进行编码，并执行转换 $|k⟩↦| g^{p}k\mod\ N⟩$ ，其中$p$是“power”，$g$是“generator”，而$N$是“module”。

### 输入

- generator

    正被估计其乘法阶（周期）的无符号整数。必须是`number`的共质数。

- number

    这个就是原数 $N$

- power

    `generator`的幂，是`target`所需要的位数

- target

    小端存储的寄存器，位数是`generator`的指数，倍数则是通过对`number`取模得来的。

### 代码

```Javascript
operation ApplyOrderFindingOracle(generator : Int, number : Int, power : Int, target : Qubit[]): Unit is Adj + Ctl {
    //检查参数是否安全
    Fact(IsCoprimeI(generator, number), "`generator` and `number` must be co-prime");
    //我们用"黑盒" 去做顺序发现是
    //通过Microsoft.Quantum.Arithmetic.MultiplyByModularInteger封装的
    //其实现为 |x⟩ ↦ |x⋅a mod N ⟩
    MultiplyByModularInteger(ExpModI(generator, power, number), number, LittleEndian(target));
}
```

## 函数 EstimatePeriod

通过 Z mod `number`，来得到周期。

### 输入

- generator

    正被估计其乘法阶（周期）的无符号整数。必须是`number`的共质数。

- number

    这个就是原数 $N$。

- useRobustPhaseEstimation

    决定是否使用微软自己的稳定相位估计方法。

### 输出

通过`generator` mod `number`得到周期

### 代码

```Javascript
operation EstimatePeriod(generator : Int, number : Int, useRobustPhaseEstimation : Bool) : Int {
     Here we check that the inputs to the EstimatePeriod operation are valid.
    Fact(IsCoprimeI(generator, number), "`generator` and `number` must be co-prime");
    //存储到目前为止找到的生成器周期的除数的变量。
    mutable result = 1;
    //我们估计周期所依据的位数中的位数。
    let bitsize = BitSizeI(number);
    //EstimatePeriod操作通过找到小数s/r的近似k/2^（bits precision）来估计周期r，其中s是某个整数。
    //请注意，如果s和r有公约数，我们将最终恢复r的一个除数，而不是r本身。
    //然而，如果我们恢复足够多的r除数，我们很快就能恢复r本身。
    let bitsPrecision = 2 * bitsize + 1;
    mutable frequencyEstimate = 0;
    repeat {

        set frequencyEstimate = EstimateFrequency(
            generator, number, useRobustPhaseEstimation, bitsize 
        );

        if (frequencyEstimate != 0) {
            set result = PeriodFromFrequency(number,frequencyEstimate, bitsPrecision, result);
        }
        else {
            Message("The estimated frequency was 0, trying again.");
        }
    }
    until(ExpModI(generator, result, number) == 1)
    fixup {
        Message("The estimated period from continued fractions failed, trying again.");
    }

    return result;
}
```

## 函数 EstimateFrequency

通过 Z mod `number`，来得到频率。

### 输入

- generator

    正被估计其乘法阶（周期）的无符号整数。必须是`number`的共质数。

- number

    这个就是原数 $N$。

- useRobustPhaseEstimation

    如果设置为`真`，那么就采用微软自己的函数进行周期估计（以此来计算频率）

- bitsize

    需要表示这个数所用的量子比特位

### 输出

$$\frac {k}{2^{bitsPrecision}}$$
中最接近
$$\frac{s}{r}$$
的分子$k$。

### 代码实现

```Javascript
operation EstimateFrequency(generator : Int, number : Int,useRobustPhaseEstimation : Bool, bitsize : Int): Int {
    mutable frequencyEstimate = 0;
    let bitsPrecision =  2 * bitsize + 1;
    //为周期发现中使用的`黑箱`本征态叠加分配量子位。
    using (eigenstateRegister = Qubit[bitsize]) {
        //将eigenstateRegister初始化为1，这是我们估计其相位的本征态的叠加。
        //我们首先将寄存器定为对无符号整数进行小端编码存取。
        let eigenstateRegisterLE = LittleEndian(eigenstateRegister);
        ApplyXorInPlace(1, eigenstateRegisterLE);
        //使用离散 oracle，以便之后进行相位分析
        let oracle = DiscreteOracle(ApplyOrderFindingOracle(generator, number, _, _));
        //如果打算使用稳定的相位估计
        if (useRobustPhaseEstimation) {
            //使用RobustPhaseEstimation估计s/r.
            //它只需要一个额外的量子位，但是却需要几个oracle
            let phase = RobustPhaseEstimation(bitsPrecision, oracle, eigenstateRegisterLE!);
            //计算k/2^bitsPrecision中最接近s/r的分子k。
            //注意，相位估计投射在随机s对应的本征态上。
            set frequencyEstimate = Round(((phase * IntAsDouble(2 ^ bitsPrecision)) / 2.0) / PI());
        }
        else {
            //使用QuantumPhaseEstimation去估计s/r
            //这里，所需要的代价就是额外`bitsPrecision`的量子位
            using (register = Qubit[bitsPrecision]) {
                let frequencyEstimateNumerator = LittleEndian(register);  
                //包含并元分数k/2^bit精度的分子k的寄存器。
                //分子是一个用big-endian格式编码的无符号整数。
                //使用Microsoft.Quantum.Arithmetic.BigEndian实现。
                QuantumPhaseEstimation(
                    oracle, eigenstateRegisterLE!, LittleEndianAsBigEndian(frequencyEstimateNumerator)
                ); 
                //直接测量并矢分数k/2^bit精度近似s/r的分子k。
                //注意相位估计投射在随机s对应的本征态上。
                set frequencyEstimate = MeasureInteger(frequencyEstimateNumerator);
            }
        }
        // 退回所有用于oracle“黑箱”本征态背，并且将它置为0态
        ResetAll(eigenstateRegister);
    }

    return frequencyEstimate;
}
```

# 经典计算部分

## 函数 MaybeFactorsFromPeriod

通过给定的`period`和`generator`试图找出`number`的因数。

### 输入

- number

    这个就是原数 $N$

- generator

    正在估计其乘法阶（周期）的无符号整数。必须与`number`有共同的质数。

- period

    估计周期（乘法阶）通过 `generator` mod `number` 实现

### 输出

指示是否成功找到因子的标志的元组，以及表示找到的因子的一对整数。请注意，第二个输出只有在第一个输出为`真`时才有意义。

- 可以去看 Microsoft.Quantum.Math.GreatestCommonDivisorI

### 代码实现

```Javascript
function MaybeFactorsFromPeriod(number : Int, generator : Int, period : Int) : (Bool, (Int, Int)) {
    //只有当周期为偶数时，才能归结为因子分解
    if (period % 2 == 0) {
        //计算 `generator` ^ `period/2` mod `number`
        let halfPower = ExpModI(generator, period / 2, number);
        //如果运气不好，半幂就是-1模N，这是一个对因式分解没有用处的例子。
        if (halfPower != number - 1) {
            //当半幂不是-1模N半幂-1或半幂+1与N共同的享非平凡除数。
            let factor = MaxI(
                GreatestCommonDivisorI(halfPower - 1, number),
                GreatestCommonDivisorI(halfPower + 1, number)
            );
            //添加一个我们找到因子的标志，并返回计算出的非平凡因子。
            return (true, (factor, number / factor));
        } else {
            //返回一个标志，表明我们遇到了一个小问题，没有得到任何因子。
            return (false, (1,1));
        }
    } else {
        //当周期为奇数时，我们必须选择另一个'generator'
        //来估计周期并重新启动。
        Message("Estimated period was odd, trying again.");
        return (false, (1,1));
    }
}
```

## 函数 PeriodFromFrequency

从输入频率中求一个数的周期。

### 输入

- number

    `number` 是 $N$，用于 $Z\ mod\ N$，其中`generator`的乘法阶是被估计的。

- frequencyEstimate

    我们要转换成周期的频率。

- bitsPrecision
  
    使用连分式算法估计s/r以恢复周期r所需的精度位数。

- currentDivisor
  
    到目前为止发现的生成周期的除数。

### 输出

通过连分式算法根据估计的频率计算出的周期。

### 代码

```Javascript
function PeriodFromFrequency(number : Int, frequencyEstimate : Int, bitsPrecision : Int, currentDivisor : Int): Int {
    // 现在我们可以用ContinuedFractionConvergentI函数来发现
    // k/2^bitsPrecision中最接近s/r的小数
    let (numerator, period) = (ContinuedFractionConvergentI(Fraction(frequencyEstimate, 2 ^ bitsPrecision), number))!;
    //ContinuedFractionConvergentI不保证分子和分母的符号。
    //这里使用AbsI来保证它们都是正的。
    let (numeratorAbs, periodAbs) = (AbsI(numerator), AbsI(period));
    //使用GreatestCommonDivisorI返回新的除数
    return (periodAbs * currentDivisor) / GreatestCommonDivisorI(currentDivisor, periodAbs);
}
```

# 官方的一些函数简介

## GreatestCommonDivisorI

Microsoft.Quantum.Math.GreatestCommonDivisorI

用于寻找最大公约数

算符表达为 $p=gcd(a,b)$

## ExpModI

Microsoft.Quantum.Math.ExpModI

输入是 $a$, $p$ , $N$

结果为 $Result=a^{p}\ mod\ N$

## AbsI

Microsoft.Quantum.Math.AbsI

整数取绝对值

## ContinuedFractionConvergentI

Microsoft.Quantum.Math.ContinuedFractionConvergentI

查找最接近于 fraction 分母小于或等于的连续分数收敛性 denominatorBound

- 输入
    分式： 分数
    denominatorBound： Int
- 输出： 分数
    最接近的分数， fraction 分母小于或等于 denominatorBound

## Fraction

Microsoft.Quantum.Math.Fraction

表示形式的有理数 $p/q$ 。 整数 $p$ 是元组的第一个元素， $q$ 是元组的第二个元素。

## ResetAll

Microsoft.Quantum.Intrinsic.ResetAll

给定一个 qubits 数组，对其进行度量，确保它们处于 $| 0 ⟩$状态，以便可以安全地释放它们。

## DiscreteOracle

Microsoft.Quantum.Oracles.DiscreteOracle

表示离散 oracle。这是一种 oracle，它实现 $U^{m}U$的固定操作和非负 $m$ 整数。

## RobustPhaseEstimation

Microsoft.Quantum.Characterization.RobustPhaseEstimation

为给定的 oracle 和 eigenstate 执行稳健的非迭代量程阶段估算算法 U ，并提供针对海森堡限制处的变化缩放阶段的单个实际值估算。

## QuantumPhaseEstimation

Microsoft.Quantum.Characterization.QuantumPhaseEstimation

量子相位估计，执行给定 oracle 和的量程阶段估算算法 U ，并将 targetState 该阶段读入大字节序量程寄存器。
