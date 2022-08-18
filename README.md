docke安装trojan-go

前提要求：  
    是否安装docker     
    是否安装docker-compose  
    检查域名解析是否正常
准备：
    需要提前准备  
    ssl证书文件    //用于https请求  
    config.json 文件   //用于trojan-go的启动  
    

复制 docker-compose.yaml 文件上传到服务器,在docker-compose.yaml文件所在目录执行如下命令即可   
```
docker-compose up -d
```

顺便附上模板文件 需要修改里面内容  
    config.json  //服务的启动配置  
    trojan-go.yml    //客户端连接配置  
    

