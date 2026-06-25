# Python 文件写入入门 —— 第一二梯队

> 接上一份文件读取的文档，这次专门讲"写"。
> 每个例子请自己敲一遍运行看看。

---

## 一、写文件的三种基本模式

### open() 写模式对比

| 模式 | 意思 | 文件不存在时 | 文件存在时 |
| --- | --- | --- | --- |
| `w` | 写入 | 自动创建 | **清空后写** |
| `a` | 追加写入 | 自动创建 | 在文件末尾追加 |
| `x` | 排它创建 | 创建成功 | 报错 FileExistsError |

### w 模式 —— 最常用的写入模式

```python
# w 模式：打开瞬间清空文件，然后写内容
with open('b.txt', 'w', encoding='utf-8') as file:
    file.write('你好')
```

运行后，b.txt 的内容就是 `你好`。如果之前有内容，全部被覆盖。

> **危险提醒**：`'w'` 在 `open()` 的那一刻就已经清空了文件，不是等到 `write()` 才清空。
>
> ```python
> # 这段代码会把 data.txt 清空，什么都没写进去
> with open('data.txt', 'w', encoding='utf-8') as file:
>     pass   # 什么都没写，但文件已经被清空了
> ```

### a 模式 —— 追加，不丢数据

```python
# a 模式：不清空，在末尾追加
with open('a.txt', 'a', encoding='utf-8') as file:
    file.write('你好')
```

每次运行，`你好` 就追加到文件末尾。适合写日志。

### x 模式 —— 防止覆盖

```python
# x 模式：文件已存在就报错
with open('demo.txt', 'x', encoding='utf-8') as file:
    file.write('你好')
```

如果 demo.txt 已经存在，Python 会报 `FileExistsError`。用来防止不小心覆盖已有文件。

---

## 二、write() 的缓冲机制 —— 为什么有时候写进去了但文件里没有

### 核心概念：缓冲区

Python 写入文件时，**不是写一次就立刻存到硬盘上**，而是先攒在内存的"缓冲区"里。缓冲区满了或者文件关闭时，才一次性写入磁盘。

这是一个**性能优化** —— 攒一批再写，比写一次就存一下快得多。

```python
import time

with open('demo.txt', 'w', encoding='utf-8') as file:
    file.write('第 1 次写入')
    file.write('第 2 次写入')
    # 文件关闭时，这两个 write 一次性写到磁盘
```

### flush() —— 强制写入

如果你需要在文件关闭**之前**就让数据落地（比如写日志后要立刻查看），用 `flush()`。

```python
import time

with open('demo.txt', 'w', encoding='utf-8') as file:
    file.write('第 1 次写入')
    file.write('第 2 次写入')
    file.flush()                # 立即写入磁盘
    # 到这里，前两次写入的内容已经在文件里了
    time.sleep(3)               # 你可以打开文件看一下
    file.write('第 3 次写入')   # 这三秒后写进去
    file.write('第 4 次写入')
    # 文件关闭时，第 3、4 次写入才落地
```

> 绝大多数情况不需要手动调 `flush()`，因为 `with` 结束时会自动关闭文件，顺带把缓冲区的数据全部写入。只有需要**即时查看**时才用。

---

## 三、seek() —— 移动文件指针

### 为什么需要 seek？

写文件时，指针默认在末尾（`a` 模式）或开头（`w` 模式）。如果你想**在写入之后回头读一读**，就需要把指针移回开头。

```python
# 写完之后想读一下
with open('a.txt', 'w+', encoding='utf-8') as file:
    file.write('你好')
    file.seek(0, 0)        # 指针移到文件开头
    result = file.read()
    print(result)          # 输出：你好
```

没有 `seek(0, 0)` 的话，`read()` 从指针当前位置（文件末尾）读，读到空字符串。

### seek() 的两个参数

| 参数 | 说明 | 常用值 |
| --- | --- | --- |
| offset | 偏移量（移动多少个字符/字节） | 0 表示不偏移 |
| whence | 从哪里开始算 | 0 = 文件开头，1 = 当前位置，2 = 文件末尾 |

最常用的就是 `seek(0, 0)` —— 回到文件开头。

```python
# 指针移动的三种方式
with open('a.txt', 'w+', encoding='utf-8') as file:
    file.write('0123456789')
    
    file.seek(0, 0)      # 回到开头
    print(file.read())   # 0123456789
    
    file.seek(3, 0)      # 从开头移 3 个字符
    print(file.read())   # 3456789
```

### 注意：文本模式下不要随意定位中文字符

```python
# 错误示范 —— 从一个汉字中间开始读，会破坏编码
with open('a.txt', 'r+', encoding='utf-8') as file:
    file.seek(1, 0)      # 一个汉字占 3 个字节，从第 1 个字节读会乱码
    print(file.read())   # 输出乱码
```

二进制模式（`'rb'`）下按字节读不会出这个问题，文本模式下建议 **seek 只用来回到开头或跳到末尾**。

---

## 四、四种读写混合模式

### 对比速查

| 模式 | 打开时文件被清空？ | 初始指针位置 | 典型用法 |
| --- | --- | --- | --- |
| `r+` | 不清空 | 文件开头 | 读文件后修改 |
| `w+` | **清空** | 文件开头 | 写完后读一下验证 |
| `a+` | 不清空 | 文件末尾 | 追加后读一下 |
| `x+` | 不清空（新文件） | 文件开头 | 安全创建后读写 |

### r+ —— 先读后写

```python
# a.txt 内容原来是 "原始内容"
with open('a.txt', 'r+', encoding='utf-8') as file:
    content = file.read()
    print('原来:', content)         # 原始内容
    file.write('新增的内容')        # 指针在末尾，追加
    file.seek(0, 0)
    print('现在:', file.read())     # 原始内容新增的内容
```

`r+` 不会清空文件，适合**读取后修改/追加**。

### w+ —— 写完后读一下验证

```python
with open('a.txt', 'w+', encoding='utf-8') as file:
    file.write('新内容')
    file.seek(0, 0)        # 写完后指针在末尾，必须移回开头才能读
    result = file.read()
    print(result)          # 新内容
```

`w+` 会清空文件，适合**重新写内容后立即验证**。

### a+ —— 追加后读一下

```python
with open('a.txt', 'a+', encoding='utf-8') as file:
    file.write('新增内容')
    file.seek(0, 0)        # a+ 的指针默认在末尾
    result = file.read()
    print(result)
```

> 注意：`a+` 模式下，**即使你 seek 到开头再 write，写的内容还是追加到末尾**。这是 `a` 模式的天生行为。

### x+ —— 安全创建新文件

```python
# 第一次运行：成功创建
# 第二次运行：报错，因为文件已存在
with open('new_file.txt', 'x+', encoding='utf-8') as file:
    file.write('新文件内容')
    file.seek(0, 0)
    print(file.read())
```

---

## 五、动手练习

### 练习 1：验证 w 模式会清空

创建一个 test.txt，写几行字。然后运行：

```python
with open('test.txt', 'w', encoding='utf-8') as file:
    file.write('我清空了原来的内容')
```

运行前打开看看，运行后再打开看看。

### 练习 2：用 a 模式写日志

```python
import time

def write_log(msg):
    with open('app.log', 'a', encoding='utf-8') as file:
        file.write(f'{time.ctime()} - {msg}\n')

write_log('程序启动')
write_log('用户登录')
write_log('用户退出')
```

运行多次，看看 app.log 的内容不断增长。

### 练习 3：写完后用 seek 读回来

```python
with open('demo.txt', 'w+', encoding='utf-8') as file:
    file.write('ABCDEFGHIJ')
    file.seek(0, 0)         # 移回开头
    print(file.read(5))     # 应该输出 ABCDE
    file.seek(5, 0)         # 跳过后 5 个
    print(file.read())      # 应该输出 FGHIJ
```

### 练习 4：验证缓冲区

```python
import time

with open('buffer_test.txt', 'w', encoding='utf-8') as file:
    file.write('第 1 条')
    print('已写入第 1 条，3 秒内可以打开文件看看有没有内容')
    time.sleep(3)
    file.flush()
    print('调用了 flush，现在再去看看文件')
    time.sleep(3)
    file.write('第 2 条')
    print('已写入第 2 条，文件关闭前再看看')
    time.sleep(3)
# 文件关闭，所有内容落盘
print('文件已关闭，现在文件里应该有两行内容')
```

---

## 六、知识点速记

- **`w` 模式**：清空后写，最常用，但要小心
- **`a` 模式**：追加，适合日志
- **`x` 模式**：排它创建，防止覆盖
- **缓冲区**：write 不立即落盘，文件关闭或 flush() 时才写
- **`flush()`**：强制将缓冲区的数据写入磁盘
- **`seek(0, 0)`**：把指针移回文件开头，用于写完后读
- **`seek(offset, whence)`**：where 0=开头, 1=当前位置, 2=末尾
- **`r+`**：读写，不清空
- **`w+`**：清空后读写
- **`a+`**：追加后读（指针在末尾）
- **`x+`**：创建新文件后读写
