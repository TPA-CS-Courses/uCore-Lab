
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