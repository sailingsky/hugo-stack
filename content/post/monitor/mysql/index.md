---
title: "prometheues监控mysql"
description: 
date: 2024-01-15T18:46:18+08:00
image: Grafana-0.png
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - prometheus
tags:
    - 监控
---
前提装好mysql服务，还有配套的prometheus以及grafana服务。
1. 下载mysqld_exporter
	官方下载地址：https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.windows-amd64.zip
2.  在mysql中添加mysqld_exporter的账号
	``` sql
		CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'exporter';
		
		GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
		
		ALTER USER exporter@localhost IDENTIFIED WITH mysql_native_password BY 'exporter';
		
		flush privileges;
	```
3. 在mysqld-exporter目录下新建`my.cnf`,加入配置
   ``` conf
	[client]
	user=exporter
	password=exporter
    ```
4. 启动mysqld_exporter:
	``` bash
		mysqld_exporter.exe --config.my-cnf=my.cnf
	```
	如图：
	
	![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151829914.png)

5. prometheus中的`prometheus.yml`增加配置：
	``` yaml
	  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
     - job_name: "mysql"

       # metrics_path defaults to '/metrics'
       # scheme defaults to 'http'.

       static_configs:
         - targets: ["localhost:9104"]  
      
	```
6. 启动prometheus，在页面上可看到新加的mysql
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151834586.png)

1. grafana配置好prometheus数据源，导入dashboard,我这用的这个dashboard. https://grafana.com/grafana/dashboards/17320-1-mysqld-exporter-dashboard/
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151840843.png)

1. 最终效果.
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401151841113.png)
