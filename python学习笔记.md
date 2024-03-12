# python学习笔记

# 1. `tqdm`

当正在处理一些耗时的东西，通过显示进度条，来指示已经做了多少

安装

```text
pip install tqdm
```

使用for循环函数时，只需将其替换为`trange`

```python
from tqdm import trange

for i in trange(100):
    sleep(0.01)
```

`tqdm`不仅适用命令行环境，还适用`iPython/Jupyter`笔记本

```python
from tqdm import tnrange, tqdm_notebook
from time import sleep

for i in tnrange(4, desc='1st loop'):
    for j in tnrange(100, desc='2nd loop'):
        sleep(0.01)
```

