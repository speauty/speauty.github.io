# 运维新手 {docsify-ignore}

## Jmeter

?> 当前环境: Linux BUG-MS-7C52 5.15.0-39-generic #42-Ubuntu SMP Thu Jun 9 23:42:32 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

### Java配置
```shell
# 安装Java
sudo apt install openjdk-8-jre-headless
# 调整参数 注释assistive_technologies
sudo gedit /etc/java-8-openjdk/accessibility.properties
```

### 安装Jmeter
```shell
# 下载Jmeter
wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.5.zip
unzip apache-jmeter-5.5.zip
mv apache-jmeter-5.5 /usr/local/jmeter
sudo ln -s /usr/local/jmeter/bin/jmeter /usr/bin/jmeter
``` 
