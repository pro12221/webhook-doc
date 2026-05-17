# Go 值接收者与指针接收者

Go 中方法的接收者可以是值类型（`T`）或指针类型（`*T`），选择哪种直接影响方法能否修改原始数据、接口满足关系和运行性能。本文通过代码示例分析两者的核心区别与选择策略。

## 1. 值接收者与指针接收者的区别

### 1.1 值接收者：操作副本，不影响原值

值接收者在方法调用时会复制整个结构体，方法内部修改的只是副本：

```go
type Square struct {
    side float32
}

func (sq Square) Scale(factor float32) {
    sq.side *= factor // 改的是副本，外面的 sq1 完全不受影响
}
```

调用 `sq1.Scale(2.0)` 后，`sq1.side` 不会改变，因为 `Scale` 接收到的是 `sq1` 的拷贝。

### 1.2 指针接收者：操作原值，修改生效

指针接收者传递的是结构体的地址，方法内部可以直接修改原始数据：

```go
func (sq *Square) Scale(factor float32) {
    sq.side *= factor // 通过指针修改，原值被改变
}
```

## 2. 混用导致的问题

当同一个类型的某些方法用值接收者、某些用指针接收者时，会产生难以预料的行为：

```go
type Shaper interface {
    Area() float32
    scale(factor float32)
}

// Area() 使用指针接收者
func (sq *Square) Area() float32 {
    return sq.side * sq.side
}

// scale() 使用值接收者
func (sq Square) scale(factor float32) {
    sq.side = sq.side * factor // 改副本，原值不变
}
```

这段代码中：
- `Area()` 能正确读取数据，因为指针指向原值
- `scale()` 无法修改原值，因为值接收者操作的是副本
- 接口 `Shaper` 只有 `*Square` 能满足，因为 `Area()` 是指针接收者定义的

## 3. 接口满足规则

| 接口方法全部为值接收者 | 接口包含指针接收者方法 |
|---|---|
| `T` 和 `*T` 都满足接口 | 只有 `*T` 满足接口 |

值接收者更"宽容"——Go 会自动对指针解引用，所以 `*T` 也能调用值接收者方法。但反过来不行：`T` 无法满足要求指针接收者的接口。

## 4. 性能影响

值接收者每次方法调用都会复制整个结构体：

```go
type BigImage struct {
    data [10 * 1024 * 1024]byte // 10MB
}

func (img BigImage) Render() { ... } // 每次调用复制 10MB
```

大结构体用值接收者，每次方法调用产生一次完整内存拷贝，性能和内存都不可接受。

## 5. 选择决策

| 需求 | 接收者类型 |
|---|---|
| 需要修改接收者 | 指针 |
| 结构体很大，避免拷贝 | 指针 |
| 需要表示"可变状态" | 指针 |
| 小值类型、只读、不可变 | 值 |

### 一致性原则

Go 官方建议：**如果一个类型有任何一个方法需要指针接收者，那所有方法都应该用指针接收者。** 混用会导致混乱——这个方法能改原值，那个改不了，调用者很难记住。

## 6. 类型断言与接口

通过类型断言可以检查接口变量的实际类型：

```go
var areaIntf Shaper
sq1 := new(Square)
sq1.side = 5
areaIntf = sq1

// 类型断言
if t, ok := areaIntf.(*Square); ok {
    fmt.Printf("The type of areaIntf is: %T\n", t)
}
if u, ok := areaIntf.(*Circle); ok {
    fmt.Printf("The type of areaIntf is: %T\n", u)
} else {
    fmt.Println("areaIntf does not contain a variable of type Circle")
}
```

注意：如果 `Shaper` 接口的方法都是值接收者定义的，`Square` 和 `*Square` 都能满足接口，此时 `areaIntf.(*Square)` 和 `areaIntf.(Square)` 都可能成功（取决于赋值时传入的是值还是指针）。

## 7. 总结

- **值接收者**：安全、不可变，但无法修改原值，大结构体有拷贝开销
- **指针接收者**：可修改原值、避免拷贝，但接口只有 `*T` 能满足
- **核心原则**：需要修改 → 用指针；结构体大 → 用指针；只读小值 → 可用值；一旦有指针接收者 → 全部用指针
