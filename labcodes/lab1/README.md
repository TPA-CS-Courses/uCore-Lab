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
