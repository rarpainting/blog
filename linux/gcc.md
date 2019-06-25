# GCC

## 调试

- `-g`: 只是编译器, 在编译的时候, 产生调试信息
- `-gstabs`: 此选项以 stabs 格式声称调试信息, 但是不包括 gdb 调试信息
- `-gstabs+`: 此选项以 stabs 格式声称调试信息, 并且包含仅供 gdb 使用的额外调试信息
- `-ggdb`: 此选项将尽可能多的生成 gdb 的可以使用的调试信息
- `-glevel`: 请求生成调试信息, 同时用 level 指出需要多少信息, 默认的 level 值是 2

## gcc8

[Avoid `-Wclass-memaccess` on memcpy of a POD type w/copy disabled with gcc8.3](https://stackoverflow.com/questions/57721104/avoid-wclass-memaccess-on-memcpy-of-a-pod-type-w-copy-disabled)
