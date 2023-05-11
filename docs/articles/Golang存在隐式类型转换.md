# Golang存在隐式类型转换？

### 前情提要

本着“具体问题，具体分析”的宗旨，简单看下实际场景（之前群里朋友提到的）：

```golang
// ...省略头部
type TmpInt16 int16
// ide提示：const SeqTmpInt int = iota + 1
const SeqTmpInt = iota + 1
func main() {
	varTmpInt16 := int16(1)
	fmt.Println("typeof(varTmpInt16):", reflect.TypeOf(varTmpInt16))
	fmt.Println("typeof(SeqTmpInt):", reflect.TypeOf(SeqTmpInt))
	fmt.Println("equation(varTmpInt16 == SeqTmpInt):", varTmpInt16 == SeqTmpInt)
	// output
	// typeof(varTmpInt16): main.TmpInt16  typeof(SeqTmpInt): int
	// equation(varTmpInt16 == SeqTmpInt): true
}
```

大概问题就是，不同类型之间可以比较。大胆猜测一波，出现了未知的隐式转换。

### 具体分析

首先，我是比较诧异的，`Golang`不是强类型么，不太应该这么花的。首先对于`typeof`这个函数查看变量类型，我还是相信的，`Goland`这边提示也和它是保持一致的。现在可以确定的是变量`varTmpInt16`的类型肯定是`main.TmpInt16`，而`SeqTmpInt`，不太确定，毕竟跟着`iota+1`生成的，那么类型应该是和`iota`保持一致。

跟进去一看`const iota = 0 // Untyped int.`，这个`untyped int`单独提出来再注释强调一下？应该是有东西的，然后顺便看了下其他类型定义，比如`true/false`的常量定义：

```golang
// true and false are the two untyped boolean values.
const (
	true  = 0 == 0 // Untyped bool.
	false = 0 != 0 // Untyped bool.
)
```

无类型布尔？感觉来了，这是一类特殊的类型，那么，照此推断，应该还有无类型字符串之类的。那么问题来了，既然无类型，那么在和有类型比较的时候，是不是就应该有类型转换？当然是能转换过去的那种无类型。那么，什么情况下，才会出现无类型类型呢？就是常量定义，不指定类型。也就是说，**只有常量**，比如：
```
consts (
    // 本身未定义类型
    UntypedInt = 1
    UntypedStr = "string"
    // true是无类型布尔，推导出UntypedTrue也是那样
    UntypedTrue = true
)
```

现在已经确定了无类型类型的存在性，为了再充分一点，就整点实际的：

```golang
// go\types\basic.go

const (
	// types for untyped values
	UntypedBool
	UntypedInt
	UntypedRune
	UntypedFloat
	UntypedComplex
	UntypedString
	UntypedNil
)
```

这里就不敢多看了，会越走越远。总之，没问题，确实存在，那么现在可以肯定了`SeqTmpInt`的类型就是`UntypedInt`，从`iota`常量推导出来的。那么就重点关注常量，比如在编译时的类型检测？百度了一下，也就是`defaultlit2()`，实现不长，简单看看：

```golang
// ./src/cmd/compile/internal/typecheck/const.go

func defaultlit2(l ir.Node, r ir.Node, force bool) (ir.Node, ir.Node) {
	if l.Type() == nil || r.Type() == nil {
		return l, r
	}

	if !l.Type().IsInterface() && !r.Type().IsInterface() {
		if l.Type().IsBoolean() != r.Type().IsBoolean() {
			return l, r
		}
		if l.Type().IsString() != r.Type().IsString() {
			return l, r
		}
	}

	if !l.Type().IsUntyped() {
		r = convlit(r, l.Type())
		return l, r
	}

	if !r.Type().IsUntyped() {
		l = convlit(l, r.Type())
		return l, r
	}

	if !force {
		return l, r
	}

	if ir.IsNil(l) || ir.IsNil(r) {
		return l, r
	}

	t := defaultType(mixUntyped(l.Type(), r.Type()))
	l = convlit(l, t)
	r = convlit(r, t)
}
```

转换细节可以参考`convlit1()`，这里就不展开了。确实会触发隐式转换，应该是无类型的特性。

### 总结一下

`Golang`存在无类型类型，在与具体类型变量比较时（也就是`OEQ ONE OLT OLE OGE OGT`这些操作）和赋值给变量会发生隐式转换。无类型数据来源主要就一种：

* 未指定类型的常量（从无类型常量推导的常量也算在这里面）；

转换规则（基于比较操作来说的）主要有下面这几条：

* 均非接口类型；
    * 布尔不支持与非布尔类型转换；
    * 字符串不支持与非字符串类型转换；
* 无类型支持向有类型转换，如果都是有类型，强制模式下，才可能支持转换；
* 非强制模式，不支持类型向其他类型转换；
* 强制模式下，`Nil`不支持与任何无类型转换；
* 强制模式下，支持最大兼容类型转换，详情参考`mixUntyped`；

终于认识到了，**无类型常量的特性隐式类型转换**。

最后简单提下，**基于无类型常量申明的变量，无类型常量类型会隐式转换为变量的默认类型**（当然类型必须兼容）。

最后的最后，不推荐频繁使用无类型常量，会导致一些不必要的开销。既然强类型，那还是写的明明白白好些。

### 参考文章

* [golang类型推断与隐式类型转换](https://www.jb51.net/article/253419.htm)

* [Go语言中的常量](https://mp.weixin.qq.com/s?__biz=Mzg2NTAyNTc5NQ==&mid=2247484428&idx=1&sn=624b1da4bd46e625fb1076e1387c4760&chksm=ce612e60f916a77625db333c954562888bd27ab3720b096b1fff7938c91a5a9602a18e4a7494&scene=27)