# Rust

## 错误处理

### Panic 机制

- unwind: 在发生 panic 的时候, 会一层一层的退出函数调用栈, 在此过程中, 当前栈内的局部变量还是可以正常析构
- abort: 在发生 panic 的时候, 会直接退出整个程序

### Panic Safety

C++ 的 exception safety 机制:
- `No-throw`: 这种层次的安全性, 保证了所有的异常都在内部正确处理完毕, 外部毫无影响
- `Strong exception safety`: 强异常安全保证, 可以保证异常发生的时候, 所有的状态都可以"回滚"到初始状态, 不会导致状态不一致问题
- `Basic exception safety`: 基本异常安全保证, 可以保证异常发生的时候, 不会导致资源泄漏
- `No exception safety`: 没有任何异常安全保证
