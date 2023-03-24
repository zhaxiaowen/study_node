# ansible使用总结

[ansible-playbook权限提升多种方式](https://www.136.la/android/show-27156.html)

#### 1.ansible提权root执行任务,免输密码

```
# 1. 修改 /etc/ansible/hosts
[apollo_change]
172.24.200.8 ansible_sudo_pass="K2JwUwt3orCkoN5qqWxn2x7SXtB5aB"

# 2.test.yml
- hosts: apollo_change
  remote_user: carapp
  tasks:
    - name: " 复制配置文件 "
      file: path=/data/scripts/test.txt state=touch
      become: true
      become_user: root
# 3. ansible-playbook test.yml
```

#### 2.ansible提权root执行任务,免输密码

```
# 1. 修改 /etc/ansible/hosts
[apollo_change]
172.24.200.8

# 2.test.yml
- hosts: apollo_change
  remote_user: carapp
  become: yes
  vars_files:
    - v1.yml
  vars:
    ansible_become_pass: "{{ root_pass_sirius }}"
  tasks:
    - name: " 复制配置文件 "
      file: path=/data/scripts/test.txt state=touch
      become: true
      become_user: root
# 3. vi v1.yml
root_pass_sirius: K2JwUwt3orCkoN5qqWxn2x7SXtB5aB

# 4.ansible-playbook test.yml
```

#### 3.ansible提权root执行任务,免输密码

```
# 1. 修改 /etc/ansible/hosts
[apollo_change]
172.24.200.8

# 2.test.yml
- hosts: apollo_change
  remote_user: carapp
  become: yes
  vars:
    ansible_become_pass: "{{ root_pass_sirius }}"
  tasks:
    - name: " 复制配置文件 "
      file: path=/data/scripts/test.txt state=touch
      become: true
      become_user: root

# 3.ansible-playbook test.yml -e "root_pass_sirius=K2JwUwt3orCkoN5qqWxn2x7SXtB5aB"
```

#### 4.ansible提权root执行任务,免输密码

```
# 1. 修改 /etc/ansible/hosts
[apollo_change]
172.24.200.8 ansible_sudo_pass="K2JwUwt3orCkoN5qqWxn2x7SXtB5aB"

# 2. ansible apollo_change -m shell -a "whoami"  --become --become-method=sudo --become-user=root
```



#### ansible指定host文件执行

```\
ansible -i host all -m shell -a "hostname"
```

#### [ansible配置ansible.cfg,修改默认inventory](https://blog.csdn.net/liumiaocn/article/details/95351475)

```
# 1. vi /data/scripts/test/ansible.cfg
[defaults]
inventory      = /data/scripts/monitor_add_script/host

# 2. vi /data/scripts/test/host
[apollo_change]
172.24.200.8 

# 3. ansible all --list-hosts
  hosts (1):
    172.24.200.8

```

