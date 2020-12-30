# heketi 管理glusterfs集群 自动扩缩容

## 架构示例图
[![](https://github.com/justmeandopensource/glusterfs/blob/master/glusterfs-heketi-demo/resources/gluster-heketi.png?raw=true)](https://github.com/justmeandopensource/glusterfs/blob/master/glusterfs-heketi-demo/resources/gluster-heketi.png?raw=true)

## 集群演示

|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Heketi Node|heketi.example.com|172.16.16.200|Centos 7|16G|4|
|Gluster Node 1|gluster-1.example.com|172.16.16.201|Centos 7|16G|4|
|Gluster Node 2|gluster-2.example.com|172.16.16.202|Centos 7|16G|4|

### glusterfs集群节点上安装glusterfs 服务
'''
{
	yum install glusterfs-server -y
	systemctl enable guslterfsd
	systemctls start glusterfsd
}
'''

## Hekeit服务设置（在heketi节点上)
### 下载heketi二进制文件
'''

{
  	cd /tmp
  	wget https://github.com/heketi/heketi/releases/download/v10.1.0/heketi-v10.1.0.linux.amd64.tar.gz
  	tar zxf heketi*
  	mv heketi/{heketi,heketi-cli} /usr/local/bin/
}

'''

### 设置heketi专用账号
```
{
  groupadd -r heketi
  useradd -r -s /sbin/nologin -g heketi heketi
  mkdir {/var/lib,/etc,/var/log}/heketi
}
```
### 设置免密登陆到glusterfs节点
```
{
  ssh-keygen -f /etc/heketi/heketi_key -t rsa -N ''
  for node in gluster-1 gluster-2; do
    ssh-copy-id -i /etc/heketi/heketi_key.pub root@$node
  done
}
```

#### 配置heketi服务
```
cp /tmp/heketi/heketi.json /etc/heketi/
```
Edit **/etc/heketi/heketi.json**, change executor to ssh and update sshexec options as shown below
```
	"executor": "ssh", 

	"_sshexec_comment": "SSH username and private key file information",
	"sshexec": {
  	  "keyfile": "/etc/heketi/heketi_key", 
  	  "user": "root", 
  	  "port": "22", 
  	  "fstab": "/etc/fstab" 
	},
```
### 创建heketi系统启动服务
```
cat <<EOF >/etc/systemd/system/heketi.service
[Unit]
Description=Heketi Server

[Service]
Type=simple
WorkingDirectory=/var/lib/heketi
EnvironmentFile=-/etc/heketi/heketi.env
User=heketi
ExecStart=/usr/local/bin/heketi --config=/etc/heketi/heketi.json
Restart=on-failure
StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
EOF
```
#### Enable and start heketi service
```
{
  systemctl daemon-reload
  systemctl enable --now heketi
}
```
### 验证heketi服务是否正常
```
curl localhost:8080/hello; echo
```
### 为heketi-cli 设置环境变量
```
export HEKETI_CLI_USER=admin
export HEKETI_CLI_KEY=secretpassword
```
### 使用heketi-cli 创建jiqun
```
	heketi-cli cluster create
```

### 添加节点到集群
```
heketi-cli node add --zone 1 --cluster e358bbfd4bae42ec06988c22158d6e31  --management-host-name osd1 --storage-host-name 172.16.16.201
```

```
heketi-cli node add --zone 2 --cluster e358bbfd4bae42ec06988c22158d6e31  --management-host-name osd2 --storage-host-name 172.16.16.202
```
###  为节点添加存储盘
```
heketi-cli device add --name /dev/sdc --node 节点id
```

