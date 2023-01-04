list 在 C++ 里代表双向链表. 和 vector 相比, 它优化了在容器中间的插入和删除：

- list 提供高效的、O(1) 复杂度的任意位置的插入和删除操作
- list 不提供使用下标访问其元素
- list 提供 push_front、emplace_front 和 pop_front 成员函数（和 deque 相同）
- list 不提供 data、capacity 和 reserve 成员函数（和 deque 相同）

它的内存布局一般是下图这个样子：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ysMl1A.jpg)

虽然 list 提供了任意位置插入新元素的灵活性, 但由于每个元素的内存空间都是单独分配、不连续, 它的遍历性能比 vector 和 deque 都要低. 这在很大程度上抵消了它在插入和删除操作时不需要移动元素的理论性能优势. 如果你不太需要遍历容器、又需要在中间频繁插入或删除元素, 可以考虑使用 list.
