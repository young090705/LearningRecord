### 换源

```python
# 执行以下命令一次性配置清华源
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes
```

恢复默认源

```python
conda config --remove-key channels
```

显示所拥有的虚拟环境

```python
conda env list
```



[知乎配置matplotlib](https://zhuanlan.zhihu.com/p/9030724506)

