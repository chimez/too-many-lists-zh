# Send, Sync, and Compile Tests

好吧，实际上我们确实还有一对 traits 需要考虑，但它们很特别. 我们要对付的是Rust的神圣罗马帝国。不安全的选择内置特性（Opt-In Built-In Traits, OIBITs）。[Send and Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html)，这实际上是选择不要(opt-out)和默认不在(build-out)的（3个中有1个相当不错！）。

像Copy一样，这些 traits 完全没有与之相关的代码，只是标记你的类型有一个特定的属性. Send 表示你的类型可以安全地发送到另一个线程。Sync 表示你的类型可以在线程之间安全共享（&Self: Send）。

关于 LinkedList是协变的论点也适用于此：一般来说，不使用花哨的内部可变性技巧的普通数据结构可以安全地进行Send和Sync。

但是我说过他们是*opt out*。那么实际上，我们已经是了吗？我们怎么会知道呢？

让我们给我们的代码添加一些新的魔法：除非我们的类型有我们期望的属性，否则随机的私有垃圾不会被编译。 

```rust ,ignore
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```text
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<a million more errors>
```

哦，天哪，什么原因！？我有那个伟大的神圣罗马帝国的笑话!

好吧，当我说原始指针只有一个安全保护时，我骗了你：这是另一个。`*const`和`*mut`为了安全起见，明确选择不要 Send和Sync，所以我们*实际上*必须选择要他们。

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

请注意，我们必须在这里写上 *unsafe impl* : 这些是 *unsafe traits* ! 不安全的代码（如并发库） 只可以依靠我们自己正确地实现这些traits.  由于没有实际的代码，我们所做的保证只是，是的，我们在线程之间发送或共享确实是安全的!

不要轻易地把这些拍上去，我是一个认证的专业人员，在这里说：是的，那里的是完全没有问题。请注意，我们不需要为IntoIter实现Send和Sync：它只是包含了LinkedList，所以它自动衍生出Send和Sync &mdash；我告诉过你，它们实际上是可以选择不要的！ (你可以用 `impl !Send for MyType {}` 这种搞笑的语法来选择不要)

Don't just slap these on lightly, but I am a Certified Professional here to say: yep there's are totally fine. Note how we don't need to implement Send and Sync for IntoIter: it just contains LinkedList, so it auto-derives Send and Sync &mdash; I told you they were actually opt out! (You opt out with the hillarious syntax of `impl !Send for MyType {}`.)

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

好的，不错!

...等等，实际上，如果那些东西*不应该*是它们不是的那种，那就真的很危险了。特别是IterMut *完全* 不应该是协变的，因为它 "像" `&mut T`。但是我们怎么能检查呢？

用 "魔法"! 好吧，实际上是用rustdoc!  好吧，我们不一定要用rustdoc，但这是最有趣的方法。 你看，如果你写一个文档并包含一个代码块，那么rustdoc会尝试编译并运行它，所以我们可以用它来制作新的匿名 "程序"，同时不影响主程序:

```rust ,ignore
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```


好的，我们已经证明了它的不变性，但是，现在我们的测试失败了。不用担心，Rustdoc可以让你通过在栅栏上注解 compile_fail 来说明这是在预料之中的!

(实际上，我们只证明了它是 "非协变的"，但老实说，如果你设法使一个类型 "意外地、不正确地成为逆变的"，那只能恭喜你了?）。

```rust ,ignore
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

耶！我建议总是在没有 compile_fail 的情况下进行测试，这样你就可以确认它编译失败的*正确原因*。 例如，如果你忘记了 `use` , 这个测试也会失败（因此也会通过），这不是我们想要的结果！。 虽然从概念上讲，能够从编译器中 "要求" 一个特定的错误是很吸引人的，但这绝对是一个噩梦，它将是一个编译器的不兼容改变: *使编译器产生更好的错误*。我们希望编译器变得更好，所以，不，你不能有这样的要求。

(哦，等等，我们实际上可以在 compile_fail 旁边指定我们想要的错误代码，**但这只在nightly上有效，而且由于上述原因，这是一个不好的主意。在非 nightly ，它将被默默地忽略。**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...还有，你注意到我们实际上使IterMut不变的部分吗？这很容易被忽略，因为我 "只是" 复制粘贴了Iter并把它放在了最后。这是它的最后一行：

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

让我们试着删除那个PhantomData:

```text
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

哈! 编译器让我们改回去，不会让我们*不*使用生命周期。让我们试试用这个*错误的*例子来代替:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

它编译了! 我们的测试现在抓住问题了吗？

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

好嗷嗷!!! 该系统是有效的! 我喜欢真正完成它的工作的测试，这样我就不必对迫在眉睫的错误感到恐惧了!
