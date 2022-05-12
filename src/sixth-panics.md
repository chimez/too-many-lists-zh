# Drop and Panic Safety

所以，嘿，你是否注意到这个评论:

```rust
// 如果前面没有，那么我们就是空列表，也需要设置后面的。
// 此外，这里还有一些完整性检查供测试，以防我们乱搞。
```

这样做对吗？

对不起，你忘了你正在读的书吗？当然是错的! (某种程度上)

让我们再看一下 pop_front 的内部结构:

```rust ,ignore
// 让 Box 复活，这样我们就可以 move 出它的值并 Drop 它
// （Box 继续神奇地为我们理解这个）。
let boxed_node = Box::from_raw(node.as_ptr());
let result = boxed_node.elem;

// 使下一个节点成为新的头.
self.front = boxed_node.back;
if let Some(new) = self.front {
    // 清理它对被移除节点的引用
    (*new.as_ptr()).front = None;
} else {
    // 如果前面是空的，那么这个列表现在就是空的了
    debug_assert!(self.len == 1);
    self.back = None;
}

self.len -= 1;
result
// Box 在这里被隐式释放了，知道没有 T 了。
```

你看到这个错误了吗？可怕的是，它实际上是这一行:

```rust ,ignore
debug_assert!(self.len == 1);
```

*真的*？我们该死的用于测试的完整性检查是一个错误吗？是的！! 好吧，如果我们实现了正确的数据结构，它就*不应该*是，但是它可以把一些良性的东西，比如 "哦，我们在保持len的更新方面做得不好"变成*一个可利用的内存安全 bug*! 为什么？因为它能 panic! 大多数时候，你不需要考虑或担心 panic，但是一旦你开始写 *真正* 不安全的代码，并且快速地玩弄 "不变量"，你就需要对 panic 保持高度警惕!

我们得谈谈[*异常安全*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html)（又称 panic 安全，又称 unwind 安全，...）。

所以，事情是这样的：默认情况下，panic 是 *unwinding* 。解散只是说 "让每一个函数立即返回" 的一种花哨方式。你可能会想 "好吧，如果*每个函数*都返回，那么程序就快死了，为什么要关心它呢？"，但你错了！

我们必须关心的原因有两个：当一个函数返回时，析构器会运行，而 unwind 可以被*caught*。在这两种情况下，代码可以在panic之后继续运行，所以我们需要非常小心，确保我们的不安全集合在可能发生恐慌的时候总是处于*某种*连贯的状态，因为每次恐慌都是隐式的提前返回!

让我们想一想，当我们走到那一行时，我们的数据结构处于什么状态:

我们在栈上有我们的 boxed_node ，并且我们已经从里面提取了元素。如果我们在这时返回，Box 将被 drop，而节点将被释放。你现在看到了吗？self.back 仍然指向那个被释放的节点。一旦我们实现了数据结构的其余部分，并开始使用 self.back 来做一些事情，这可能会导致 use-after-free！你知道吗？哈!

有趣的是，这一行也有类似的问题，但它更安全:

```rust ,ignore
self.len -= 1;
```

默认情况下，在调试构建中，Rust 会检查下溢和溢出，当它们发生时就会 panic。是的，每一个算术运算都是一个panic安全的危险！这个*更好*，因为它发生在我们修复了所有的不变量之后，所以它不会导致内存安全问题......只要我们不相信len是对的，但是，如果我们下溢，那肯定是错的，所以我们无论如何都会死的！调试断言在某种意义上是*糟糕的*，因为它可以将一个小问题升级为一个关键问题!

我曾多次提到 "不变量 "这个术语，这是因为它对 panic 安全来说是一个非常有用的概念! 基本上，对于我们数据结构的外部观察者来说，有一些属性是我们一直在维护的。 对于 LinkedList 来说，其中之一就是任何在我们的链表中可以到达的节点会被分配和初始化。

在实现的*内部*，我们有更多的灵活性来*暂时*破坏不变性，只要我们确保*在别人发现之前*修复它们。这实际上是Rust对集合的所有权和借用系统的 "杀手级应用 " 之一：如果一个操作需要一个 `&mut Self`，那么我们就可以*保证* 我们对我们的数据结构有独占的访问权，并且我们可以暂时破坏不变性，安全地知道没有人可以偷偷地破坏它。

也许这一点的最大体现是[Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain)， 它实际上让你完全粉碎了 Vec 的一个核心不变性，并开始从Vec的 *前端*甚至*中间*移出值。这之所以是*合理的*，是因为我们返回的 Drain 迭代器持有对Vec的 `&mut` ，并且所以所有的访问都是在它的后面。在 Drain 迭代器消失之前，没有人可以观察到 Vec，然后它的析构器可以在任何人注意到之前 "修复" Vec，它是完------

[这并不完美](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain)。不幸的是，你[不能在你不控制运行的代码中依赖析构器](https://doc.rust-lang.org/std/mem/fn.forget.html)，所以即使有了Drain，我们也需要做一点额外的工作来使我们的类型总是保留不变性，但要用一种愚蠢的方式。[我们只是在开始时将Vec的len设置为0](https://doc.rust-lang.org/std/mem/fn.forget.html)，所以如果有人泄露了Drain，那么他们将拥有一个*安全的*Vec......但他们也将失去一堆数据。你泄露了我？我泄露你! 以眼还眼! 真正的正义!

关于你*可以*使用析构器来实现恐慌安全的情况，请查看 [BinaryHeap::sift_up case study](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up).

总之，我们的 LinkedLists 不需要所有这些花哨的东西，我们只需要对我们破坏不变量的地方、对我们信任/要求正确的东西保持更多的警惕，并避免在有问题的任务中引入不必要的 unwind。

在这种情况下，我们有两个选择来使我们的代码更加健壮:

* 更积极地使用像 `Option::take` 这样的操作，因为它们更具有 "事务性"，并有保留不变性的倾向。

* 干掉 debug_asserts，相信我们自己能写出更好的测试，有专门的 "完整性检查" 函数，并且永远不会在用户代码中运行。

原则上，我喜欢第一个选项，但它实际上对一个双向链表并不奏效，因为所有东西都是双重冗余编码的。Option::take 不能解决这个问题，但是把 debug_assert 往下移一行就可以。 但是说真的， 为什么要给自己找麻烦呢？ 让我们删除这些 debug_assert ，并确保任何可以恐慌的东西都在我们的方法的开始或结束处，应该知道在那里我们的不变性是成立的。

(在这种情况下，把它们看作是*前条件*和*后条件*也许更准确，但你真的应该尽可能地把它们当作不变量来对待！)

下面是我们现在的完整实现:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // 安全性: 这是个链表，你还想要什么?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // 把新的头放在旧的前面
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // 如果前面没有，那么我们就是空列表，也需要设置后面的。
                self.back = Some(new);
            }
            // 这些东西永远都有！
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // 只有在有前面的节点需要弹出时才需要做一些事情。
            self.front.map(|node| {
                // 让 Box 复活，这样我们就可以 move 出它的值并 Drop 它
                // （Box 继续神奇地为我们理解这个）。
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // 使下一个节点成为新的头.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // 清理它对被移除节点的引用
                    (*new.as_ptr()).front = None;
                } else {
                    // 如果前面是空的，那么这个列表现在就是空的了
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box 在这里被隐式释放了，知道没有 T 了。
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

这里有什么可 panic 的？好吧，要知道这一点需要你是一个Rust专家，但值得庆幸的是，我是一个Rust专家!

在这段代码中，我看到的唯一*可能*发生恐慌的地方（除非有人在启用debug_asserts的情况下重新编译stdlib，但这不是你应该做的）是`Box::new`（在内存不足的情况下）和 len 运算。所有这些东西都在我们的方法的最后或最开始，所以是的，我们很好，很安全!

...你对`Box::new`能够恐慌感到惊讶吗？恐慌会给你带来麻烦! 尽量保留这些不变量，这样你就不需要担心了！"。

