# flatbuffers 包 API 参考

FlatBuffers 的 Cangjie 运行时库。通常不直接手工序列化：先用 `flatc --cangjie` 从 `.fbs` schema 生成访问器代码，再配合本包的 `Builder`（写入）和生成的类型（读写）完成编解码。

```
import flatbuffers.*
```

**典型用法**

```cangjie
import flatbuffers.*

// 本示例基于下方极简 schema，已用 flatc --cangjie 生成 Sample_Weapon / Sample_Monster。
// 生成代码的类名带命名空间前缀，可先用 type 别名简化（也可直接使用原名）；
// 生成代码与用户代码需在同一包内（生成的 init 与 getter 默认不带 public）。
type Monster = Sample_Monster
type Weapon = Sample_Weapon

main(): Int64 {
    let builder = Builder(0)

    // 1. 先构造被引用的字符串与子表（深度优先）
    let weaponName = builder.createString("sword")
    let weapon = Weapon.createWeapon(builder, weaponName, 3)
    let weapons = builder.createVectorOfTables([weapon])
    let monsterName = builder.createString("ORC")

    // 2. 用生成的 create 函数一次性构造表（全部为位置参数，按 schema 字段顺序传入）
    let monster = Monster.createMonster(builder, monsterName, weapons)

    // 3. 收尾并取回缓冲区
    builder.finish(monster)
    let bytes = builder.finishedBytes()

    // 4. 读取：使用生成的类型（getter 以 Get 开头；返回 Option 的字段用 match 解包）
    let m = Monster(bytes, 0)
    println("name = ${m.GetName()}")
    for (w in m.GetWeapons()) {
        match (w) {
            case Some(weapon) => println("weapon damage = ${weapon.GetDamage()}")
            case None => println("weapon missing")
        }
    }
    return 0
}
```

运行输出：

```text
name = ORC
weapon damage = 3
```

> **约定**：写入必须遵循"深度优先、先构造被引用对象"的顺序——字符串、向量、子表必须在其所属表 `create`/`finish` 之前完成构造。

**示例对应的 schema**

保存为 `mini.fbs`，执行 `flatc --cangjie mini.fbs` 即可生成示例中用到的类型（生成类名带命名空间前缀，即上方示例中的 `Sample_Weapon` / `Sample_Monster`）：

```fbs
namespace Sample;

table Weapon {
  name:string;
  damage:short;
}

table Monster {
  name:string;
  weapons:[Weapon];
}

root_type Monster;
```

# class Builder

序列化缓冲区的管理器。采用"从后往前"写入（prepend）模型，最终 `finishedBytes()` 返回已使用部分。

```cangjie
public class Builder
```

## init

```cangjie
public init(initialSize: Int64)
```

**功能：** 创建 Builder，`initialSize` 为预分配缓冲区字节数。

**参数：**

- `initialSize`: 预分配的初始容量，小于等于 0 时不预分配（内部按需扩容）。

## func finish

```cangjie
public func finish(rootTable: UInt32): Unit
```

**功能：** 结束序列化，将根表偏移写入缓冲区头部。调用后 Builder 进入 finished 状态，不可再写入。

**参数：**

- `rootTable`: 根表的偏移（由生成的 `createXxx` 返回）。

> **注意**：写入类接口在 `finish` 之后调用时行为未定义，请勿复用 finished 状态的 Builder 写入；如需复用请调用 `reset()`。

## func finishWithFileIdentifier

```cangjie
public func finishWithFileIdentifier(rootTable: UInt32, fid: Array<UInt8>): Unit
```

**功能：** 同 `finish`，并在根表偏移之后（即缓冲区起始部分的字节 4–8）写入 4 字节文件标识（file identifier），用于接收方校验 schema 类型。

**参数：**

- `fid`: 4 字节文件标识，例如 `"MONS".toArray()`。长度不是 4 字节时抛出 `FlatBuffersException`。

## func finishSizePrefixed

```cangjie
public func finishSizePrefixed(rootTable: UInt32): Unit
```

**功能：** 同 `finish`，并在缓冲区起始处额外写入 4 字节总长度，适用于流式传输/共享内存场景。

## func finishSizePrefixedWithFileIdentifier

```cangjie
public func finishSizePrefixedWithFileIdentifier(rootTable: UInt32, fid: Array<UInt8>): Unit
```

**功能：** `finishSizePrefixed` 与 `finishWithFileIdentifier` 的组合。

## func finishedBytes

```cangjie
public func finishedBytes(): Array<UInt8>
```

**功能：** 返回已使用部分的字节切片（从当前写入头到缓冲区末尾），可直接用于传输或传给生成的读取类型。

**返回值：** 序列化后的字节数组。

## func reset

```cangjie
public func reset(): Unit
```

**功能：** 重置 Builder 到初始状态（保留已分配的缓冲区），可复用同一 Builder 序列化多个对象。

## func createString

```cangjie
public func createString(s: String): UInt32
```

**功能：** 向缓冲区写入一个 UTF-8 字符串（含结尾 `'\0'`），返回其偏移。

**参数：**

- `s`: 待写入的字符串。

**返回值：** 字符串在缓冲区中的偏移，作为生成代码中 `addXxx`/`createXxx` 的参数。

## func createByteString

```cangjie
public func createByteString(s: Array<UInt8>): UInt32
```

**功能：** 同 `createString`，但输入为原始字节（不校验 UTF-8），写入后追加 `'\0'`。

## func createSharedString

```cangjie
public func createSharedString(s: String): UInt32
```

**功能：** 同 `createString`，但对相同内容只写入一次并复用偏移，适合重复字符串较多的场景。

## func createByteVector

```cangjie
public func createByteVector(v: Array<UInt8>): UInt32
```

**功能：** 写入一个 `[ubyte]` 向量（4 字节长度 + 元素），返回偏移。

## func createInt32Vector / createUInt32Vector / createUInt64Vector

```cangjie
public func createInt32Vector(v: Array<Int32>): UInt32
public func createUInt32Vector(v: Array<UInt32>): UInt32
public func createUInt64Vector(v: Array<UInt64>): UInt32
```

**功能：** 写入对应标量类型的向量（4 字节长度 + 按对齐要求排列的元素），返回偏移。

## func createVectorOfTables

```cangjie
public func createVectorOfTables(offsets: Array<UInt32>): UInt32
```

**功能：** 将一组已构造好的表偏移写成一个表向量，返回向量偏移。

**参数：**

- `offsets`: 各子表偏移（由生成的 `createXxx` 返回），向量内元素顺序与数组顺序一致。

# class FlatBufferObject

所有生成类型（`Monster` 等）的公共基类，负责从缓冲区定位根表。

```cangjie
public open class FlatBufferObject
```

## init

```cangjie
public init(buf: Array<UInt8>, root: UInt32)
```

**功能：** 读取缓冲区 `root` 位置的根表偏移并定位表。

**参数：**

- `buf`: 完整的 FlatBuffer 字节缓冲区。
- `root`: 根偏移，通常为 0。

## 成员变量

- `let table: Table` —— 定位后的表，供生成代码读取字段。

# class Table

对缓冲区中某个表的低层读取视图。生成代码的字段 getter 基于它实现；手工解析（不依赖生成代码）时也可以直接使用。

```cangjie
public class Table
```

## init

```cangjie
public init(buf: Array<UInt8>, position: UInt32)
```

**功能：** 创建指向缓冲区 `position` 处表的读取视图。

**参数：**

- `buf`: 完整缓冲区。
- `position`: 表的起始偏移，指向表的 soffset 字段本身（vtable 位置 = `position` − soffset 值，字段偏移均相对 `position` 计算）。

## 成员变量

- `public var bytes: Array<UInt8>` —— 完整的 FlatBuffer 字节缓冲区。
- `public var pos: UInt32` —— 当前表的起始偏移。

## func offset

```cangjie
public func offset(slot: UInt16): UInt16
```

**功能：** 查询 vtable 中槽位号为 `slot` 的字段偏移。字段未写入（或超出 vtable 范围）时返回 0。

**参数：**

- `slot`: 字段在 vtable 中的槽位号（由 schema 决定，生成代码中固定）。

## func getIndirect

```cangjie
public func getIndirect(offsetVal: UInt32): UInt32
```

**功能：** 解引用表中常见的 "偏移的偏移" 结构，返回目标对象实际位置：`offsetVal + *(uoffset*)offsetVal`。

## func getString

```cangjie
public func getString(offsetVal: UInt32): String
```

**功能：** 读取 `offsetVal` 处字符串（不含结尾 `'\0'`）并解码为 UTF-8 `String`。

## func getByteVector

```cangjie
public func getByteVector(offsetVal: UInt32): Array<UInt8>
```

**功能：** 读取 `offsetVal` 处 `[ubyte]` 向量的全部元素。

## func getVectorLenBySlot / getVectorStartBySlot

```cangjie
public func getVectorLenBySlot(slot: UInt16): Int64
public func getVectorStartBySlot(slot: UInt16): UInt32
```

**功能：** 按 vtable 槽位读取向量的长度 / 元素区起始偏移。字段不存在时长度返回 0、起始返回 0。

## func get{Type}

```cangjie
public func getBool(offsetVal: UInt32): Bool
public func getInt8(offsetVal: UInt32): Int8
public func getUInt8(offsetVal: UInt32): UInt8
public func getInt16(offsetVal: UInt32): Int16
public func getUInt16(offsetVal: UInt32): UInt16
public func getInt32(offsetVal: UInt32): Int32
public func getUInt32(offsetVal: UInt32): UInt32
public func getInt64(offsetVal: UInt32): Int64
public func getUInt64(offsetVal: UInt32): UInt64
public func getFloat32(offsetVal: UInt32): Float32
public func getFloat64(offsetVal: UInt32): Float64
```

**功能：** 从缓冲区 `offsetVal` 绝对偏移处读取对应标量。通常与 `offset(slot)` 配合使用：先查询字段偏移 `o`，字段存在时字段值位于 `table.pos + UInt32(o)`，再用对应 `get{Type}` 读取；`o` 为 0 表示字段未写入。

## func get{Type}Slot

```cangjie
public func getInt32Slot(slot: UInt16, default: Int32): Int32
// Bool / Int8 / UInt8 / Int16 / UInt16 / Int32 / UInt32 / Int64 / UInt64 / Float32 / Float64 各有一组
```

**功能：** 按 vtable 槽位读取标量字段，字段未写入时返回 `default`。这是读取可选字段最常用的接口。

## func getUnion

```cangjie
public func getUnion(offsetVal: UInt32): FlatBufferObject
```

**功能：** 读取 `offsetVal` 处 union 成员，返回 `FlatBufferObject`；再按 union type 字段转型为具体生成类型。

# 异常

```cangjie
public class FlatBuffersException <: Exception
```

**功能：** 序列化过程中的错误检查异常，可按类型捕获。

**典型触发场景：**

- `finishedBytes()` 在 `finish` 之前调用
- 文件标识 `fid` 长度不是 4 字节
- 向量/表的创建顺序错误（嵌套构造、缺少 finish）

**示例：**

```cangjie
try {
    let b = Builder(0)
    // 文件标识必须是 4 字节，传入 3 字节触发异常
    b.finishWithFileIdentifier(0, "MON".toArray())
} catch (e: FlatBuffersException) {
    println("serialization error: ${e}")
}
```

# 仅供生成代码使用的接口

以下 `Builder`/`Table` 接口语义依赖写入顺序等内部约定，仅应由 `flatc --cangjie` 生成的代码调用，用户代码请勿直接使用：

- `startObject` / `endObject` —— 开始/结束一个表（生成 vtable 并做去重复用）
- `startVector` / `endVector` —— 开始/结束一个向量
- `prepend{Type}` / `prepend{Type}Slot` / `prependUOffsetT` / `prependUOffsetTSlot` / `prependStructSlot` —— 倒序写入标量与偏移
- `prep` / `pad` / `offset` —— 对齐与位置管理

如确需手工序列化，请先阅读 `docs/source/internals.md` 了解 FlatBuffers 的 wire format 与写入顺序约束。
