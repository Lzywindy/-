**<center><font face="微软雅黑" size=10>Shor算法学习</font></center>**

# 代码简介

此示例包含实现Shor整数因式分解的量子算法的Q#代码。基于Stephane Beauregard的一篇论文，基本的模块化算法是在相位编码中实现的，他给出了一个量子电路需要 $2n+3$ 个量子位和$O(n^{3}log(n))$的基本量子门来进行对$n$比特数的因式分解。

# 主函数  FactorSemiprimeInteger

## 概要

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

## 概要

将`target`转化为无符号小小端存储的整数 $k$ 进行编码，并执行转换 $|k⟩↦| g^{p}k\mod\ N⟩$ ，其中$p$是“power”，$g$是“generator”，而$N$是“module”。

### 输入

- generator

    The unsigned integer multiplicative order ( period )of which is being estimated. Must be co-prime to `modulus`.

- modulus

    The modulus which defines the residue ring Z mod `modulus` in which the multiplicative order of `generator` is being estimated.

- power

    Power of `generator` by which `target` is multiplied.

- target

    Register interpreted as LittleEndian which is multiplied by given power of the generator. The multiplication is performed modulo
`modulus`.

```Javascript
operation ApplyOrderFindingOracle(generator : Int, modulus : Int, power : Int, target : Qubit[]): Unit is Adj + Ctl {
     Check that the parameters satisfy the requirements.
    Fact(IsCoprimeI(generator, modulus), "`generator` and `modulus` must be co-prime");

     The oracle we use for order finding essentially wraps
     Microsoft.Quantum.Arithmetic.MultiplyByModularInteger operation
     that implements |x⟩ ↦ |x⋅a mod N ⟩.
     We also use Microsoft.Quantum.Math.ExpModI to compute a by which
     x must be multiplied.
     Also note that we interpret target as unsigned integer
     in little-endian encoding by using Microsoft.Quantum.Arithmetic.LittleEndian
     type.
    MultiplyByModularInteger(ExpModI(generator, power, modulus), modulus, LittleEndian(target));
}
```

/ # Summary
/ Finds a multiplicative order of the generator
/ in the residue ring Z mod `modulus`.
/
/ # Input
/ ## generator
/ The unsigned integer multiplicative order ( period )
/ of which is being estimated. Must be co-prime to `modulus`.
/ ## modulus
/ The modulus which defines the residue ring Z mod `modulus`
/ in which the multiplicative order of `generator` is being estimated.
/ ## useRobustPhaseEstimation
/ If set to true, we use Microsoft.Quantum.Characterization.RobustPhaseEstimation and
/ Microsoft.Quantum.Characterization.QuantumPhaseEstimation
/
/ # Output
/ The period ( multiplicative order ) of the generator mod `modulus`

```Javascript
operation EstimatePeriod(generator : Int, modulus : Int, useRobustPhaseEstimation : Bool) : Int {
     Here we check that the inputs to the EstimatePeriod operation are valid.
    Fact(IsCoprimeI(generator, modulus), "`generator` and `modulus` must be co-prime");

     The variable that stores the divisor of the generator period found so far.
    mutable result = 1;

     Number of bits in the modulus with respect to which we are estimating the period.
    let bitsize = BitSizeI(modulus);

     The EstimatePeriod operation estimates the period r by finding an
     approximation k/2^(bits precision) to a fraction s/r, where s is some integer.
     Note that if s and r have common divisors we will end up recovering a divisor of r
     and not r itself. However, if we recover enough divisors of r
     we recover r itself pretty soon.

     Number of bits of precision with which we need to estimate s/r to recover period r.
     using continued fractions algorithm.
    let bitsPrecision = 2 * bitsize + 1;

     A variable that stores our current estimate for the frequency
     of the form s/r. 
    mutable frequencyEstimate = 0;

    repeat {

        set frequencyEstimate = EstimateFrequency(
            generator, modulus, useRobustPhaseEstimation, bitsize 
        );

        if (frequencyEstimate != 0) {
            set result = PeriodFromFrequency(modulus,frequencyEstimate, bitsPrecision, result);
        }
        else {
            Message("The estimated frequency was 0, trying again.");
        }
    }
    until(ExpModI(generator, result, modulus) == 1)
    fixup {
        Message("The estimated period from continued fractions failed, trying again.");
    }

    return result;
}
```

/ # Summary
/ Estimates the frequency of a generator
/ in the residue ring Z mod `modulus`.
/
/ # Input
/ ## generator
/ The unsigned integer multiplicative order ( period )
/ of which is being estimated. Must be co-prime to `modulus`.
/ ## modulus
/ The modulus which defines the residue ring Z mod `modulus`
/ in which the multiplicative order of `generator` is being estimated.
/ ## useRobustPhaseEstimation
/ If set to true, we use Microsoft.Quantum.Characterization.RobustPhaseEstimation else
/ this operation uses Microsoft.Quantum.Characterization.QuantumPhaseEstimation
/ ## bitsize
/ Number of bits needed to represent the modulus.
/
/ # Output
/ The numerator k of dyadic fraction k/2^bitsPrecision
/ approximating s/r.

```Javascript
operation EstimateFrequency(generator : Int, modulus : Int,useRobustPhaseEstimation : Bool, bitsize : Int): Int {
    mutable frequencyEstimate = 0;
    let bitsPrecision =  2 * bitsize + 1;
    
     Allocate qubits for the superposition of eigenstates of
     the oracle that is used in period finding.
    using (eigenstateRegister = Qubit[bitsize]) {

         Initialize eigenstateRegister to 1, which is a superposition of
         the eigenstates we are estimating the phases of.
         We first interpret the register as encoding an unsigned integer
         in little endian encoding.
        let eigenstateRegisterLE = LittleEndian(eigenstateRegister);
        ApplyXorInPlace(1, eigenstateRegisterLE);

         An oracle of type Microsoft.Quantum.Oracles.DiscreteOracle
         that we are going to use with phase estimation methods below.
        let oracle = DiscreteOracle(ApplyOrderFindingOracle(generator, modulus, _, _));

        if (useRobustPhaseEstimation) {

             Use Microsoft.Quantum.Characterization.RobustPhaseEstimation to estimate s/r.
             RobustPhaseEstimation needs only one extra qubit, but requires
             several calls to the oracle.
            let phase = RobustPhaseEstimation(bitsPrecision, oracle, eigenstateRegisterLE!);
            
             Compute the numerator k of dyadic fraction k/2^bitsPrecision
             approximating s/r. Note that phase estimation projects on the eigenstate
             corresponding to random s.
            set frequencyEstimate = Round(((phase * IntAsDouble(2 ^ bitsPrecision)) / 2.0) / PI());
        }
        else {
             Use Microsoft.Quantum.Characterization.QuantumPhaseEstimation to estimate s/r.
             When using QuantumPhaseEstimation we will need extra `bitsPrecision`
             qubits
            using (register = Qubit[bitsPrecision]) {
                let frequencyEstimateNumerator = LittleEndian(register);  

                 The register that will contain the numerator k of
                 dyadic fraction k/2^bitsPrecision. The numerator is an unsigned
                 integer encoded in big-endian format. This is indicated by
                 use of Microsoft.Quantum.Arithmetic.BigEndian type.
                QuantumPhaseEstimation(
                    oracle, eigenstateRegisterLE!, LittleEndianAsBigEndian(frequencyEstimateNumerator)
                ); 
                
                 Directly measure the numerator k of dyadic fraction k/2^bitsPrecision
                 approximating s/r. Note that phase estimation project on
                 the eigenstate corresponding to random s.
                set frequencyEstimate = MeasureInteger(frequencyEstimateNumerator);
            }
        }
        
         Return all the qubits used for oracle's eigenstate back to 0 state
         using Microsoft.Quantum.Intrinsic.ResetAll.
        ResetAll(eigenstateRegister);
    }

    return frequencyEstimate;
}
```

/ # Summary
/ Find the period of a number from an input frequency.
/
/ # Input
/ ## modulus
/ The modulus which defines the residue ring Z mod `modulus`
/ in which the multiplicative order of `generator` is being estimated.
/ ## frequencyEstimate
/ The frequency that we want to convert to a period. 
/ ## bitsPrecision
/ Number of bits of precision with which we need to 
/ estimate s/r to recover period r using continued 
/ fractions algorithm.
/ ## currentDivisor
/ The divisor of the generator period found so far.
/
/ # Output
/ The period as calculated from the estimated frequency via
/ the continued fractions algorithm.
/
/ # See Also
/ - Microsoft.Quantum.Math.ContinuedFractionConvergentI
```
function PeriodFromFrequency(modulus : Int, frequencyEstimate : Int, bitsPrecision : Int, currentDivisor : Int): Int {
    
     Now we use Microsoft.Quantum.Math.ContinuedFractionConvergentI
     function to recover s/r from dyadic fraction k/2^bitsPrecision.
    let (numerator, period) = (ContinuedFractionConvergentI(Fraction(frequencyEstimate, 2 ^ bitsPrecision), modulus))!;
    
     ContinuedFractionConvergentI does not guarantee the signs of the numerator
     and denominator. Here we make sure that both are positive using
     AbsI.
    let (numeratorAbs, periodAbs) = (AbsI(numerator), AbsI(period));

     Return the newly found divisor.
     Uses Microsoft.Quantum.Math.GreatestCommonDivisorI function from Microsoft.Quantum.Math.
    return (periodAbs * currentDivisor) / GreatestCommonDivisorI(currentDivisor, periodAbs);
}
```

## 函数 MaybeFactorsFromPeriod

Tries to find the factors of `modulus` given a `period` and `1generator`.

### 输入

- modulus

    The modulus which defines the residue ring Z mod `modulus` in which the multiplicative order of `generator` is being estimated.

- generator

    The unsigned integer multiplicative order ( period ) of which is being estimated. Must be co-prime to `modulus`.

- period

    The estimated period ( multiplicative order ) of the generator mod `modulus`.

### 输出

A tuple of a flag indicating whether factors were found successfully,
and a pair of integers representing the factors that were found.
Note that the second output is only meaningful when the first
output is `true`.

- 可以参照

    Microsoft.Quantum.Math.GreatestCommonDivisorI

### 代码实现

```Javascript
function MaybeFactorsFromPeriod(modulus : Int, generator : Int, period : Int) 
: (Bool, (Int, Int)) {
     Period finding reduces to factoring only if period is even
    if (period % 2 == 0) {
         Compute `generator` ^ `period/2` mod $N$
         using Microsoft.Quantum.Math.ExpModI.
        let halfPower = ExpModI(generator, period / 2, modulus);

         If we are unlucky, halfPower is just -1 mod N,
         which is a trivial case and not useful for factoring.
        if (halfPower != modulus - 1) {

             When the halfPower is not -1 mod N
             halfPower-1 or halfPower+1 share non-trivial divisor with $N$.
             We find a divisor Microsoft.Quantum.Math.GreatestCommonDivisorI.
            let factor = MaxI(
                GreatestCommonDivisorI(halfPower - 1, modulus), 
                GreatestCommonDivisorI(halfPower + 1, modulus)
            );
            
             Add a flag that we found the factors, and return computed non-trivial factors.
            return (true, (factor, modulus / factor));
        } else {
             Return a flag indicating we hit a trivial case and didn't get any factors.
            return (false, (1,1));
        }
    } else {
         When period is odd we have to pick another generator to estimate
         period of and start over.
        Message("Estimated period was odd, trying again.");
        return (false, (1,1));
    }
}
```