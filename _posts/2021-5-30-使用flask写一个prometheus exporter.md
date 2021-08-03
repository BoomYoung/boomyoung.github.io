---
title: prometheus-使用python自定义exporter开发实例
layout: article
tags: 
  - prometheus
    监控
    flask
mode: immersive
lang: zh-Hans
outhor: Boom Young
pageview: true
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---


> 一个需求，监控GPU机器的每个容器的gpu显存使用情况，并生成报表



### 调研 

由于是要采集每台GPU机器的数据，首先想到的是使用zabbix agent的方式，但是这个需要在每台机器部署数据采集脚本配置监控项，成本较高。

prometheus的exporter可以使用k8s集群部署，并且prometheus的查询方式更有利于做报表输出



所以，这个需求采用了自定义exporter方式开发，并使用k8s部署



# 一、数据采集

我们知道，查询gpu的显存使用量通常使用`nvidia-smi`命令

```shell
[root@server ~]# nvidia-smi
Tue Jun  8 16:58:32 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.87.00    Driver Version: 418.87.00    CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla P4            On   | 00000000:3B:00.0 Off |                   0* |
| N/A   72C    P0    51W /  75W |   2822MiB /  7611MiB |     97%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla P4            On   | 00000000:AF:00.0 Off |                   0* |
| N/A   46C    P0    32W /  75W |   1985MiB /  7611MiB |     26%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0     37412      C   ./****                                       835MiB |
|    0    221484      C   ./****                                      1977MiB |
|    1    219930      C   ./****                                      1975MiB |
+-----------------------------------------------------------------------------+
```

可以从末尾几行得到几个关键信息，PID和GPU Memory的对应关系，拿到了PID，我们就只需要找到PID对应的容器名就可以了。

linux系统有个cgroup这个概念，网上解释为**控制组，它提供了一套机制用于控制一组特定进程对资源的使用**，具体的有兴趣可以搜索一下。

在`/proc/PID/cgroup` 文件下存储着一些内核信息，而PID->CONTAINER ID的对应关系就藏在里面

```shell
[root@server ~]# cat /proc/37412/cgroup 
11:devices:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
10:perf_event:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
9:net_prio,net_cls:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
8:freezer:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
7:memory:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
6:blkio:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
5:cpuset:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
4:hugetlb:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
3:cpuacct,cpu:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
2:pids:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
1:name=systemd:/kubepods/burstable/podf69b516a-c807-11ea-b574-e8611f15cb84/5dde7ce79575a55a5cac3fa75d08db2988ca778982f3cd1aaa1be50500e89a3a
```

查看正在运行的容器，发现容器ID对应着cgroup文档以'/'为分隔最后一部分的字符串的前12位，即`awk -F'/' '{print $NF}' | cut -c 1-12`

![image](\assets\blog\prometheus_exporter_docker_ps.png)

那么问题就解决了，开始编码吧。


    
# 二、Coding



在看这一节之前最好先了解prometheus自定义exporter的规则，详见附录。



代码将分为两个主要部分，一个是主要数据采集，一个是http服务暴漏

### 1、项目目录规划

```
├── gpu-exporter
│   ├── app
│   │   ├── gpu_exporter.py
│   │   └── nvidia-smi-query.py
│   ├── Dockerfile
│   ├── .drone.yml
│   ├── README.md
│   └── requirements.txt
```

### 2、数据采集 nvidia-smi-query.py

```python
#!/usr/bin/env python3
# -*- encoding: utf-8 -*-
"""
@File    :   nvidia_smi_query.py
@Time    :   2021/6/19 14:30
@Author  :   boomyoung
@Version :   1.0
@Contact :   boomyoung@***.com
@Desc    :   
"""
import json
import logging
import os
import subprocess

import docker


class NvidiaSmiQuery(object):

    def __init__(self):
        self.log_to = logging.getLogger()

        self._QUERY_GPU = '/usr/bin/nvidia-smi --query-gpu="uuid,index" --format=csv,noheader'
        self._QUERY_APP = '/usr/bin/nvidia-smi --query-compute-apps="gpu_uuid,pid,used_memory" --format=csv,noheader'

    def queryGpu(self) -> dict:
        """
        获取gpu_uuid和gpu索引的对应关系
        :return:
        """
        status, output = subprocess.getstatusoutput(self._QUERY_GPU)
        # status, output = (0, 'GPU-9ac93f4d-726e-c371-f189-7ed073d96fad, 0\nGPU-0822505e-f54d-898a-66eb-16ec1a79ff45, 1')
        gpu_dict = {}
        if status == 0:
            gpu_list = output.split('\n')

            for i in gpu_list:
                try:
                    gpu_uuid = i.split(',')[0].strip()
                    gpu_index = i.split(',')[1].strip()
                except:
                    continue
                gpu_dict[gpu_uuid] = gpu_index
        return gpu_dict

    def getContainerId(self, pid: str) -> str:
        """
        通过pid获取容器id
        :param pid:
        :return:
        """
        cgroup_file = "/home/proc/{}/cgroup".format(pid)
        # 1:name=systemd:/kubepods/burstable/pod079552fe-c445-11eb-90a6-b8ca3af096f1/173939f128334165c78443e4045d3e7cee45795f3c7b3029c72b3388215bfe2e
        container_id = ''
        if os.path.exists(cgroup_file):
            status, output = subprocess.getstatusoutput(
                "/usr/bin/tail -n 1 %s | awk -F'/' '{print $NF}' | cut -d'-' -f 2 | cut -c 1-12" % cgroup_file)
            if status == 0:
                container_id = output.strip()
        return container_id

    def getContainerName(self, container_id: str) -> str:
        """
        容器内连接宿主机docker api,
        通过容器ID获取容器名
        :param container_id:
        :return:
        """
        try:
            client = docker.from_env(version='auto', timeout=5)
            container_name = client.containers.get(container_id).name
        except docker.errors.DockerException:
            try:
                client = docker.DockerClient(base_url="tcp://127.0.0.1:2375", version="auto", timeout=5)
                container_name = client.containers.get(container_id).name
            except:
                try:
                    client = docker.DockerClient(base_url="unix://var/run/docker.sock", version="auto", timeout=5)
                    container_name = client.containers.get(container_id).name
                except:
                    container_name = ''
        except Exception as e:
            # self.log_to.error('Port 2375 Connection refused! detail: {}'.format(e))
            container_name = ''
        return container_name

    def queryComputeApp(self) -> list:
        status, output = subprocess.getstatusoutput(self._QUERY_APP)

        data_list = []
        if status == 0:
            app_list = output.split('\n')

            for i in app_list:
                try:
                    gpu_uuid = i.split(',')[0].strip()
                    pid = i.split(',')[1].strip()
                    used_memory = i.split(',')[2].replace(' ', '')
                except:
                    continue

                data_dict = {
                    'pid': pid,
                    'used_memory': used_memory,
                    'gpu_uuid': gpu_uuid
                }
                data_list.append(data_dict)
        return data_list

    def getQueryResult(self) -> tuple:
        """
        返回查询结果
        pid: 进程id
        used_memory：已使用的显存
        gpu_index：gpu索引
        container_name：容器名
        container_id：容器ID
        :return:
        """

        status, _ = subprocess.getstatusoutput('/usr/bin/nvidia-smi')
        if status != 0:
            return 100, _

        gpu_list = []

        gpu_info = self.queryGpu()

        gpu_compute = self.queryComputeApp()
        if gpu_compute:
            for i in gpu_compute:
                gpu_uuid = i.get('gpu_uuid')
                pid = i.get('pid')
                used_memory = i.get('used_memory')

                gpu_index = gpu_info.get(gpu_uuid, '')

                container_id = self.getContainerId(pid)
                container_name = self.getContainerName(container_id)

                data_dict = {
                    'pid': pid,
                    'used_memory': used_memory,
                    'gpu_index': gpu_index,
                    'container_name': container_name,
                    'container_id': container_id
                }
                gpu_list.append(data_dict)
            return 0, gpu_list
        else:
            return 101, "nvidia-smi query fails."
            

```

- 有些GPU机器可能并未使用GPU，要对采值做一步错误处理
- /home/proc/$pid/cgroup 为宿主机的/proc/$pid/cgroup挂载的目录
- 注意：不是所有系统的cgroup文件的第一行能拿到容器ID，要做个错误判断



输出结果参照下方

```
root@25c1f5d73942:/gpu-exporter/app# ./nvidia-smi-query.py
GPU-f9529c59-52fd-bd67-2c29-76073a2c76be    0   159355  835MiB  a78fcbd9564f
GPU-f9529c59-52fd-bd67-2c29-76073a2c76be    0   220678  1933MiB d3719911f2cd
GPU-f9529c59-52fd-bd67-2c29-76073a2c76be    0   2482    835MiB  0f6a0e913c52
GPU-1f9ecad0-1b3e-4961-eb93-99a61bb12588    1   1445    1933MiB 76202fcac53c
root@25c1f5d73942:/gpu-exporter/app# 
```

- 第一列为gpu_uuid，主要用来join拼接
- 第二列gpu卡序列
- 第三列PID，为了寻找容器ID，后面用不到了
- 第四列显存使用量
- 第五列容器ID



一个疑问，为什么没有在这个脚本直接输出容器名？因为获取容器名需要调用docker命令，而在容器环境内就牵扯到一个问题，**在容器内部调用宿主机的docker命令行**，这个实现起来简单但是总是会有些意想不到问题。好在docker有个api这东西，我们可以通过tcp或sock连接，只需要将宿主机的2375端口打开或者挂载docker相关目录。

### 3、主程序 gpu_exporter.py

这个脚本主要用到了，`flask`、`prometheus_client`、`docker`等第三方库，用pip直接安装即可。

各个库的用法不在本文的介绍当中，直接贴代码：

```python
# !/usr/bin/env python3
# -*- encoding: utf-8 -*-
"""
@File    :   gpu-exporter.py
@Time    :   2021.01.15 15:32:40
@Author  :   boomyoung
@Version :   1.0
@Contact :   boomyoung@***.com
@Desc    :   Prometheus Exporter, Container GPU memory occupancy monitoring.
             Python, Shell, Flask, Prometheus, Docker API.
"""
import sys

import prometheus_client

from prometheus_client import Gauge
from prometheus_client.core import CollectorRegistry

from flask import Response, Flask

import logging
from logging.handlers import RotatingFileHandler

from nvidia_smi_query import NvidiaSmiQuery

app = Flask(__name__)
app.debug = True

used_gpu_memory = Gauge("used_gpu_memory", "GPU memory usage of container",
                        ["container_name", "gpu", "pid", "container_id", "unit"])

BYTE_UNITS = {
    'b': 1,
    'k': 1024,
    'm': 1024 * 1024,
    'g': 1024 * 1024 * 1024
}


def setupLogger(log_file, level=logging.INFO):
    log = logging.getLogger()
    formatter = logging.Formatter('%(asctime)s -- %(levelname)s -- %(message)s')
    log_file_handler = RotatingFileHandler(filename=log_file, maxBytes=1024 * 1024, backupCount=3)
    log_file_handler.setFormatter(formatter)
    log.setLevel(level)
    log.addHandler(log_file_handler)


@app.route("/metrics")
def index():
    # setupLogger('/var/log/gpu-exporter.log')
    log_to = logging.getLogger()

    unit = 'bits'

    status, result = NvidiaSmiQuery().getQueryResult()
    if status == 0:
        # log_to.info('nvidia-smi query SUCCESS!output: \n{}'.format(result))
        for i in result:
            pid = i.get('pid', '')
            gpu_mem_used = i.get('used_memory', '')
            gpu_index = i.get('gpu_index', '')
            container_name = i.get('container_name', '')
            container_id = i.get('container_id', '')
            if "KiB" in gpu_mem_used:
                gpu_mem_used = gpu_mem_used.strip('KiB')
                conversion = BYTE_UNITS.get('k')
            elif "MiB" in gpu_mem_used:
                gpu_mem_used = gpu_mem_used.strip('MiB')
                conversion = BYTE_UNITS.get('m')
            elif "GiB" in gpu_mem_used:
                gpu_mem_used = gpu_mem_used.strip('GiB')
                conversion = BYTE_UNITS.get('g')
            else:
                gpu_mem_used = 0
                conversion = BYTE_UNITS.get('b')

            used_gpu_memory.labels(container_name, gpu_index, pid, container_id, unit).set(
                float(gpu_mem_used) * conversion)

        return Response(prometheus_client.generate_latest(used_gpu_memory),
                        mimetype="text/plain")
    elif status == 101:
        # log_to.warning(result)
        used_gpu_memory.labels('None', 'None', 'None', 'None', unit).set(int(status))
        return Response(prometheus_client.generate_latest(used_gpu_memory),
                        mimetype="text/plain")
    else:
        # log_to.error(result)
        used_gpu_memory.labels('ERROR', 'ERROR', 'ERROR', 'ERROR', unit).set(int(status))
        return Response(prometheus_client.generate_latest(used_gpu_memory),
                        mimetype="text/plain")


if __name__ == "__main__":
    app.run(host="0.0.0.0", port="44110")

```

标准输出如下

```
# HELP used_gpu_memory GPU memory usage of container
# TYPE used_gpu_memory gauge
used_gpu_memory{container_id="76202fcac53c",container_name="k8s_dx-tel-gpu02-418_aiges-tel-418.iat-tel-gpu02-6b4bb64b87-8lbt8_iat_7aa11875-acd0-11eb-90a6-b8ca3af096f1_0",gpu="1",unit="bits"} 2.026897408e+09
used_gpu_memory{container_id="a78fcbd9564f",container_name="ivws-channels",gpu="0",unit="bits"} 8.7556096e+08
used_gpu_memory{container_id="d3719911f2cd",container_name="k8s_dx-tel-gpu01-418_aiges-tel-418.iat-tel-gpu01-bb7cf87b-szrdl_iat_9f24479b-c1b6-11eb-90a6-b8ca3af096f1_0",gpu="0",unit="bits"} 2.026897408e+09
used_gpu_memory{container_id="0f6a0e913c52",container_name="k8s_ivw-1014_ivws.ivws-66549f8f96-m2gkc_ivw_3a74970b-c801-11ea-b574-e8611f15cb84_65",gpu="0",unit="bits"} 8.7556096e+08
```



### 4、Dockerfile

```
# 使用自制的基础镜像
FROM hub.***.com/prometheus-exporter/ubuntu_flask:14.04_nvi418.87.00

# 将所有文件目录复制到容器内的/gpu-exporter下
COPY . /gpu-exporter

# 设置工作目录
WORKDIR /gpu-exporter

# 已在基础镜像内集成
# RUN sed -i 's#http://archive.ubuntu.com/#http://mirrors.tuna.tsinghua.edu.cn/#' /etc/apt/sources.list
# RUN apt-get update --fix-missing && apt-get install -y python-pip libltdl7 --fix-missing
# RUN python -m pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 执行主程序
CMD ["python3","/gpu-exporter/app/gpu_exporter.py"]
```



#### 4.1 自制基础镜像

最基础的镜像使用了公司私有HUB的ubuntu gpu虚拟化的镜像，但是该镜像系统版本为Ubuntu14，在Dockerfile中pip安装三方库遇到了很大的问题，于是在此基础上重新制作了一个镜像。

该镜像包含了：

- python3.4
- pip19.1.1
- python docker-api3.7.3
- Flask 1.0.4
- prometheus-client0.11.0



环境安装及打包过程如下：

https://www.yuque.com/docs/share/6ff6b1d4-e32f-4511-8822-4bce527ba679?# 《Ubuntu14 pip3安装模块遇到的坑》

```shell
[root@nodes ~]# docker export -o ubuntu_flask:14.04_nvi418.87.00.tar eafd1215128e
[root@nodes ~]# ls
anaconda-ks.cfg  dmidecode-3.0-2.el7.x86_64.rpm  huInsideTheContainer.sh       monitor.sql                                  psycopg2-2.8.6         ubuntu_flask:14.04_nvi418.87.00
a.sh             draw.io-x86_64-13.0.3.rpm       Lib_Utils-1.00-09.noarch.rpm  mysql57-community-release-el7-11.noarch.rpm  psycopg2-2.8.6.tar.gz  ubuntu_flask:14.04_nvi418.87.00.tar
cobbler.ks       grafana-6.7.3-1.x86_64.rpm      MegaCli-8.02.21-1.noarch.rpm  net_conf                                     requ.txt               yumbak
data2            hf-iflymonitor.sh               monitor-multi-rent            original-ks.cfg                              sys_init_hf.py
[root@nodes ~]# docker tag SOURCE_IMAGE[:TAG] hub.***.com/prometheus-exporter/IMAGE[:TAG]^C
[root@nodes ~]# docker import ubuntu_flask:14.04_nvi418.87.00.tar ubuntu_flask:14.04_nvi418.87.00
sha256:97ec1d2fdbfc33328ad0445ee786a9a03f80cfde956b988ee33721e79e73f486
[root@nodes ~]# docker images|grep ubuntu
ubuntu_flask                                                14.04_nvi418.87.00                         97ec1d2fdbfc        9 seconds ago       839MB
ubuntu                                                      xenial                                     9ff95a467e45        2 weeks ago         135MB
grafana/grafana                                             7.3.7-ubuntu                               e413e067c422        4 months ago        271MB
hub.***.com/aiaas/aiges_ubuntu                          14.04_nvi418.87.00                         7f212ddfe5df        19 months ago       860MB
[root@nodes ~]# ls^C
[root@nodes ~]# docker tag ubuntu_flask:14.04_nvi418.87.00 hub.***.com/prometheus-exporter/ubuntu_flask:14.04_nvi418.87.00
[root@nodes ~]# docker images|grep ubuntu
ubuntu_flask                                                14.04_nvi418.87.00                         97ec1d2fdbfc        53 seconds ago      839MB
hub.***.com/prometheus-exporter/ubuntu_flask            14.04_nvi418.87.00                         97ec1d2fdbfc        53 seconds ago      839MB
ubuntu                                                      xenial                                     9ff95a467e45        2 weeks ago         135MB
grafana/grafana                                             7.3.7-ubuntu                               e413e067c422        4 months ago        271MB
hub.***.com/aiaas/aiges_ubuntu                          14.04_nvi418.87.00                         7f212ddfe5df        19 months ago       860MB
[root@nodes ~]# docker push hub.***.com/prometheus-exporter/ubuntu_flask:14.04_nvi418.87.00
The push refers to repository [hub.***.com/prometheus-exporter/ubuntu_flask]
eedbb3fc1029: Pushed 
14.04_nvi418.87.00: digest: sha256:cf3a51ed9565eb79132ff852aca82b03ebb2f654d8879c8dc8c06093c61d4c2d size: 529
```

### 5、Drone CI流程配置

#### 5.1 GitLab+Drone+HUB打通

drone的配置参照之前的一篇文章

https://www.yuque.com/docs/share/ecf03783-08e0-4b44-868e-e425291e455b?# 《GitLab + Drone》

#### 5.2 .drone.yml

```yaml
matrix:
  IMAGE_REPO:
    - hub.***.com/prometheus-exporter/gpu-exporter # 定义了一个镜像存储路径的全局变量，可通过${IMAGE_REPO}引用
  RESGISTRY:
    - hub.***.com # 定义镜像仓库地址
  REPORT_EMAIL:
    - boomyoung@***.com # CI/CD报告发送的邮件地址，可通过${EMAIL}引用

clone:
  git:
    image: plugins/git
    depth: 50
    tags: true

# drone流水线描述
pipeline:

  build:
    image: plugins/docker
    registry: ${RESGISTRY}
    repo: ${IMAGE_REPO}
    secrets: [ docker_username, docker_password ]
    tags: # 每次构建两个镜像，一个标签为latest，另一个标签为git提交的tag
      - latest
      - ${DRONE_TAG=latest}
    dockerfile: Dockerfile
    insecure: true # 启用对此 registry 的不安全通信
    when:
      event: [ tag ]
```



# 三、部署

项目git tag提交后会生成镜像

```shell
docker pull hub.iflytek.com/prometheus-exporter/gpu-exporter:latest
```

### 3.1 docker启动

```shell
docker run -itd -p 9900:9900 -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock -v /proc:/home/proc --device=/dev/nvidia0:/dev/nvidia0 --device=/dev/nvidia1:/dev/nvidia1 --device=/dev/nvidiactl:/dev/nvidiactl --device=/dev/nvidia-uvm:/dev/nvidia-uvm --device=/dev/nvidia-uvm-tools:/dev/nvidia-uvm-tools --privileged=true hub.***.com/prometheus-exporter/gpu-exporter:latest
```

### 3.2 k8s方式部署

待续。。。



# 四、验证

```shell
[root@hu-10 ~]# curl "http://127.0.0.1:9900/metrics"
# HELP used_gpu_memory GPU memory usage of container
# TYPE used_gpu_memory gauge
used_gpu_memory{container_id="5336e795c3f2",container_name="k8s_s99d61854_s99d61854.s99d61854-546dc767b6-m86fj_default_c400beb5-5a23-11eb-9cb1-6c92bfcf13f0_0",gpu="0",unit="bits"} 6.74234368e+08
used_gpu_memory{container_id="59686a0772a8",container_name="k8s_s2bda39d4_s2bda39d4.s2bda39d4-f47c47bc-52md8_default_10c8c9ed-b6f1-11eb-aede-6c92bfcf135e_0",gpu="1",unit="bits"} 7.72800512e+08
used_gpu_memory{container_id="656eaa94f758",container_name="k8s_sa4c9e944_sa4c9e944.sa4c9e944-65596dff5c-t5sd4_default_89acaeac-b493-11eb-aede-6c92bfcf135e_0",gpu="0",unit="bits"} 1.223688192e+09
used_gpu_memory{container_id="4240e9bc91f0",container_name="k8s_s5982833f_s5982833f.s5982833f-74f5464dbd-wfk7m_default_0ed35275-bf67-11eb-aede-6c92bfcf135e_0",gpu="0",unit="bits"} 7.72800512e+08
```



# 五、接入prometheus

待续。。。





# 附录：

### 1、自定义prometheus exporter

prometheus所能识别的metrics数据格式是固定的，要包含HELP和TYPE，HELP是指标的注释信息，TYPE必须带有指标的类型

标准metrics指标类型有：

- Counter (累积)
  Counter一般表示一个单调递增的计数器。
- Gauge (测量)
  Gauge一般用于表示可能动态变化的单个值，可能增大也可能减小。
- Histogram (直方图)
- Summary (概略图)

常用的就Counter、Gauge，下面举一个标准的metrics数据

```shell
# HELP test_key This is a test KEY
# TYPE test_key gauge
test_key{tag1="a",tag2='aa'} 111
test_key{tag1="b",tag2='bb'} 123
test_key{tag1="c"mtag='sd'} 344
```

无论是采用什么语言编写exporter，最终只要打印如上的数据格式即可