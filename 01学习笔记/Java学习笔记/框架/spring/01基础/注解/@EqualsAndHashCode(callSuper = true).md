`@EqualsAndHashCode(callSuper = true)` 是 **Lombok** 提供的一个注解，用于自动生成 `equals()` 和 `hashCode()` 方法。

## 核心作用

- **`callSuper = true`**：生成的 `equals()` 和 `hashCode()` 方法会**调用父类的实现**
- **`callSuper = false`**（默认）：只考虑当前类的字段，忽略父类继承的字段

## 为什么需要 `callSuper = true`？

### 问题场景（`callSuper = false`）
```java
@EqualsAndHashCode  // 默认 callSuper = false
class Parent {
    int parentId = 1;
}

class Child extends Parent {
    int childId = 1;
}

// 问题：两个不同 parentId 的对象可能相等
Child c1 = new Child();  // parentId=1, childId=1
Child c2 = new Child();  // parentId=2, childId=1 (直接修改)
// 由于只比较 childId，c1.equals(c2) 返回 true ❌
```

### 正确方式（`callSuper = true`）
```java
@EqualsAndHashCode(callSuper = true)
class Child extends Parent {
    int childId = 1;
}

// 正确：会同时比较 parentId 和 childId
c1.equals(c2);  // 返回 false ✅
```

## 使用规则

| 场景 | callSuper 设置 | 原因 |
|------|---------------|------|
| 类**没有继承**（直接继承 Object） | `false`（默认） | 调用 Object 的实现无意义 |
| 类**有继承**，父类有字段 | `true` | 避免漏掉父类字段的比较 |
| 类有继承，但父类无字段或已规范实现 | `false` | 但需谨慎，推荐设为 true |

## 规范建议

1. **继承体系中默认使用 `callSuper = true`**，除非明确知道父类没有需要比较的字段
2. 遵循 **equals 契约**：
   - 对称性：子类 equals 必须考虑父类
   - 传递性：正确处理继承关系
   - 一致性

3. 如果父类没有合理的 equals 实现，考虑**组合优于继承**

## 示例对比

```java
@Data
@EqualsAndHashCode(callSuper = true)  // ✅ 标准做法
public class User extends BaseEntity {
    private String name;
    private String email;
}

// BaseEntity 定义了 id, createTime 等字段
// 这样两个 User 对象只有在 id 和 name 都相等时才被认为相等
```

**总结**：当子类继承父类且父类有需要参与比较的字段时，务必设置 `callSuper = true`，这是避免 bug 的最佳实践。