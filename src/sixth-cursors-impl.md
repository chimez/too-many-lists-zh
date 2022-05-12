# 实现光标 

好的，所以我们只打算用std的CursorMut来处理，因为不可变的版本实际上并不有趣。就像我最初的设计一样，它有一个包含None的 "幽灵" 元素来表示列表的开始/结束，你可以 "走过它" 来环绕到列表的另一边。为了实现它，我们将需要:

* 一个指向当前节点的指针
* 一个指向列表的指针
* 当前的索引

等等，当我们指向 "幽灵" 的时候，索引是什么？

*眉头紧皱* ... *检查std* ... *不喜欢std的答案*

好吧，很合理地，游标上的`index`返回一个`Option<usize>`。std的实现做了一堆垃圾来避免将其存储为一个Option，但是......我们是一个链表，这很好。std 也有  cursor_front/cursor_back 的东西，它在前面/后面的元素上启动光标，这感觉很直观，但是当列表为空时，必须做一些奇怪的事情。

如果你想的话，你可以实现这些东西，但是我打算减少所有重复的垃圾和极端的情况，只做一个光秃秃的`cursor_mut`方法，从幽灵开始，人们可以使用 move_next/move_prev 来获得他们想要的（如果你真的想的话，你可以把它包起来作为cursor_front）。

让我们开始行动吧:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

很直接，我们的列表中的每一个项目都有一个字段。现在是`cursor_mut`方法:

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

既然我们从幽灵开始，我们就可以从一切都 None 开始，很好，很简单! 接下来是移动：

```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }
}
```

所以有4种有趣的情况。

* 正常的情况
* 正常的情况，但我们到达了幽灵的位置
* 幽灵的情况，我们走到了列表的前面
* 幽灵的情况，但是列表是空的，所以什么都不做

move_prev 是完全相同的逻辑，但是前后颠倒了，索引变化也颠倒了:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // We're on a real element, go to its previous (front)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // We just walked to the ghost, no more index
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // We're at the ghost, and there is a real back, so move to it!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // We're at the ghost, but that's the only element... do nothing.
    }
}
```

接下来让我们添加一些方法来查看光标周围的元素：current, peek_next, 和 peek_prev。**非常重要的标注：**这些方法必须通过`&mut self` 来借用我们的光标，而且结果必须与该借用相联系。我们不能让用户得到一个可变引用的多个副本，也不能让他们在抓着这样的引用时使用我们的任何插入/删除/拆分/分片的API!

值得庆幸的是，当你使用生命周期消除时，这是Rust的默认假设，所以，我们将默认做正确的事情!

```rust ,ignore
pub fn current(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

头部为空，Option 方法和（省略的）编译器错误现在都在思考。我对`Option<NonNull>`的东西持怀疑态度，但是，该死的它真的让我自动驾驶这段代码。 我花了太多的时间来写基于数组的数据结构，在那里你从来没有使用过Option，哇，这真好！"。(`(*node.as_ptr())`还是很惨的，但是，这只是Rust的原始指针...)

接下来我们有一个选择：我们可以直接跳到分割和拼接，这就是这些API的全部意义，或者我们可以在单元素插入/移除方面采取一个小步骤。我有一种感觉，我们只是想在分割和拼接方面实现插入/删除，所以......让我们先做这些，看看结果如何（当我写到这里的时候，我真的不知道）。


# 拆分

首先是 split_before 和 split_after, 它们将当前元素之前/之后的所有内容作为 LinkedList 返回（在幽灵元素处停止，除非你在幽灵处，在这种情况下，我们只是返回整个List，并且光标现在指向一个空列表）。

*斜眼看* 好的，这个实际上是一些不简单的逻辑，所以我们必须一步一步地谈出来。

我看到split_before有4种潜在的有趣情况。

* 正常情况
* 正常情况，但prev是幽灵
* 幽灵情况，我们返回整个列表，成为空的。
* 幽灵情况，但是列表是空的，所以什么都不做，返回空列表

让我们从极端情况开始。第三种情况我认为就是

```rust
mem::replace(self.list, LinkedList::new())
```

对吗？我们变成了空的，我们返回整个列表，而且我们的字段已经是 None，所以没有什么可更新的。很好。哦，嘿嘿，这也是在第四种情况下做的正确的事情!

现在是正常情况......好吧，我需要一些ASCII图来说明。在最一般的情况下，我们有这样的东西:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

而我们想要产生这个:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```


所以我们需要打破cur和prev之间的联系，而且......上帝啊，需要改变的东西太多了。好吧，我只需要把这分成几个步骤，这样我就能说服自己这是有意义的。这将是一个有点过于冗长，但我至少可以让它有意义:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // We are pointing at a real element, so the list is non-empty.
        unsafe {
            // Current state
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // What self will become
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // What the output will become
            let output_len = old_len - new_len;
            let output_front = self.list.front;
            let output_back = prev;

            // Break the links between cur and prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            }

            // Produce the result:
            self.list.len = new_len;
            self.list.front = new_front;
            self.list.back = new_back;
            self.index = new_idx;

            LinkedList {
                front: output_front,
                back: output_back,
                len: output_len,
                _boo: PhantomData,
            }
        }
    } else {
        // We're at the ghost, just replace our list with an empty one.
        // No other state needs to be changed.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

注意：这些 if-let 处理“正常情况，但是 prev 是幽灵” 的情况:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

如果*你*想，你可以把这一切都压缩住，然后做优化，比如:

* 把两次访问 `(*cur.as_ptr()).front` 写成 `(*cur.as_ptr()).front.take()` 
* 注意，new_back 是一个 noop ，并删除这两个

就我所知，其他的都是顺便做了正确的事情。当我们写测试时，我们会看到! (复制粘贴实现 split_after)

我已经不再犯错了，我只想尽力写出最完美的代码。这就是我*实际*写数据结构的方法：只是把事情分解成琐碎的步骤和案例，直到它能在我的脑海中出现，并且看起来万无一失。然后写一大堆的测试，直到我确信我没有把它搞乱。

因为我所做的大多数数据结构工作都是*极端不安全的*，我一般不会依靠编译器来捕捉错误，而 Miri 在当时并不存在！所以我只需要眯着眼睛看，直到我头疼，并尽力做到永远不犯错。

不要写不安全的Rust代码! 安全的Rust是如此的好!!!!


# 拼接

只剩下一个boss要打，就是 splice_before 和 splice_after，我预期这是最棘手的。这两个函数 *接收* 一个 LinkedList, 并将其内容移植到 outrs 中。我们的列表可能是空的，他们的列表可能是空的，我们有鬼魂要处理......。*叹气* 让我们用 splice_before 一步步来吧.

* 如果他们的列表是空的，我们就不需要做任何事情。
* 如果我们的列表是空的，那么我们的列表就成为他们的列表。
* 如果我们指向鬼魂，那么这就会追加到后面（改变list.back）。
* 如果我们指向第一个元素(0)，这个就追加到前面(change list.front)
* 在正常情况下，我们做一大堆的指针操作。

一般情况下是这样的:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

成为这样:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

好吗？好的。让我们把它写出来... *大口大口地呼吸，然后投身其中*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                *self.list = input;
            }
        }
    }
```

好吧，这个确实很可怕，现在真的感觉到了`Option<NonNull>`的痛苦。但是我们可以做很多清理工作。首先，我们可以把这段代码拉到最后，因为我们一直想这么做。我不*爱*（虽然有时是个noop，而设置`input.len`更多的是对未来代码扩展的偏执）。

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> 移动值的使用：`input`。

啊，对了，在 "我们是空的" 情况下，我们正在移动列表。让我们用交换来代替它:

```rust ,ignore
// We're empty, become the input, remain on the ghost
std::mem::swap(self.list, &mut input);
```

在这种情况下，写法将是无指针的，但是，它们仍然可以工作（我们可能也可以在这个分支中提前返回以安抚编译器）。

这种 unwrap 只是我对案例进行反向思考的结果，可以通过让 if-let 提出正确的问题来解决:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

调整索引是在分支内部重复进行的，所以也可以取出来:

```rust
*self.index.as_mut().unwrap() += input.len;
```

好吧，把这一切放在一起，我们得到这个:

```rust
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // We're appending to the front, see append to back below
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // We can either `take` the input's pointers or `mem::forget`
    // it. Using take is more responsible in case we do custom
    // allocators or something that also needs to be cleaned up!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

好吧，这仍然很糟糕，但主要是因为 -- 不，好吧，只是发现了一个 bug:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

我们 `take` input.front，然后在下一行把它 unwrap! *叹气*，我们在同样的镜像情况下做同样的事情。 我们本来可以在测试中立即发现这个问题， 但是， 我们现在正在努力做到完美， 而我只是在现场就做这个，这正是我看到它的时刻。这就是我没有像往常一样乏味，分阶段做事情的结果。更加明确!

```rust
// We can either `take` the input's pointers or `mem::forget`
// it. Using `take` is more responsible in case we ever do custom
// allocators or something that also needs to be cleaned up!
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // No prev, we're appending to the front
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

好了，现在这个，这个我可以容忍。我唯一的抱怨是，我们没有 in_front/in_back 进行重复计算（也许我们可以重新调整我们的条件，但不管怎样）。真的，这基本上就是你在C语言中写的东西，但是`Option<NonNull>`的垃圾让它变得很乏味。我可以接受这一点。好吧，我们应该让原始指针更好地用于这些东西。但是，超出了本书的范围。

总之，我已经精疲力尽了，所以，`insert` 和 `remove` 以及所有其他的API可以留给读者作为练习。

这是我们光标的最终代码，我试图复制粘贴组合代码。我做得对吗？只有当我写下一章并测试这个怪胎时才会知道!

```rust ,ignore
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }
}
```
