# Windows错误排查步骤

## 常用命令

### 端口监听查看

```shell
# 列出所有端口的情况
netstat -ano 

# 查看某个端口占用情况
tasklist -ano | findstr "2333"
```



### 通过PID强制关闭程序

```
taskkill
```





