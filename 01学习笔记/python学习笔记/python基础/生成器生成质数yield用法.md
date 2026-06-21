这个问题选得好！用生成器生成质数，既能练习`yield`的用法，又能用到经典的算法思维。我给你写一个**从基础到优化**的完整demo，顺便把我在`yield`里挖的坑给你填平了。(deepseek生成)

---
[[yield关键字]]
## 方案一：最直观的实现（适合理解生成器）

```python
def prime_generator(limit):
    """生成所有小于等于limit的质数"""
    for num in range(2, limit + 1):  # 从2开始
        is_prime = True
        # 检查num是否能被2到sqrt(num)之间的数整除
        for i in range(2, int(num ** 0.5) + 1):
            if num % i == 0:
                is_prime = False
                break
        if is_prime:
            yield num  # 是质数就yield出去

# 使用：生成100以内的所有质数
for prime in prime_generator(100):
    print(prime, end=" ")
# 输出：2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97
```

**执行流程（对照Java）**：
1. `prime_generator(100)` 被调用时，**不会执行任何代码**，只返回一个生成器对象。
2. 进入 `for` 循环时，才开始执行：
   - `num=2`：内层循环不执行（range(2,2)为空）→ `is_prime=True` → `yield 2` → **暂停**。
   - 下次迭代：从 `yield` 后面继续 → `num=3` → 检查 → `yield 3` → **暂停**。
   - 直到 `num=100` 结束。

**这个版本的缺点**：每次判断质数都要从2试除到sqrt(n)，效率低但容易理解。

---

## 方案二：优化版——埃拉托斯特尼筛法（性能最好）

```python
def prime_sieve_generator(limit):
    """用埃氏筛生成质数（一次性筛出所有合数）"""
    is_prime = [True] * (limit + 1)  # 先假设全是质数
    is_prime[0] = is_prime[1] = False  # 0和1不是质数
    
    for num in range(2, int(limit ** 0.5) + 1):
        if is_prime[num]:  # 如果num是质数
            # 把num的所有倍数标记为合数
            for multiple in range(num * num, limit + 1, num):
                is_prime[multiple] = False
    
    # 遍历所有数，yield出质数
    for num in range(2, limit + 1):
        if is_prime[num]:
            yield num

# 使用
for prime in prime_sieve_generator(100):
    print(prime, end=" ")
# 同样输出所有质数，但计算速度快了10倍以上
```

**原理图解（用Java思维理解）**：
- 这就像你有一张100个座位的表，全标记为"可用"。
- 从2开始：2是质数，把4、6、8、10...全部标记为"已占用"。
- 跳到3：3是质数，把9、12、15...标记为"已占用"。
- 4已经被标记了，跳过。
- 5没被标记，是质数，标记25、30、35...
- 最后没被标记的数就是质数。

**为什么快**：不用对每个数做除法，用"排除法"批量干掉合数。

---

## 方案三：无限生成器（生产质数到天荒地老）

```python
def infinite_prime_generator():
    """无限生成质数（永远不停）"""
    yield 2  # 先把2特殊处理
    primes = [2]  # 存已经找到的质数
    num = 3
    
    while True:
        is_prime = True
        # 只用已找到的质数去试除
        for p in primes:
            if p * p > num:  # 优化：超过sqrt就不用试了
                break
            if num % p == 0:
                is_prime = False
                break
        
        if is_prime:
            primes.append(num)
            yield num  # 无限yield，外部控制何时停止
        
        num += 2  # 只检查奇数

# 使用：取前25个质数
gen = infinite_prime_generator()
primes_100 = []
for _ in range(25):  # 控制取多少个
    primes_100.append(next(gen))
print(primes_100[-1])  # 97（第25个质数）

# 或者：取所有小于100的质数
gen = infinite_prime_generator()
result = []
while True:
    p = next(gen)
    if p >= 100:
        break
    result.append(p)
print(result)  # [2, 3, 5, ..., 97]
```

**这个方案的妙处**：
- 不需要预先知道`limit`，想停就停。
- 只保存已找到的质数（很小的内存），不用保存所有数字的状态。
- 类似于Java里用`Iterator`无限生成。

---

## 方案四：最Pythonic的写法（生成器表达式 + filter）

```python
def is_prime(n):
    """判断单个数字是否是质数"""
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return False
    return True

# 生成器表达式 + filter
prime_gen = filter(is_prime, range(2, 101))
print(list(prime_gen))  # 直接用list()消费所有生成的值

# 或者更简洁的写法
prime_gen = (x for x in range(2, 101) if all(x % i != 0 for i in range(2, int(x ** 0.5) + 1)))
print(list(prime_gen))
```

**Java对照**：相当于 `IntStream.range(2, 101).filter(this::isPrime).boxed().collect(Collectors.toList())`，但Python的生成器表达式更轻量。

---

## 🎯 实战：把生成器用到你的AI学习场景

假设你要生成**模拟的token ID序列**来做测试：

```python
def token_stream_generator(max_tokens=1000, vocab_size=50000):
    """生成模拟的token流，用于测试推理速度"""
    import random
    for _ in range(max_tokens):
        yield random.randint(0, vocab_size - 1)

# 模拟流式推理
for token_id in token_stream_generator(10):
    print(f"生成token: {token_id}")
    # 这里可以模拟逐个token的生成和输出
```

---

## 💡 从Java视角理解生成器的内存优势

```java
// Java：先算好所有质数，存进List
List<Integer> primes = new ArrayList<>();
for (int i = 2; i <= 100; i++) {
    if (isPrime(i)) primes.add(i);
}
// 内存占用：25个int ≈ 100字节（实际ArrayList有额外开销）

// Python生成器：不存任何东西
def prime_gen(limit):
    for i in range(2, limit + 1):
        if is_prime(i):
            yield i
gen = prime_gen(100)  # 内存占用：几乎为0
// 每次next()才计算一个，算完就丢弃
```

**关键区别**：
- Java的`List`：**先全算好放内存**，你什么时候遍历都行（可重复）。
- Python的生成器：**边算边给**，算完就扔（一次性的，像水流）。

---

## 🧪 给你的小测试（验证你懂了没）

运行下面这段代码，写出输出顺序：

```python
def test_generator():
    print("1. 启动生成器")
    yield 2
    print("2. 返回2后继续")
    yield 3
    print("3. 返回3后继续")
    yield 5
    print("4. 全部完成")

gen = test_generator()
print("A. 创建生成器")
print("B. 第一次next:", next(gen))
print("C. 第二次next:", next(gen))
print("D. 第三次next:", next(gen))
```

**预期输出**（先别往下翻，自己脑跑）：
```
A. 创建生成器
B. 第一次next: 1. 启动生成器
2
C. 第二次next: 2. 返回2后继续
3
D. 第三次next: 3. 返回3后继续
5
```

如果输出顺序完全对上号了，说明你对生成器的执行流程已经彻底通透！

---

## 📌 给你的实战建议

1. **日常用方案一**（直观好理解），**性能用方案二**（筛法最快）。
2. **无限生成器**在AI里很少用，但能帮你深刻理解`yield`的暂停机制。
3. **把 `range(2, 101)` 改成 `range(2, 1000001)`**，感受一下生成器的内存优势（列表会爆内存，生成器完全没事）。
4. **组合使用**：`sum(prime_generator(1000))` 直接求和，不用显式for循环。

---

现在你既会用`yield`，又会算质数了。接下来想挑战什么？**`yield from`**（把多个生成器串联）？还是**协程（`async/await`）**？或者是**用生成器实现一个简单的数据流水线（类似PyTorch的DataLoader）**？告诉我，我继续给你拆解！