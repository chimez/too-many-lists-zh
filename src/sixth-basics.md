# 基础功能 

好了，这就是本书中最糟糕的部分，也是为什么我花了7年时间才写完这一章的原因 是时候把一大堆我们已经做过 5 次的非常无聊的东西快速掠过了，但是因为我们必须做两次，而且要用`Option<NonNull<Node<T>>`，所以特别啰嗦，特别长！

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }
}
```
PhantomData 是一个没有字段的奇怪类型，所以你只需说出它的名字就能实现一个。*耸肩*

```rust ,ignore
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
            (*old).front = Some(new);
            (*new).back = Some(old);
        } else {
            // 如果前面没有，那么我们就是空列表，也需要设置后面的。
            // 此外，这里还有一些完整性检查供测试，以防我们乱搞。
            debug_assert!(self.back.is_none());
            debug_assert!(self.front.is_none());
            debug_assert!(self.len == 0);
            self.back = Some(new);
        }
        self.front = Some(new);
        self.len += 1;
    }
}
```

```text
error[E0614]: type `NonNull<Node<T>>` cannot be dereferenced
  --> src\lib.rs:39:17
   |
39 |                 (*old).front = Some(new);
   |                 ^^^^^^
```


啊，是的，我真的很讨厌我的指针式的孩子。我们需要用`as_ptr`明确地从 NonNull 中获取原始指针，因为 DerefMut 是以`&mut`来定义的，我们不想在不安全的代码中随意引入安全引用

```rust ,ignore
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
```

```text
   Compiling linked-list v0.0.3
warning: field is never read: `elem`
  --> src\lib.rs:16:5
   |
16 |     elem: T,
   |     ^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: `linked-list` (lib) generated 1 warning (1 duplicate)
warning: `linked-list` (lib test) generated 1 warning
    Finished test [unoptimized + debuginfo] target(s) in 0.33s
```

很好，现在是 pop（和 len）。

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    unsafe {
        // 只有在有前面的节点需要弹出时才需要做一些事情。
        // 请注意，我们不需要再使用 `take' 了，因为所有的东西都是Copy，
        // 没有任何 dtors 会在我们搞砸的时候运行......对吗？） 对吗吗吗? :))
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
                debug_assert!(self.len == 1);
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
```

```text
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
```

在我看来是合理的，是时候写一个测试了！

```rust ,ignore
#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // 尝试打破一个空列表
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```


```text
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src\lib.rs

running 1 test
test test::test_basic_front ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

万岁，我们是完美的!

......对吗？

