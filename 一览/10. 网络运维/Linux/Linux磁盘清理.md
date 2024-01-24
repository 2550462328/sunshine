#### 1. 清理日志文件

```
sudo rm -rf /var/log/*
```



#### 2. 清理临时文件

```
sudo rm -rf /tmp/*
```



#### 3. 清理软件缓存

```
sudo apt-get clean
```



#### 4. 检查大文件

```
# 查找系统中的大文件
sudo find / -type f -size +100M
```



#### 5. 清理无用软件

```
sudo apt-get autoremove
```