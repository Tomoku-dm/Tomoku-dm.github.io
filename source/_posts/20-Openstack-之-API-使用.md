---
title: Openstack 之 API 使用
date: 2019-06-10 20:53:25
tags:
 - python
 - openstack
---
## 前言
本篇文章主要介绍 Openstack  API 的使用，在此之前我们先简单介绍一下 `Openstack` 。

`Openstack` 是一个搭建云平台的一个 IAAS 解决方案，可以看成就是一个软件。openstack能干什么呢，可以搭建公有云，私有云，企业云。

`Openstack`组件包括：

- **Compute(代号为“Nova”)**
  - 这个是最核心的，Nova最开始的时候，可以说是一套虚拟化管理程序，还可以管理网络和存储。
- **Identity(代号为“Keystone”)**
  - 这是提供身份认证和授权的组件。任何系统，身份认证和授权，其实都比较复杂。尤其Openstack 那么庞大的项目，每个组件都需要使用统一认证和授权。
- **Dashboard(代号为“Horizon”)**
  - 为所有OpenStack的服务提供了一个模块化的web-based用户界面。使用这个Web GUI，可以在云上完成大多数的操作，如启动实例，分配IP地址，设置访问控制等。
- **Image Service(代号为“Glance”)**
  - 镜像管理。目前Glance的镜像存储，支持本地存储，NFS，swift，sheepdog和Ceph，基本是够用了。
    目前Glance的最大需求就是多个数据中心的镜像管理，如何复制，不过这个功能已经基本实现。还有就是租户私有的image管理，这些目前功能都已经实现。
- **Network(代号为“Quantum”)**
  - 这是网络管理的组件，也是重头戏，Openstack的未来，基本都要靠quantum。Quantum 后端可以是商业产品或者开源。开源产品支持Openvswitch，和linux bridge。网络设备厂商都在积极参与，让他们的产品支持Quantum。
- **Object Storage(代号为“Swift”)**
  - 对象存储组件，不是存储虚拟机的。
- **Block Storage(代号为“Cinder”)**
  - 这是存储管理的组件。Cinder存储管理主要是指虚拟机的存储管理。



## API 使用

### 一、获取 TOKEN 权限

- 在使用各组件 API 之前，必须通过认证拿到token。从 token 的结果中可以看到各组件 API 的 ip 入口。

- ```php
  $ curl -X POST -H "Content-Type: application/json"  -d 
  '{
      "auth":{
               "passwordCredentials":
               {
                  "username": "admin",
                  "password":"xxxxxxxxx"
                },
       "tenantName":"admin"            
         }
  }' "http://172.20.31.10:35357/v2.0/tokens"
  
  {
      "access": {
          "metadata": {
              "is_admin": 0,
              "roles": [
                  "3fc758f9421249deb277db9fe1521b30",
                  "e4bbc640275b41869a0b3e6c6316b61f"
              ]
          },
          "serviceCatalog": [
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8780",
                          "id": "36568a30d4b9451eaa2af8a0be3a01d0",
                          "internalURL": "http://172.20.31.10:8780",
                          "publicURL": "http://172.20.31.10:8780",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "placement",
                  "type": "placement"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:9696",
                          "id": "4acd2d071f7149cb9da46a6ace3b40db",
                          "internalURL": "http://172.20.31.10:9696",
                          "publicURL": "http://172.20.31.10:9696",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "neutron",
                  "type": "network"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8776/v2/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "id": "363a810c9bfe4381bf4a6daeedced071",
                          "internalURL": "http://172.20.31.10:8776/v2/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "publicURL": "http://172.20.31.10:8776/v2/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "cinderv2",
                  "type": "volumev2"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8776/v3/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "id": "9f13a94e8ff4484795c1f87dfe08c0d1",
                          "internalURL": "http://172.20.31.10:8776/v3/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "publicURL": "http://172.20.31.10:8776/v3/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "cinderv3",
                  "type": "volumev3"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8042",
                          "id": "d30e0cd250794c089954322d7592ce20",
                          "internalURL": "http://172.20.31.10:8042",
                          "publicURL": "http://172.20.31.10:8042",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "aodh",
                  "type": "alarming"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8041",
                          "id": "12e3c6d69c7c45d5a5f2989c9516142b",
                          "internalURL": "http://172.20.31.10:8041",
                          "publicURL": "http://172.20.31.10:8041",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "gnocchi",
                  "type": "metric"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8774/v2/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "id": "034b40f84cf84f3eb1fbf0f8b24e5e16",
                          "internalURL": "http://172.20.31.10:8774/v2/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "publicURL": "http://172.20.31.10:8774/v2/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "nova_legacy",
                  "type": "compute_legacy"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8000/v1",
                          "id": "14dbbc32cdec4cc780bf7ec5c49ce255",
                          "internalURL": "http://172.20.31.10:8000/v1",
                          "publicURL": "http://172.20.31.10:8000/v1",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "heat-cfn",
                  "type": "cloudformation"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8777",
                          "id": "564d882201004adab6cc9139320417b7",
                          "internalURL": "http://172.20.31.10:8777",
                          "publicURL": "http://172.20.31.10:8777",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "ceilometer",
                  "type": "metering"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8776/v1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "id": "7bc2ab9c836e42d7825b32ec1dd947a2",
                          "internalURL": "http://172.20.31.10:8776/v1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "publicURL": "http://172.20.31.10:8776/v1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "cinder",
                  "type": "volume"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8004/v1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "id": "11f6ba6343e342538a066fb8e2afed0c",
                          "internalURL": "http://172.20.31.10:8004/v1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "publicURL": "http://172.20.31.10:8004/v1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "heat",
                  "type": "orchestration"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8774/v2.1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "id": "7019da3f53094e298f99c992dfd9cfc7",
                          "internalURL": "http://172.20.31.10:8774/v2.1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "publicURL": "http://172.20.31.10:8774/v2.1/1bdf602d2e9d484fa8e645eeb2f7faa4",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "nova",
                  "type": "compute"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:9292",
                          "id": "23742c0e49bd4702a68436b7b9a3d124",
                          "internalURL": "http://172.20.31.10:9292",
                          "publicURL": "http://172.20.31.10:9292",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "glance",
                  "type": "image"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:8977",
                          "id": "2323e2f4367c4a21b31e0063f8d0c7f5",
                          "internalURL": "http://172.20.31.10:8977",
                          "publicURL": "http://172.20.31.10:8977",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "panko",
                  "type": "event"
              },
              {
                  "endpoints": [
                      {
                          "adminURL": "http://172.20.31.10:35357/v3",
                          "id": "6f51ba03f943453e893e93f1f4fb9ad3",
                          "internalURL": "http://172.20.31.10:5000/v3",
                          "publicURL": "http://172.20.31.10:5000/v3",
                          "region": "RegionOne"
                      }
                  ],
                  "endpoints_links": [],
                  "name": "keystone",
                  "type": "identity"
              }
          ],
          "token": {
              "audit_ids": [
                  "dTqX8utbQVKjekNznu5HSA"
              ],
              "expires": "2019-06-18T00:47:30.000000Z",
              "id": "gAAAAABc_aiiuPpu6EelJCHVsEYjYXd1qy-jzOdXxjQ9CuqoZO_nd7bPjLTAow_0Sl7xkJC6kFHYiTGC9GLLtudZTunXtanTxRUGUrUF8VvQ7Cpi465GtYvOM-_yQYOPG8xgA5tyVIDRKz0V7W6tDi1-Ph07Dkj1HJ08iQxxxxxxxxxxxxxx",
              "issued_at": "2019-06-10T00:47:30.000000Z",
              "tenant": {
                  "description": "Bootstrap project for initializing the cloud.",
                  "enabled": true,
                  "id": "1bdf602d2e9d484fa8e645eeb2f7faa4",
                  "name": "admin"
              }
          },
          "user": {
              "id": "0d858f861c3140a5a31078eb0cbe6def",
              "name": "admin",
              "roles": [
                  {
                      "name": "heat_stack_owner"
                  },
                  {
                      "name": "admin"
                  }
              ],
              "roles_links": [],
              "username": "admin"
          }
      }
  }
  ```

### 二、nova 组件API 

- 获取所有虚拟机(server)，注意：这里使用的是 admin 项目获取的 token，所以要看到所有项目的虚拟机，需要加 all_tenants=true，下面我只需要特定项目的所有机器，所以使用 project_id=xxx。

- ```php
  $ curl -X GET -H "X-Auth-Token: gAAAAABc9i2PAdWo-4MWE-kcVPDazv8rNLBxBB9UK8-X6kXA3mi8PLDm10cQpI-Fr7Yo1F-J2NiPTEAWcEKQvtDO2_MPghr0cvblaDuVifuRpfqYqKG_SXE3E5m_zUqU8KhjfiQOmaKUJq2tx4JpuDqcsoK0viAxxxxxx" "http://172.20.31.10:8774/v2.1/servers/detail?all_tenants=true&project_id=4bf9b1cc0cb748a88b8526655734da2c" -s | python -m json.tool
  
  {
      "servers": [
          {
              "OS-DCF:diskConfig": "MANUAL",
              "OS-EXT-AZ:availability_zone": "nova",
              "OS-EXT-SRV-ATTR:host": "rg5-ostack005.cloud.hfb3.iflytek.net",
              "OS-EXT-SRV-ATTR:hypervisor_hostname": "rg5-ostack005.cloud.hfb3.iflytek.net",
              "OS-EXT-SRV-ATTR:instance_name": "instance-00001570",
              "OS-EXT-STS:power_state": 1,
              "OS-EXT-STS:task_state": null,
              "OS-EXT-STS:vm_state": "active",
              "OS-SRV-USG:launched_at": "2019-06-05T13:08:53.000000",
              "OS-SRV-USG:terminated_at": null,
              "accessIPv4": "",
              "accessIPv6": "",
              "addresses": {
                  "Classical_Network_2": [
                      {
                          "OS-EXT-IPS-MAC:mac_addr": "fa:16:3e:a9:51:c1",
                          "OS-EXT-IPS:type": "fixed",
                          "addr": "172.20.35.27",
                          "version": 4
                      }
                  ]
              },
              "config_drive": "True",
              "created": "2019-06-05T13:08:44Z",
              "flavor": {
                  "id": "f7e1c2a5-167c-4e1d-8de8-0a0034b8796c",
                  "links": [
                      {
                          "href": "http://172.20.31.10:8774/flavors/f7e1c2a5-167c-4e1d-8de8-0a0034b8796c",
                          "rel": "bookmark"
                      }
                  ]
              },
              "hostId": "59cc48a230b88736c992262b7b481c5fd3d01dce2ea2048ca60a6416",
              "id": "a48a86e7-547b-4f3c-be5f-9c9bf29e2034",
              "image": {
                  "id": "b1c6d306-0eee-433d-b7f3-3d7946033eff",
                  "links": [
                      {
                          "href": "http://172.20.31.10:8774/images/b1c6d306-0eee-433d-b7f3-3d7946033eff",
                          "rel": "bookmark"
                      }
                  ]
              },
              "key_name": null,
              "links": [
                  {
                      "href": "http://172.20.31.10:8774/v2.1/servers/a48a86e7-547b-4f3c-be5f-9c9bf29e2034",
                      "rel": "self"
                  },
                  {
                      "href": "http://172.20.31.10:8774/servers/a48a86e7-547b-4f3c-be5f-9c9bf29e2034",
                      "rel": "bookmark"
                  }
              ],
              "metadata": {},
              "name": "2019.6.5-No.01",
              "os-extended-volumes:volumes_attached": [
                  {
                      "id": "8eda595f-f83b-48f0-9aef-5c327ba489f2"
                  },
                  {
                      "id": "7a1632ae-f1ae-406c-a5a7-ccd5de377256"
                  },
                  {
                      "id": "7261e9e4-1781-4263-9a97-f7d86330ff1c"
                  }
              ],
              "progress": 0,
              "security_groups": [
                  {
                      "name": "default"
                  }
              ],
              "status": "ACTIVE",
              "tenant_id": "4bf9b1cc0cb748a88b8526655734da2c",
              "updated": "2019-06-05T13:08:53Z",
              "user_id": "49883c026fdf4cfeb5c660393f996973"
          },
          ...
  ```



- 获取所有物理机信息(hypervisors）

- ```php
  $ curl -X GET -H "X-Auth-Token: gAAAAABc9i2PAdWo-4MWE-kcVPDazv8rNLBxBB9UK8-X6kXA3mi8PLDm10cQpI-Fr7Yo1F-J2NiPTEAWcEKQvtDO2_MPghr0cvblaDuVifuRpfqYqKG_SXE3E5m_zUqU8KhjfiQOmaKUJq2tx4JpuDqcsoK0viAB5mxxxxx"  "http://172.20.31.10:8774/v2.1/os-hypervisors" -s | python -m json.tool
  
  {
      "hypervisors": [
          {
              "hypervisor_hostname": "rg5-ostack003.cloud.hfb3.iflytek.net",
              "id": 4,
              "state": "up",
              "status": "enabled"
          },
          {
              "hypervisor_hostname": "rg5-ostack012.cloud.hfb3.iflytek.net",
              "id": 7,
              "state": "up",
              "status": "enabled"
          },
          ...
      ]
  }
  
  ```



### 三、Keystone 组件 API

- 获取所有项目

- ```php
  $ curl -X GET -H "X-Auth-Token: gAAAAABc9i2PAdWo-4MWE-kcVPDazv8rNLBxBB9UK8-X6kXA3mi8PLDm10cQpI-Fr7Yo1F-J2NiPTEAWcEKQvtDO2_MPghr0cvblaDuVifuRpfqYqKG_SXE3E5m_zUqU8KhjfiQOmaKUJq2tx4JpuDqcsoK0viAB5mi3XrcXxxxxxxxxx"  "http://172.20.31.10:5000/v3/projects" -s | python -m json.tool 
  {
      "links": {
          "next": null,
          "previous": null,
          "self": "http://172.20.31.10:5000/v3/projects"
      },
          {
              "description": "Bootstrap project for initializing the cloud.",
              "domain_id": "default",
              "enabled": true,
              "id": "1bdf602d2e9d484fa8e645eeb2f7faa4",
              "is_domain": false,
              "links": {
                  "self": "http://172.20.31.10:5000/v3/projects/1bdf602d2e9d484fa8e645eeb2f7faa4"
              },
              "name": "admin",
              "parent_id": "default"
          },
          {
              "description": "",
              "domain_id": "default",
              "enabled": true,
              "id": "4bf9b1cc0cb748a88b8526655734da2c",
              "is_domain": false,
              "links": {
                  "self": "http://172.20.31.10:5000/v3/projects/4bf9b1cc0cb748a88b8526655734da2c"
              },
              "name": "zazhao",
              "parent_id": "default"
          },
          ...
         
      ]
  }
  ```

  

- 获取所有用户

- ```php
  $ curl -X GET -H "X-Auth-Token: gAAAAABc9i2PAdWo-4MWE-kcVPDazv8rNLBxBB9UK8-X6kXA3mi8PLDm10cQpI-Fr7Yo1F-J2NiPTEAWcEKQvtDO2_MPghr0cvblaDuVifuRpfqYqKG_SXE3E5m_zUqU8KhjfiQOmaKUJq2tx4JpuDqcsoK0viAB5mi3Xrxxxxxxxxxxxx"  "http://172.20.31.10:5000/v3/users" -s | python -m json.tool
  {
      "links": {
          "next": null,
          "previous": null,
          "self": "http://172.20.31.10:5000/v3/users"
      },
      "users": [
          {
              "description": "",
              "domain_id": "default",
              "email": "",
              "enabled": true,
              "id": "003f4fc6c5264b53ae04bd9142800ebd",
              "links": {
                  "self": "http://172.20.31.10:5000/v3/users/003f4fc6c5264b53ae04bd9142800ebd"
              },
              "name": "xiangxia",
              "options": {},
              "password_expires_at": null
          },
          {
              "description": "",
              "domain_id": "default",
              "email": "",
              "enabled": true,
              "id": "016745db50bd45bb9c390362dc998fee",
              "links": {
                  "self": "http://172.20.31.10:5000/v3/users/016745db50bd45bb9c390362dc998fee"
              },
              "name": "xqxie3",
              "options": {},
              "password_expires_at": null
          },
  ...
  ```

  

## Python 处理 API 输出

通过 API 拿到 json 格式的信息后，我们就可以使用python 来处理格式化了，python 处理 json 文件就是处理字典。下面简单写个例子：

- 更新 token 文件，通过 shell 脚本实现

- ```php
  $ curl -X POST  http://172.20.31.10:35357/v2.0/tokens -H "Content-Type: application/json"   -d '{"auth":{"passwordCredentials":{"username": "admin","password":"xxxxxxxxxx"},"tenantName":"admin"}}' | python -m json.tool  > token-tmp.txt
  
  ```





- 使用上述 shell 脚本 更新token，并更新 获取的server 和 user 信息，

- ```python
  import subprocess
  import json
  
  ## get token
  cmd_token = "/usr/bin/sh /home/dlp/mudu/scripts/openstack/curl-api/get-token.sh"
  sub_token = subprocess.Popen(cmd_token, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
  with open("/home/dlp/mudu/scripts/openstack/curl-api/token-tmp.txt",'r') as token_f:
          token_temp = json.loads(token_f.read())
  for i in range(len(token_temp)):
          token = token_temp['access']['token']['id']
  #       print token
  
  ## update server and user file
  
  cmd_uuser = "curl -X GET -H \"X-Auth-Token: " + token + "\"  \"http://172.20.31.10:5000/v3/users\" -s | python -m json.tool > /home/dlp/mudu/scripts/openstack/curl-api/user-source "
  cmd_userver = "curl -X GET -H \"X-Auth-Token: " + token + "\"  \"http://172.20.31.10:8774/v2.1/servers/detail?all_tenants=true&project_id=4bf9b1cc0cb748a88b8526655734da2c\" -s | python -m json.tool > /home/dlp/mudu/scripts/openstack/curl-api/server-source "
  
  sub_uuser = subprocess.Popen(cmd_uuser, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
  sub_userver = subprocess.Popen(cmd_userver, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
  
  
  
  ```



- 格式化 获取到的信息，将 用户 和 虚拟机 对应输出。

- ```python
  import json
  import collections
  import subprocess
  servers_dict = {}
  
  ## get servers and user_id
  with open("/home/dlp/mudu/scripts/openstack/curl-api/server-source",'r') as server_f:
  	server_temp = json.loads(server_f.read())
  for i in range(len(server_temp['servers'])):
  #	print i
  	addrtemp = server_temp['servers'][i]['addresses'].values()
  	servers_dict[i] = {"ip":addrtemp[0][0]['addr'].encode('utf-8'),"user_id":server_temp['servers'][i]['user_id'].encode('utf-8')}
  #	print addtemp[0][0]['addr']
  #	print temp['servers'][i]['user_id']
  	#print servers_dict[0]
  
  
  ## get users and  set group
  with open("/home/dlp/mudu/scripts/openstack/curl-api/user-source",'r') as user_f:
          user_temp = json.loads(user_f.read())
  for i in range(len(user_temp['users'])):
  	#print user_temp['users'][i]['name']
  	#print user_temp['users'][i]['id']
  	for j in servers_dict:
  		#print servers_dict[j]['user_id']
  		if user_temp['users'][i]['id'] == servers_dict[j]['user_id']:
  			servers_dict[j]["username"] = user_temp['users'][i]['name'].encode('utf-8')
  			cmd = "id " + user_temp['users'][i]['name'].encode('utf-8') + "  | awk '{print $2}' | awk '{match($0,/\([^()]*\)/);print substr($0,RSTART+1,RLENGTH-2)}'"
  			sub = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
  			servers_dict[j]["group"] = sub.stdout.read().strip('\n')
  			print servers_dict[j]
  
  ```



## 参考

- [Openstack API](https://developer.openstack.org/api-ref/compute/?expanded=add-associate-floating-ip-addfloatingip-action-deprecated-detail#create-server)
- [获取token 通过 python](https://blog.csdn.net/nirendao/article/details/54977717)


