---
title: "rsync"
date: 2019-03-22T23:35:35+08:00
draft: false
tags: [tools, linux]
description: ""
---

### 拷贝本地文件

```
rsync -rtv src_folder/ dst_folder/ 
```

src_folder 后面要加 slash，否则在 dst_folder 中会创建一个与 src_folder 同名的文件夹。
加了 slash 的话，只会拷贝 src_folder 中的文件到 dst_folder 中，不会创建文件夹。

选项含义：

- `-r`: 递归，拷贝所有子文件夹
- `-t`: 拷贝后的文件会保持原来的 modification time 属性
- `-v`: verbose

#### `-a` 选项

- makes the copy recursive
- preserve the modification times
- copies the symlinks that it encounters as symlinks
- preserve the permissions,  owner and group information
- preserve device and special files. 

This is useful if you are copying the entire home folder of a user, or if you are copying system folders somewhere else.

### 特殊符号 escape

可以用单引号把文件夹的名字包裹起来，escape 掉空格之类的符号。

```
rsync -rtv 'so{ur ce/' 'dest ina{tion/'
```

### 增量拷贝

```
rsync -rtvuc source_folder/ destination_folder/
```

- `-u`：考虑两端文件的 modification time 和 size 决定是否增量传输。
- `-c`：对两端文件计算 checksum，不相等的进行 copy

### 同步两端文件

同步的意思是：

- 向远端拷贝本地添加的文件
- 删除远端上本地已删除的文件

```
rsync -rtvu --delete source_folder/ destination_folder/
```

可以选择删除文件的时机：

- `--delete-before`
- `--delete-after`
- `--delete-during`
- `--delete-delay`

### 传输时压缩

使用 `-z` 在传输时压缩。

### 在本地和远端之间传输文件

如果远端开着 ssh/rsh/rsync 服务，就可以在本地和远端之间传输文件。

命令格式：

```
rsync -rtvz source_folder/ user@domain:/path/to/destination_folder/
rsync -rtvz source_folder/ user@xxx.xxx.xxx.xxx:/path/to/destination_folder/
rsync -rtvz source_folder/ server_name:/path/to/destination_folder/
```

### exclude 某些文件

#### 1. 使用 `--exclude` 选项。

```
rsync -rtv --exclude 'path/to/directory' source_folder/ destination_folder/
rsync -rtv --exclude 'path/to/file.txt' source_folder/ destination_folder/
```

#### 2. 使用 `--exclude-from` 结合文件。

```
rsync -rvz --exclude-from 'excluded.txt' source_folder/ destination_folder/
```

excluded.txt 文件的内容示例：

```
directory
relative/path/to/directory
file.txt
relative/path/to/file.txt
/home/juan/directory
/home/juan/file.txt
*.swp
```

并且可以使用 `--delete-excluded` 选项来删除已经存在，但被 exclude 的文件。

### 附加选项

- `-t` Preserves the modification times of the files that are being transferred.
- `-q` Suppress any non-error message, this is the contrary to `-v` which increases the verbosity.
- `-d` Transfer a directory without recursing, this is, only the files that are in the folder are transferred.
- `-l` Copy the symlinks as symlinks.
- `-L` Copy the file that a symlink is pointing to whenever it finds a symlink.
- `-W` Copy whole files. When we are using the delta-transfer algorithm we only copy the part of the archive that was updated, sometimes this is not desired.
- --progress Shows the progress of the files that are being transferred.
- `-h` Shows the information that rsync provides us in a human readable format, the amounts are given in K's, M's, G's and so on.

Reference: [Synchronizing folders with rsync](https://www.jveweb.net/en/archives/2010/11/synchronizing-folders-with-rsync.html)


