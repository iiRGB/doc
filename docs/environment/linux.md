# Linux 相关记录

## 常用命令

- 端口

  ``` shell
  添加
  firewall-cmd --zone=public --add-port=80/tcp --permanent    （--permanent永久生效，没有此参数重启后失效）
  重新载入
firewall-cmd --reload
  查看
  firewall-cmd --zone= public --query-port=80/tcp
  删除
  firewall-cmd --zone= public --remove-port=80/tcp --permanent
  ```
  
  

## 常见问题



## 其他

