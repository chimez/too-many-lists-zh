# Variance and PhantomData

"现在先放一边，以后再修" 很烦人，所以我们现在就做一些硬核的布局。

在制造 unsafe Rust 数据结构时，有五个可怕的骑士:

1. [Variance](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
2. [Drop Check](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
3. [NonNull Optimizations](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
4. [The isize::MAX Allocation Rule](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
5. [Zero-Sized Types](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

庆幸的是，最后两个问题现在对我们来说不是问题。

第三个问题我们*可以*把它变成我们的问题，但是它带来的麻烦比它的价值更多 -- 如果你选择了 LinkedList，你就已经在内存效率上放弃了100倍的战斗。

第二种是我曾经坚持认为非常重要的东西，并且 std 也会乱来，但是默认是安全的，乱来的方法是不稳定的，而且你需要 *非常努力* 才能注意到默认配置的限制，所以，不要担心。

这就给我们留下了 Variance。说实话，你也可以放弃这个，但我还有作为一个数据结构人的自尊，所以我们要去做这些 Variance 东西。

那么，惊讶吧：Rust 有子类型。特别地，`&'big T` 是`&'small T`的一个*子类型*，为什么？因为如果某些代码需要一个在程序的某些特定区域内存活的引用，那么给它一个存活时间更长的引用通常是很好的。就像，从直觉上来说，这就是真的，对吗？

为什么这很重要呢？想象一下，有些代码需要两个相同类型的值:

```rust ,ignore
fn take_two<T>(_val1: T, _val2: T) { }
```

这是一些非常无聊的代码，因此我们应该期望它在 T=&u32 的情况下能正常工作，对吗？

```rust
fn two_refs<'big: 'small, 'small>(
    big: &'big u32, 
    small: &'small u32,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

是的，它编译得很好!

现在我们来玩玩，把它包在，哦，我不知道，`std::cell::Cell` 里:

```rust ,compilefail
use std::cell::Cell;

fn two_refs<'big: 'small, 'small>(
    // NOTE: 改了这两行
    big: Cell<&'big u32>, 
    small: Cell<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
error[E0623]: lifetime mismatch
 --> src/main.rs:7:19
  |
4 |     big: Cell<&'big u32>, 
  |               ---------
5 |     small: Cell<&'small u32>,
  |                 ----------- these two types are declared with different lifetimes...
6 | ) {
7 |     take_two(big, small);
  |                   ^^^^^ ...but data from `small` flows into `big` here
```

嗯??? 我们没有碰过生命周期，为什么编译器现在生气了！？

啊，生命周期的 "子类型 "的东西一定很简单，所以如果你用任何东西包住引用，它就会失败，看看它在 Vec 中是不是也会失败:

```rust
fn two_refs<'big: 'small, 'small>(
    big: Vec<&'big u32>, 
    small: Vec<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/playground`
```

看，它不能编译......等等，什么??? Vec 有魔法??????

嗯，是的, 但也不是。这个魔法一直都在我们体内，这个魔法就是✨*Variance*✨。

如果你想了解所有血腥的细节，请阅读[nomicon 关于子类型的章节](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)，但基本上子类型*并不*总是安全的。特别是当涉及到可变引用时，它并不安全，因为你可以使用像 `mem::swap` 这样的东西，然后突然间就出现了悬空的指针！

那些 "像可变引用" 的东西是*不变量*(invariant)，这意味着它们会阻止在它们的泛型参数上发生子类型化。所以为了安全起见，`&mut T` 在 T 上是不变量，而 `Cell<T>` 在 T 上是不变量，因为 `&Cell<T>` 基本上就是 `&mut T`（因为内部可变性）。

几乎所有不是不变量的东西都是*协变量*(covariant)，这只是意味着子类型化 "通过 "它并继续正常工作（也有逆变量(contravariant)，使子类型化向后退，但它们真的很罕见，没有人喜欢它们，所以我不会再提它们）。

数据结构通常包含一个指向其数据的可变指针，所以你可能希望它们也是不变量，但事实上，它们不需要是不变量! 由于 Rust 的所有权系统，`Vec<T>`在语义上等同于 `T`，这就意味着它可以很安全地是协变的！

不幸的是，这个定义是不变量:

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

但是Rust是如何决定事物的变异性(variance)的呢？在1.0之前的好日子里，我们只是让人们指定他们想要的变异性，但......这绝对是一场灾难。子类型和变异性真的很难让人理解，而且核心开发人员在基本术语上确实存在分歧！因此，我们转而采用了 "以实例说明变异性 "的方法：编译器只是查看你的字段并复制它们的变异性。如果有任何分歧，那么不变量总是获胜，因为那是安全的。

那么，在我们的类型定义中，有什么东西让Rust生气了呢？`*mut`!

Rust 中的裸指针实际上允许你做任何事情，但它们恰恰有一个安全特性：因为大多数人不知道Rust中存在变异性和子类型，而*不正确*的协变量将是非常危险的，`*mut T` 是不变量，因为它很有可能被 "作为" `&mut T` 使用。

作为一个花了很多时间在Rust中编写数据结构的人，这对我来说是非常恼火的。这就是为什么我在制作[std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html)时，加入了这个小魔法:

> 与`*mut T`不同，`NonNull<T>`被选择为对 `T`是协变量。这使得在构建协变类型时使用 `NonNull<T>` 成为可能，但如果在一个实际上不应该是协变的类型中使用，就会引入不健全的风险。

但是，嘿，它的接口是围绕`*mut T`建立的，这算什么! 难道这只是魔法吗！？让我们来看看:

```rust
pub struct NonNull<T> {
    pointer: *const T,
}


impl<T> NonNull<T> {
    pub unsafe fn new_unchecked(ptr: *mut T) -> Self {
        // SAFETY: the caller must guarantee that `ptr` is non-null.
        unsafe { NonNull { pointer: ptr as *const T } }
    }
}
```

不，不。这里没有魔法! NonNull 只是滥用了 `*const T` 是协变量这一事实，并将其存储起来，在API边界的`*mut T`之间来回转换，使其 "看起来"像是在存储一个`*mut T`。真是所有技巧！这就是为什么Rust中的数据结构是协变的! 而这是很痛苦的! 所以我为你做了“好的指针类型”! 不客气! 享受你的子类型吧!

解决你所有问题的方法是使用 NonNull，然后如果你想再次拥有可空的指针，使用 `Option<NonNull<T>>`。我们真的要这么做吗？

是的！这很糟糕，但我们要做的是生产级的链表，所以我们要吃下所有的蔬菜，用艰难的方式来做事情（我们可以直接使用裸`*const T`并到处类型强转，但我真的想看看这有多痛苦...为了人体工程学）。

所以，这是我们最终的类型定义:

```rust
use std::ptr::NonNull;

// !!!这里变了!!!
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

...等等，不，最后一件事。任何时候你做原始指针的事情，你都应该添加一个幽灵来保护你的指针:

```rust ,ignore
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    /// We semantically store values of T by-value.
    _boo: PhantomData<T>,
}
```

在这种情况下，我认为我们*实际上*不需要[PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)，但任何时候你使用 NonNull（或一般的原始指针），你都应该添加它以确保安全，并让编译器和其他人清楚你*认为*你在做什么。

PhantomData 是我们给编译器提供一个额外的 "例子 " 字段的方法，这个字段*只在概念上*存在于你的类型中，但由于各种原因（指令、类型清除...）并不存在。在这个例子中，我们使用 NonNull 是因为我们声称我们的类型 "就像 " 它存储了一个值 T 一样，所以我们添加一个 PhantomData 来明确这一点。

stdlib 实际上有其他理由这样做，因为它可以访问被诅咒的[Drop Check overrides](https://doc.rust-lang.org/nightly/nomicon/dropck.html)重写，但是这个功能已经被重做了很多次，我实际上不知道 PhantomData 这个东西对它来说是否还是*有用*了。我还是会永远崇拜它，因为Drop Check的魔力已经烙在我的脑子里了！

(Node字面意思是存储一个 T，所以它不必做这个，耶！)

......好了，我们现在已经完成了布局! 接下来是实际的基本功能!
