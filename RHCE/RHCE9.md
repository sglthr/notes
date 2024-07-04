# RHCE9

## 1、安装和配置ansible

```shell
sudo yum install -y ansible-core ansible-navigator
mkdir -p /home/greg/ansible/{roles,mycollection}
cd ansible/
cat /etc/ansible/ansible.cfg
ansible-config init --disabled > ansible.cfg

cp ansible.cfg ansible.cfg.bak
# 开两个窗口，把.cfg的内容除了第一行全部删掉，然后从bak查找模板，把需要的粘贴过去
vim ansible.cfg
vim ansible.cfg.bak

# 最终配置为如下所示，vault_password_file那行先注释，16题做完后再打开
cat ansible.cfg
[defaults]
inventory=/home/greg/ansible/inventory
roles_path=/home/greg/ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles
collections_path=/home/greg/ansible/mycollection:/usr/share/ansible/collections
#vault_password_file=/home/greg/ansible/secret.txt
remote_user=greg
host_key_checking=False
[privilege_escalation]
become=True

# 创建主机清单
vim /home/greg/ansible/inventory
[dev]
node1
[test]
node2
[prod]
node3
node4
[balancers]
node5
[webservers:children]
prod

# 验证
# 查看角色目录
ansible-galaxy list
# /home/greg/ansible/roles
# /usr/share/ansible/roles
# /etc/ansible/roles

# 查看主机清单
ansible-inventory --graph
@all:
  |--@balancers:
  |  |--node5
  |--@dev:
  |  |--node1
  |--@test:
  |  |--node2
  |--@ungrouped:
  |--@webservers:
  |  |--@prod:
  |  |  |--node3
  |  |  |--node4
  
# 测试能否成功远程控制
ansible all -m ping

# 更新执行环境（必要操作）
podman login utility.lab.example.com -u admin -p redhat
ansible-navigator images

# 查看已安装的集合
ansible-navigator collections
```



## 2、配置默认存储库

```shell
# 查模块名称
ansible-doc -l | grep yum
# 查模块用法，查找示例
ansible-doc yum_repository

# vim相关配置
echo set nu ts=2 et cuc sw=2 autoindent > ~/.vimrc

# 编辑剧本
vim /home/greg/ansible/yum_repo.yml
---
- name: Set Yum_repo
  hosts: all
  tasks:
    - name: EX294_BASE
      ansible.builtin.yum_repository:
        name: EX294_BASE
        description: "EX294 base software"
        file: EX294_BASE
        baseurl: http://content/rhel9.0/x86_64/dvd/BaseOS
        gpgcheck: yes
        gpgkey: http://content/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: yes
        
    - name: EX294_STREAM
      ansible.builtin.yum_repository:
        name: EX294_STREAM
        description: "EX294 stream software"
        file: EX294_STREAM
        baseurl: http://content/rhel9.0/x86_64/dvd/AppStream
        gpgcheck: yes
        gpgkey: http://content/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: yes
        
# 运行剧本
ansible-navigator run yum_repo.yml -m stdout

# 验证
ansible all -m shell -a "yum repoinfo"
ansible all -m shell -a "yum install -y ftp"
ansible all -m shell -a "rpm -q ftp"
```



## 3、安装软件包

```shell
# 查看示例
ansible-doc yum

# 编辑剧本
vim /home/greg/ansible/packages.yml
---
- name: Install task 1
  hosts: dev,test,prod
  tasks: 
    - name: install packages
      ansible.builtin.yum:
        name: "{{ packages }}"
        state: present
      vars:
        packages:
        - php
        - mariadb

    - name: install group
      ansible.builtin.yum:
        name: "@RPM Development Tools"
        state: present
      when: inventory_hostname in groups.dev
      
    - name: update all packages
      ansible.builtin.yum:
        name: '*'
        state: latest
      when: inventory_hostname in groups.dev
# 如果是多个主机组，可以按照以下示例的方法写，或者直接另写一段playbook，把hosts重新定义
# when: inventory_hostname in groups.dev or inventory_hostname in groups.test


# 验证
ansible-navigator run packages.yml -m stdout
ansible dev,test,prod -m shell -a "rpm -q php mariadb"
ansible dev -m shell -a "yum grouplist"
ansible dev -m shell -a "yum update"
```



## 4、使用RHEL系统角色（or 18）

```shell
# 查找包名
yum search role

# 安装系统角色
sudo yum install -y rhel-system-roles.noarch
ansible-galaxy list

# 复制模板
cp /usr/share/doc/rhel-system-roles/selinux/example-selinux-playbook.yml selinux.yml

# 编辑剧本，保留两个变量，tasks删除到block块即可
vim /home/greg/ansible/selinux.yml
---
- hosts: all
  vars:
    selinux_policy: targeted
    selinux_state: enforcing
  tasks:
    - name: execute the role and catch errors
      block:
        - name: Include selinux role
          include_role:
            name: rhel-system-roles.selinux
      rescue:
        # Fail if failed for a different reason than selinux_reboot_required.
        - name: handle errors
          fail:
            msg: "role failed"
          when: not selinux_reboot_required

        - name: restart managed host
          reboot:

        - name: wait for managed host to come back
          wait_for_connection:
            delay: 10
            timeout: 300

        - name: reapply the role
          include_role:
            name: rhel-system-roles.selinux
            
# 使用rpm包安装的角色，还是用ansible-playbook执行
# 使用集合安装的角色，要用ansible-navigator执行
ansible-playbook selinux.yml
ansible all -m shell -a 'getenforce; grep ^SELINUX= /etc/selinux/config'
```



## 5、配置collection

```shell
# 这里要注意- name前面不能有空格
vim requirements.yml
---
collections:
- name: http://classroom/materials/redhat-insights-1.0.7.tar.gz
- name: http://classroom/materials/community-general-5.5.0.tar.gz
- name: http://classroom/materials/redhat-rhel_system_roles-1.19.3.tar.gz

# 验证，这里collection不能加s，否则报错
ansible-galaxy collection install -r requirements.yml -p /home/greg/ansible/mycollection

# 查看是否有上面三个包
ansible-navigator collections

ansible-navigator doc community.general.filesystem -m stdout
```



## 6、使用ansible galaxy 安装角色

```shell
vim /home/greg/ansible/roles/requirements.yml
---
- src: http://classroom/materials/haproxy.tar
  name: balancer
- src: http://classroom/materials/phpinfo.tar
  name: phpinfo
  
# 验证
ansible-galaxy install -r /home/greg/ansible/roles/requirements.yml
ansible-galaxy list
```



## 7、创建和使用角色

```shell
# 初始化创建apache角色
ansible-galaxy role init --init-path /home/greg/ansible/roles apache

# 查看是否创建成功
ansible-galaxy list

# 查看模板
ansible-doc yum
ansible-doc systemd
ansible-navigator doc firewalld -m stdout

# 编辑tasks，这里每个task的name要写在第一行
vim roles/apache/tasks/main.yml
---
- name: Install Apache
  ansible.builtin.yum:
    name: httpd
    state: latest

- name: Start Apache
  ansible.builtin.systemd:
    name: httpd
    state: started
    enabled: yes

- name: Start Firewalld
  ansible.builtin.systemd:
    name: firewalld
    state: started
    enabled: yes

- name: Allow http
  ansible.posix.firewalld:
    service: http
    permanent: yes
    state: enabled
    immediate: yes

- template:
    src: index.html.j2
    dest: /var/www/html/index.html

# 编辑j2模板，注意点和下划线（"." 和 "_"）
vim roles/apache/templates/index.html.j2
Welcome to {{ ansible_nodename }} on {{ ansible_default_ipv4.address }}

# 编辑剧本
vim /home/greg/ansible/apache.yml
---
- name: webservers role
  hosts: webservers
  roles:
  - apache
  
# 验证
ansible-navigator run apache.yml -m stdout
ansible webservers --list-hosts

# 尝试访问
curl http://node3
Welcome to node3.lab.example.com on 172.25.250.11

curl http://node4
Welcome to node4.lab.example.com on 172.25.250.12
```



## 8、从ansible galaxy使用角色

```shell
# 编辑剧本
vim /home/greg/ansible/roles.yml
---
- name: Use role balancer
  hosts: balancers
  roles:
  - balancer

- name: Use role phpinfo
  hosts: webservers
  roles: 
  - phpinfo
  
# 验证
ansible-navigator run roles.yml -m stdout

# 多次访问，验证是否轮询
curl http://node5
curl http://node5/hello.php | head -n 3
```



## 9、创建和使用逻辑卷（or 19）

```shell
# 查找模板
ansible-navigator doc community.general.lvol -m stdout
ansible-navigator doc community.general.filesystem -m stdout
ansible-doc debug

# 编辑剧本
vim /home/greg/ansible/lv.yml
---
- name: Create volume
  hosts: all
  tasks: 
    - block:
      - name: Create a logical volume of 1500m
        community.general.lvol:
          vg: research
          lv: data
          size: 1500

      - name: Create a ext4 filesystem of 1500m
        community.general.filesystem:
          fstype: ext4
          dev: /dev/research/data
      
      rescue:
      - name: Create failed
        ansible.builtin.debug:
          msg: Could not create logical volume of that size

      - name: Create a logical volume of 800m
        community.general.lvol:
          vg: research
          lv: data
          size: 800

      - name: Create a ext4 filesystem of 800m
        community.general.filesystem:
          fstype: ext4
          dev: /dev/research/data
      when: ansible_lvm.vgs.research is defined
      
    - debug:
        msg: Volume group done not exist
      when: ansible_lvm.vgs.research is not defined
      
# 验证
ansible-navigator run lv.yml -m stdout
ansible all -m shell -a 'lvs'
ansible all -m shell -a 'blkid /dev/research/data'
```



## 10、生成主机文件

```shell
wget http://classroom/materials/hosts.j2

# 编写j2模板
vim hosts.j2
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
{% for i in groups.all %}
{{ hostvars[i].ansible_default_ipv4.address }} {{ hostvars[i].ansible_nodename }} {{ hostvars[i].ansible_hostname }}
{% endfor %}

# 查找模板
ansible-doc template

# 编写剧本
vim hosts.yml
---
- name: Generate hosts file
  hosts: all
  tasks:
    - name: Template a file to /etc/myhosts
      ansible.builtin.template:
        src: hosts.j2
        dest: /etc/myhosts
      when: inventory_hostname in groups.dev
      
# 验证
ansible-navigator run hosts.yml -m stdout
ansible all -m shell -a 'cat /etc/myhosts'
```



## 11、修改文件内容

```shell
# 查找模板
ansible-doc copy

# 编写剧本
vim /home/greg/ansible/issue.yml
---
- name: Issue
  hosts: all
  tasks:
    - name: Content 1
      ansible.builtin.copy:
        content: 'Development'
        dest: /etc/issue
      when: inventory_hostname in groups.dev

    - name: Content 2
      ansible.builtin.copy:
        content: 'Test'
        dest: /etc/issue
      when: inventory_hostname in groups.test

    - name: Content 1
      ansible.builtin.copy:
        content: 'Production'
        dest: /etc/issue
      when: inventory_hostname in groups.prod
      
# 验证
ansible-navigator run issue.yml -m stdout
ansible all -m shell -a 'cat /etc/issue'
```



## 12、创建Web内容目录

```shell
# 查找模板
ansible-doc file
ansible-doc copy
ls -lZ /var/www/
httpd_sys_content_t

# 编写剧本，这里注意目录权限'2775'必须要用引号，否则不生效
vim /home/greg/ansible/webcontent.yml
---
- name: webcontent
  hosts: dev
  roles:
  - apache
  tasks:
    - name: Create a dir
      ansible.builtin.file:
        path: /webdev
        group: webdev
        state: directory
        mode: '2775'

    - name: Create a symbolic link
      ansible.builtin.file:
        src: /webdev
        dest: /var/www/html/webdev
        state: link
    
    - name: Copy using inline content
      ansible.builtin.copy:
        content: 'Development'
        dest: /webdev/index.html
        setype: httpd_sys_content_t
        
# 验证
ansible-navigator run webcontent.yml -m stdout
curl http://node1/webdev/
```



## 13、生成硬件报告

```shell
# 查找模板
ansible dev -m setup | grep -e hostname -e mem -e bios
ansible dev -m setup -a 'filter=*device*'
ansible-doc get_url
ansible-doc lineinfile
curl http://classroom/materials/hwreport.empty

# 编写剧本
vim /home/greg/ansible/hwreport.yml
---
- name: hwreport
  hosts: all
  tasks: 
   - name: Download hwreport
     ansible.builtin.get_url:
       url: http://classroom/materials/hwreport.empty
       dest: /root/hwreport.txt

   - name: Report 1
     ansible.builtin.lineinfile:
       path: /root/hwreport.txt
       regexp: '^HOST='
       line: HOST={{ ansible_hostname }}

   - name: Report 2
     ansible.builtin.lineinfile:
       path: /root/hwreport.txt
       regexp: '^MEMORY='
       line: MEMORY={{ ansible_memtotal_mb }}

   - name: Report 3
     ansible.builtin.lineinfile:
       path: /root/hwreport.txt
       regexp: '^BIOS='
       line: BIOS={{ ansible_bios_version }}

   - name: Report 4
     ansible.builtin.lineinfile:
       path: /root/hwreport.txt
       regexp: '^DISK_SIZE_VDA='
       line: DISK_SIZE_VDA={{ ansible_devices.vda.size }}

   - name: Report 5
     ansible.builtin.lineinfile:
       path: /root/hwreport.txt
       regexp: '^DISK_SIZE_VDB='
       line: DISK_SIZE_VDB={{ ansible_devices.vdb.size | default('NONE', true) }}
       
# 测试
ansible-navigator run hwreport.yml -m stdout
ansible all -m shell -a 'cat /root/hwreport.txt'
```



## 14、创建密码库

```shell
# 创建密钥文件
echo whenyouwishuponastar > /home/greg/ansible/secret.txt

# 创建密码库
ansible-vault create /home/greg/ansible/locker.yml --vault-password-file=/home/greg/ansible/secret.txt
---
pw_developer: Imadev
pw_manager: Imamgr

# 验证
cat locker.yml
ansible-vault view locker.yml --vault-password-file=secret.txt
```



## 15、创建用户账号

```shell
# 查找模板
ansible-doc group
ansible-doc user
wget http://classroom/materials/user_list.yml
cat user_list.yml

# 编写剧本，注意创建用户那里为groups，不是group，代表补充组（一个用户可以属于多个组）
vim /home/greg/ansible/users.yml
---
- name: Create user1
  hosts: dev,test
  vars_files:
  - locker.yml
  - user_list.yml
  tasks:
    - name: Ensure group "devops" exists
      ansible.builtin.group:
        name: devops
        state: present

    - name: Add user1
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: devops
        password: "{{ pw_developer | password_hash('sha512') }}"
        password_expire_max: "{{ item.password_expire_max }}"
      loop: "{{ users }}"
      when: item.job == 'developer'

- name: Create user2
  hosts: prod
  vars_files:
  - locker.yml
  - user_list.yml
  tasks:
    - name: Ensure group "opsmgr" exists
      ansible.builtin.group:
        name: opsmgr
        state: present

    - name: Add user2
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: opsmgr
        password: "{{ pw_manager | password_hash('sha512') }}"
        password_expire_max: "{{ item.password_expire_max }}"
      loop: "{{ users }}"
      when: item.job == 'manager'

# 验证
ansible-navigator run users.yml --vault-password-file=secret.txt -m stdout
ansible all -m shell -a 'id bob;id sally;id fred'

ssh bob@node1
bob@node1's password: ’Imadev‘
chage -l bob

ssh sally@node3
sally@node3's password: 'Imamgr'
chage -l sally
```



## 16、更新ansible库的密钥

```shell
wget http://classroom/materials/salaries.yml

ansible-vault rekey salaries.yml 
Vault password: 'insecure8sure'
New Vault password: 'bbs2you9527'
Confirm New Vault password: 'bbs2you9527'
Rekey successful

# 验证
ansible-vault view salaries.yml 
Vault password: 'bbs2you9527'
haha

# 添加密钥配置，将第一题中注释的部分取消注释
vim ansible.cfg
vault_password_file=/home/greg/ansible/secret.txt
```



## 17、配置cron作业

```shell
# 查找模板
ansible-doc cron

# 编写剧本
vim /home/greg/ansible/cron.yml
---
- name: cron
  hosts: test
  tasks: 
    - name: cron job
      ansible.builtin.cron:
        name: "job1"
        minute: "*/2"
        hour: "*"
        job: 'logger "EX200 in progress"'
        user: natasha

# 验证
ansible test -m shell -a 'crontab -l -u natasha'
```



## 18、使用TimeSync RHEL系统角色（or 4）

```shell
sudo yum install -y rhel-system-roles

# 获取timesync角色
cp /usr/share/doc/rhel-system-roles/timesync/example-single-pool-playbook.yml  /home/greg/ansible/timesync.yml

# cp -r /usr/share/ansible/roles/rhel-system-roles.timesync/ roles/timesync

# 编写剧本
vim /home/greg/ansible/timesync.yml
---
- name: NTP
  hosts: all
  roles:
    - rhel-system-roles.timesync
  vars:
    timesync_ntp_servers:
      - hostname: classroom.lab.example.com
        iburst: yes

# 验证
ansible-playbook timesync.yml
ansible all -m shell -a 'cat /etc/chrony.conf'
ansible all -m shell -a 'systemctl status chronyd'
```



## 19、创建分区（or 9）

```yaml
---
- name: LVM
  hosts: balancers
  tasks:
    - name: Create /newpart1
      ansible.builtin.file:
        path: /newpart1
        state: directory
    
    - name: Create /newpart2
      ansible.builtin.file:
        path: /newpart2
        state: directory

    - block:
      - name: Create /dev/vdc1 with a size of 1500MiB
        community.general.parted:
          device: /dev/vdc
          number: 1
          state: present
          part_end: 1500MiB
      
      - name: Create a ext4 filesystem on /dev/vdc1
        community.general.filesystem:
          fstype: ext4
          dev: /dev/vdc1
      
      - name: Mount /dev/vdc1
        ansible.posix.mount:
          path: /newpart2
          src: /dev/vdc1
          fstype: ext4
          state: mounted

      - name: Create /dev/vdb1 with a size of 1500MiB
        community.general.parted:
          device: /dev/vdb
          number: 1
          state: present
          part_end: 1500MiB
      
      - name: Create a ext4 filesystem on /dev/vdb1
        community.general.filesystem:
          fstype: ext4
          dev: /dev/vdb1
      
      - name: Mount /dev/vdb1
        ansible.posix.mount:
          path: /newpart1
          src: /dev/vdb1
          fstype: ext4
          state: mounted

      rescue:
      - name: Create /dev/vdb1 with a size of 800MiB
        community.general.parted:
          device: /dev/vdb
          number: 1
          state: present
          part_end: 800MiB
        when: ansible_devices.vdb is defined

      - name: Create a ext4 filesystem on /dev/vdb1
        community.general.filesystem:
          fstype: ext4
          dev: /dev/vdb1
        when: ansible_devices.vdb is defined
      
      - name: Mount /dev/vdb1
        ansible.posix.mount:
          path: /newpart1
          src: /dev/vdb1
          fstype: ext4
          state: mounted
        when: ansible_devices.vdb is defined

      - name: Create /dev/vdc1 with a size of 800MiB
        community.general.parted:
          device: /dev/vdc
          number: 1
          state: present
          part_end: 800MiB
        when: ansible_devices.vdc is defined
      
      - name: Create a ext4 filesystem on /dev/vdc1
        community.general.filesystem:
          fstype: ext4
          dev: /dev/vdc1
        when: ansible_devices.vdc is defined
      
      - name: Mount /dev/vdc1
        ansible.posix.mount:
          path: /newpart2
          src: /dev/vdc1
          fstype: ext4
          state: mounted
        when: ansible_devices.vdc is defined

    - name: Vdd is not defined
      debug:
        msg: Volume group done not exist
      when: ansible_devices.vdd is not defined
```

