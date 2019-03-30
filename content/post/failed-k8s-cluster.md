---
title: "一次失败的k8s部署经历"
date: 2019-03-30T23:04:44+08:00
lastmod: 2019-03-30T23:04:44+08:00
draft: false
keywords: ["k8s"]
description: ""
tags: ["k8s"]
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

这篇文章将会完全是混乱的结构，混乱的记录。

昨天下班想，这个周末要把拖了很久的自己建一个k8s集群这件事给办了，结果没想到，终究还是只做了一半。


### 一、 部件

* vagrant
* virtualbox
* docker
* kubeadm
* ansible

### 二、 参照来源

主要步骤参考这篇文章： [Kubernetes Setup Using Ansible and Vagrant](https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/)

这次部署只是照着这篇文章`step by step`做的，只是一些国内特殊环境需要改变的地方做了一些改动，无奈还是失败了，严重怀疑智商。

### 三、 第一次尝试

刚开始想利用代理做，有现成的`shadowsocks`代理，直接开个代理给`k8s`用是不是就行了。

#### 3.1 本地`sock5`代理

1. 安装

> pip install shadowsocks

2. 编辑

> vim /etc/shadowsocks.json

```json
{
  "server": "proxy ip",
  "server_port": "port",
  "local_address": "127.0.0.1",
  "local_port": "1080",
  "password": "pass",
  "method": "aes-256-cfb",
  "timeout": "300",
  "workers": "1"
}
```

3. 启动：

> `sslocal -c /etc/shadowsocks.json start`

#### 3.2 sock5代理转http代理

sock5代理是不能直接给终端用的，需要转换为`http`代理，这里用`prioxy`

1. 安装

> brew install privoxy

2. 编辑 

> vim /usr/local/etc/privoxy/config

最后两行添加:

> listen-address  0.0.0.0:8118  # 要代理给局域网内其他终端使用，所以要绑定0.0.0.0
> forward-socks5 / 127.0.0.1:1080 .

3. 启动

> /usr/local/Cellar/privoxy/3.0.26/sbin/privoxy /usr/local/etc/privoxy/config

3.3 vagrant设置

为了比较方便的设置虚拟机里面的代理，用了vagrant插件

安装

> vagrant plugin install vagrant-proxyconf

#### 3.3 vagrant配置文件

* 编辑`Vagrantfile`

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :
#
IMAGE_NAME = "bento/ubuntu-16.04"
N = 2
Vagrant.configure("2") do |config|
    config.ssh.insert_key = false
    if Vagrant.has_plugin?("vagrant-proxyconf")
  	  config.proxy.http     = "http://192.168.0.107:8118/"  # 本机代理ip:端口
  	  config.proxy.https    = "http://192.168.0.107:8118/"  
  	  config.proxy.no_proxy = "0,1,2,3,4,5,6,7,8,9,10.0.0.0/8,https://192.168.50.10,https://192.168.50.12, https://192.168.50.11" # k8s内部通信不能使用代理
  	end

    config.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 2
    end
      
    config.vm.define "k8s-master" do |master|
        master.vm.box = IMAGE_NAME
        master.vm.network "private_network", ip: "192.168.50.10"
        master.vm.hostname = "k8s-master"
        master.vm.provision :shell, inline: $bootstrap
        master.vm.provision "ansible" do |ansible|
            ansible.playbook = "kubernetes-setup/master-playbook.yml"
        end
		master.vm.synced_folder ".", "/vagrant", create: true, owner: "root", group: "root", mount_options: ["dmode=777","fmode=777"]
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
            node.vm.hostname = "node-#{i}"
            node.vm.provision :shell, inline: $bootstrap
            node.vm.provision "ansible" do |ansible|
                ansible.playbook = "kubernetes-setup/node-playbook.yml"
            end
			node.vm.synced_folder ".", "/vagrant", create: true, owner: "root", group: "root", mount_options: ["dmode=777","fmode=777"]
        end
    end
end
```

#### 小结

第一次主要尝试就是这些了，结果是失败，因为一个未知名的错误，Google提示man in middle之类的，当时是凌晨四点，就放弃去睡了。

### 四、第二次尝试

第一次尝试时候，都在报拉取`k8s.gcr.io`这个registry的镜像时候失败的，那么就换种方式，从其他源拉取镜像，然后打上一个这样的标签,下面是抄的一份脚本:

```bash
images=(
    kube-apiserver:v1.14.0
    kube-controller-manager:v1.14.0
    kube-scheduler:v1.14.0
    kube-proxy:v1.14.0
    pause:3.1
    etcd:3.3.10
    coredns:1.3.1
)


for imageName in ${images[@]} ; do
    docker pull registry.aliyuncs.com/google_containers/$imageName
    docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
done
```

这样就不用代理了，删掉`vagrant`里面的代理配置。

第二次尝试主要就是这个了。

结果集群成功拉起来了，但是`calico-node`和`calico-controller`pod `ERROR`折腾了好久没有头绪，好像是calico访问etcd时候出错，（再次严重怀疑智商）, 先记录下来，我要睡觉。
