# Docker

## 在Docker中可能看到什么

### 文件系统隔离
文件系统的隔离是通过chroot和Union FS来实现的。

#### `chroot`
`chroot`是Linux操作系统提供的一个系统调用，该系统调用可以让进程设置特定的路径为根目录。调用了`chroot`的进程所看到文件系统都是从指定路径开始的，无法再看到更外层目录的文件系统。
```
/
|
+---tmp
|    |
|    +---foo
|    |
|    +---bar
|
+---usr
```
假设有上面一个文件系统，程序在初始启动时可以看到完整的文件系统的，当调用`chroot("/tmp")`之后，程序能所看到的文件系统将是下面的情况
```
/
|
+---foo
|
+---bar
```
#### `Union FS`
Union FS是联合文件系统，它允许对多个目录挂载到同一个挂载点，并得到一个合并的目录结构视图。Union FS有上下层关系，上下层中同名的文件在挂载点中是上层覆盖下层。
![](overlay_constructs.jpg)

假设有下面这样一个文件树视图

```
.
|
+---A
|   |
|   +---a.txt
|   |
|   +---x.txt
|
+---B
|   |
|   +---b.txt
|   |
|   +---x.txt
|
+---C
|
+---merge
|
+---work
```

通过Union FS将A,B,C合并挂载到merge目录：
```
# mount \
> -t overlay overlay \
> -o lowerdir=A:B,upperdir=C,workdir=work \
> merge
```
`lowerdir=A:B` 表示将A,B目录作为下层，并且A在B之上，因此B目录中同名的文件`x.txt`将被A目录中的覆盖。

`upperdir=C` 表示C目录作为上层，在merge目录中的发生修改文件将被写到C目录中；在merge中删除了A,B目录中同名文件之后，将在C目录中增加一个字符类设备型的同名文件。

`workdir=work` 是overlay用于工作的目录。

执行上面的`mount`命令后将得到下面一这样一个文件视图
```
.
|
+---A
|   |
|   +---a.txt
|   |
|   +---x.txt
|
+---B
|   |
|   +---b.txt
|   |
|   +---x.txt
|
+---C
|
+---merge
|     |
|     +---a.txt
|     |
|     +---b.txt
|     |
|     +---x.txt
|
+---work
```

### namespace隔离
