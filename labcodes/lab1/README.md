## gdb土法调试
调试方法记录：
修改后的makefile文件如下
```
qemu: $(UCOREIMG)
	$(V)$(QEMU) -no-reboot  -nographic -hda  bin/ucore.img
debug: $(UCOREIMG)
	$(V)$(QEMU) -no-reboot -S -s -nographic -hda bin/ucore.img   	
gdb: $(UCOREIMG)
	gdb -q -tui -x tools/gdbinit
```

`make qemu`：使用qemu模拟跑OS
`make debug`：使用qemu模拟跑OS，等待GDB链接
`make gdb`：使用GDB，并连接qemu


## vscode调试

```json
{
	"version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) 启动",
            "type": "cppdbg",
            "request": "launch",
			"program": "${workspaceFolder}/bin/kernel",
            "miDebuggerServerAddress": "localhost:1234",
            // 注意cwd这里！！
			"cwd": "${workspaceFolder}", 
 			"MIMode": "gdb",
        }

    ]
}
```
这样，启动一个终端跑`make debug`，然后用vscode在一个源文件中加断点，按F5就能开始调试啦。



## 任务
- [x] 练习5：实现函数调用堆栈跟踪函数 
        ebp：寄存器存放当前线程的栈底指针
        eip：寄存器存放下一个CPU指令存放的内存地址，当CPU执行完当前的指令后，从EIP寄存器中读取下一条指令的内存地址，然后继续执行。

        +|  栈底方向        | 高位地址
        |    ...        |
        |    ...        |
        |  参数3        |
        |  参数2        |
        |  参数1        |
        |  返回地址        |
        |  上一层[ebp]    | <-------- [ebp]
        |  局部变量        |  低位地址
    
        注意，eip，ebp都是虚拟地址
        处理这种虚拟地址方式就是将其强转为(uint32_t *)
        想要取这个地址的值，一方面可以用取值符号取得，如*addr，另一方面推荐使用数组的方式，addr[0]
- [x] 练习6：练习6：完善中断初始化和处理 
    注意，Make时候可能会出现这样的错误
    ```
    'obj/bootblock.out' size: 604 bytes
    604 >> 510!!
    make: *** [Makefile:157: bin/bootblock] Error 255
    ```
    是因为gcc版本的原因（其主要原因为5.0版本之后编译生成文件过大，选择4.7左右），链接：https://github.com/chyyuu/os_kernel_lab/issues/50
    下载4.8版本的gcc方法：
    https://blog.csdn.net/qq_31175231/article/details/77774971
    https://zhuanlan.zhihu.com/p/453542931