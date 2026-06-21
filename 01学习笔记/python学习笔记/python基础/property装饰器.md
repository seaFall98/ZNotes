Python `property` 装饰器 完整用法

# Python `property` 装饰器 完整用法

## 一、作用

`@property` 是**内置装饰器** 作用：

1. **把类中的方法 伪装成 属性**
    
2. 调用时**不加括号**，像访问变量一样简洁
    
3. 规范封装：**私有属性取值、赋值、删除**，控制访问权限
    
4. 替代 Java 的 `getter / setter`
    

---

## 二、基础用法：只读属性（getter）

### 场景：不让外部直接改，只能读取

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.__age = age  # 私有属性

    # 把方法变成属性
    @property
    def age(self):
        # 可以加校验逻辑
        if self.__age < 0:
            return 0
        return self.__age

p = Person("小明", 20)
# 像访问属性一样调用，不用加()
print(p.age)
# p.age = 30  报错：只读不能赋值
```

- 加了 `@property` → 此方法**只能读，不能改**
    

---

## 三、设置赋值方法：@属性名.setter（setter）

语法固定：

```python
@property          # 读
def 名字(self):...

@名字.setter       # 写
def 名字(self,值):...
```

### 完整读写示例

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.__age = age

    # 读取
    @property
    def age(self):
        return self.__age

    # 赋值
    @age.setter
    def age(self, value):
        # 写入前做数据校验
        if not isinstance(value, int):
            print("年龄必须是整数")
            return
        if value < 0 or value > 150:
            print("年龄不合法")
            return
        self.__age = value


p = Person("小红", 18)
print(p.age)       # 读属性
p.age = 25         # 直接赋值，自动走setter
print(p.age)

p.age = -10        # 触发校验，赋值失败
```

### 执行逻辑

1. `p.age` → 执行 `@property` 方法取值
    
2. `p.age = 数值` → 执行 `@age.setter` 方法赋值
    

---

## 四、删除属性：@属性名.deleter

控制 `del 对象.属性` 行为

```python
class Person:
    def __init__(self, age):
        self.__age = age

    @property
    def age(self):
        return self.__age

    @age.setter
    def age(self, v):
        self.__age = v

    @age.deleter
    def age(self):
        print("正在删除年龄属性")
        del self.__age

p = Person(20)
del p.age
```

---

## 五、经典实战：封装身高体重，自动算 BMI

```python
class Student:
    def __init__(self, height, weight):
        self.height = height
        self.weight = weight

    # 只读属性：自动计算BMI
    @property
    def bmi(self):
        return self.weight / (self.height ** 2)


s = Student(1.75, 65)
# 直接当成属性访问，不用调用方法
print(s.bmi)
```

优势：

- BMI 是**计算出来的**，不存变量
    
- 外界直接 `s.bmi` 读取，简洁优雅
    

---

## 六、三种写法总结

1. **只读**：只写 `@property`
    
2. **读写**：`@property` + `@xxx.setter`
    
3. **读写删**：再加 `@xxx.deleter`
    

---

## 七、和 Java 对比（你很好理解）

- Python `@property` = Java `getter`
    
- Python `@xxx.setter` = Java `setter` 目的统一： **私有化成员变量，对外提供统一访问入口，加入数据校验**
    

---

## 八、易错点

1. setter 方法**方法名必须和 property 一模一样**
    
2. 顺序不能乱：**先写 @property，再写 setter**
    
3. 私有变量习惯用 `__变量名` 双下划线
    

需要我给你出一道练习题巩固吗？