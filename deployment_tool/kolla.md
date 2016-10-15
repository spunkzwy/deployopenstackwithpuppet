# Kolla
## 简介
Kolla是OpenStack Big Tent Governace下的一个项目，项目的目标是**
为openstack云操作人员提供production-ready的容器和部署工具**
Kolla使用Docker容器和Anisble playbooks来实现这个目标。
Kolla是开箱即用的，即使你是个新手也可以很快的使用kolla快速部署你的openstack集群。Kolla也允许你根据实际的需求来定制化的部署。
## Kolla体验
可以参照kolla官方文档https://github.com/openstack/kolla/blob/master/doc/quickstart.rst进行部署。
## 实现
###配置文件管理
每个openstack服务都运行在一个容器中，那kolla是怎么管理openstack的配置的呢? 我们拿nova-compute的配置管理来举例
####a.首先kolla会使用ansible为nova-compute生成一份配置文件放在/etc/kolla/nova-compute/目录下。看代码
```yaml
#nova_custom_config默认是/etc/kolla/configs/nova
#node_config_directory默认是 /etc/kolla
- name: Copying over nova.conf
  merge_configs:
    vars:
      service_name: "{{ item }}"
    sources:
      - "{{ role_path }}/templates/nova.conf.j2"
      - "{{ node_custom_config }}/global.conf"
      - "{{ node_custom_config }}/database.conf"
      - "{{ node_custom_config }}/messaging.conf"
      - "{{ node_custom_config }}/nova.conf"
      - "{{ node_custom_config }}/nova/{{ item }}.conf"
      - "{{ node_custom_config }}/nova/{{ inventory_hostname }}/nova.conf"
    dest: "{{ node_config_directory }}/{{ item }}/nova.conf"
  with_items:
    - "nova-api"
    - "nova-compute"
    - "nova-compute-ironic"
    - "nova-conductor"
    - "nova-consoleauth"
    - "nova-novncproxy"
    - "nova-scheduler"
    - "nova-spicehtml5proxy"
```
大家可能会注意到kolla使用merge_configs来完成配置文件的合并，那么merge_configs是干什么的呢?顾名思义，merge_configs就是把多个配置文件合成一个,kolla为什么要这样做呢？
openstack配置选项非常多但是真正需要管理的则很少，对这部分选项kolla使用模版的方式管理，同时由于merge_configs的使用，使得用户可以非常方便的添加自己的定制化选项。比如你部署kolla在一台虚拟机上，你必须使用QEMU hypervisor来替代KVM hypervisor。那么你可以在/etc/kolla/config/nova/nova-compute.conf中添加以下配置
```language
[libvirt]
virt_type=qemu
```
merge_configs的代码在 ansible/action_plugins/merge_configs.py
####b.启动容器时/etc/kolla以docker卷的形式挂载到/var/lib/kolla/config_files目录下
看代码
```language
- name: Starting nova-libvirt container
  kolla_docker:
    action: "start_container"
    common_options: "{{ docker_common_options }}"
    image: "{{ nova_libvirt_image_full }}"
    name: "nova_libvirt"
    pid_mode: "host"
    privileged: True
    volumes:
      - "{{ node_config_directory }}/nova-libvirt/:{{ container_config_directory }}/:ro"
      - "/etc/localtime:/etc/localtime:ro"
      - "/lib/modules:/lib/modules:ro"
      - "/run/:/run/"
      - "/dev:/dev"
      - "/sys/fs/cgroup:/sys/fs/cgroup"
      - "kolla_logs:/var/log/kolla/"
      - "libvirtd:/var/lib/libvirt"
      - "nova_compute:/var/lib/nova/"
      - "nova_libvirt_qemu:/etc/libvirt/qemu"
  when: inventory_hostname in groups['compute']
```
####c.容器启动脚本会根据nova-compute.json来将配置文件拷贝到/etc并设置合适的权限
```language
{
    "command": "nova-compute",
    "config_files": [
        {
            "source": "{{ container_config_directory }}/nova.conf",
            "dest": "/etc/nova/nova.conf",
            "owner": "nova",
            "perm": "0600"
        }{% if nova_backend == "rbd" %},
        {
            "source": "{{ container_config_directory }}/ceph.*",
            "dest": "/etc/ceph/",
            "owner": "nova",
            "perm": "0700"
        }{% endif %}
    ]
}
```

###compute节点升级问题
由于所有服务都运行在容器中，那么是不是我升级compute节点时，该节点的虚机都会进入关机状态呢，kolla使用super-privilege的容器来解决了这个问题具体可以参考kolla PTL的文章https://sdake.io/2015/01/28/an-atomic-upgrade-process-for-openstack-compute-nodes/


##使用
###查看log
```bash
cd /var/lib/docker/volumes/kolla_logs/
```
###进入容器调试
```bash
docker exec -it service_name  bash
```
###root权限问题
出于安全考虑很多kolla服务都是运行在非root下，进入容器后拿不到root权限，可以修改/etc/kolla/nova_compute/config.json添加以下
```language
```
重启容器后即可sudo到root用户下调试

###修改架构
可以通过修改inventory文件的方法非常方便的修改部署架构
比如默认haproxy是部署在网络节点，我想把他部署在一个单独的节点
###定制化build镜像
参考 https://github.com/openstack/kolla/blob/master/doc/image-building.rst

##总结
配置管理灵活方便
可以平滑升级
部署简单
