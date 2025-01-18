# Ghi chép về vRA

Tạo một máy chủ ảo làm ansible để thực hiện làm tự động hóa trong vRA - [link hướng dẫn](https://www.vcrocs.info/vmware-aria-automation-ansible-integration/)

Cài đặt gói cơ bản để ansible có thể sử dụng được trong VRA trên máy chủ Ansible 
```
apt update 
apt install python3-pip -y
apt update 
apt install python3-pip -y
python3 -m pip install --upgrade pip
pip3 install pynetbox==7.0.0
pip3 install ansible==2.9.17
pip3 install netaddr
pip3 install pyvmomi
pip3 install jmespath
ansible-galaxy collection install netbox.netbox
ansible-galaxy collection install community.vmware
ansible-galaxy collection install ansible.utils
ansible-galaxy collection install community.general
ansible-galaxy collection install checkmk.general
```

[Test thử trước khi thực hiện với vRA](tutorial.md)

Play sẽ thực hiện các role để cài từng thành phần
```bash
- hosts: all
  roles:
    - { role: app, become: true }
    - { role: mariadb, become: true }
    - { role: notify, become: true }
```

Trong repo này đang xây dựng quy trình như sau:
- role: `app`
    - Cho phép người dùng chọn 1 trong các user devadmin, dbaadmin, sysadmin  (kết hợp loại input enum hay string ...) để tạo tài khoản cho OS, Người dùng nhập mật khẩu cho User OS.
    - Cài đặt agnet checkmk cho client - Phía playbook đang khai báo đường dẫn trực tiếp đến Checkmk Server để caif checkmk Agent. Ansible tự động add Host lên CheckMK server 
    - Cấu hình rsyslog cho client - Chọn Log Server từ Input vRA
- role: `mariadb`
    - Cài đặt mariaDB ==> Chọn version từ Input. Người dùng Nhập thông tin: Tên DB, user/pass cho MySQL từ Input. 
- role: `notify`
    - Gửi thông báo về email cho người dùng về: IP, user, thông tin phiên bản, tk của OS/mariaDB</br>![image1](/image/vra_1.png)


Phía vRA thực hiện cấu hình blueprint:
```
formatVersion: 1
inputs:
  image:
    type: string
    enum:
      - Ubuntu20
    description: Chọn phiên bản hệ điều hành
    title: Select OS
    default: Ubuntu20
  size:
    type: string
    oneOf:
      - title: Small (1 Cores Cpu, 2GB Memory)
        const: flavor-1-2
      - title: Lagger (2 Cores Cpu, 4GB Memory)
        const: flavor-2-4
      - title: Medium (4 Cores Cpu, 8GB Memory
        const: flavor-4-8
    description: Chọn tài nguyên cho máy chủ
    title: Resource Size
    default: flavor-1-2
  # NTP_srv:
  #   type: string
  #   enum:
  #     - 172.16.99.160
  #     - 0.0.0.0
  #     - localhost
  #   description: Chọn IP Server NTP
  #   title: Select IP NTP
  #   default: 172.16.99.160
  Log_srv:
    type: string
    enum:
      - 172.16.99.207:12514
      - 0.0.0.0
      - localhost
    description: Chọn IP Server Log
    title: Select IP Log
    format: hidden
    default: 172.16.99.207:12514
  network:
    type: string
    oneOf:
      - title: VLAN99 MGMT Network (172.16.99.0/24)
        const: net:vlan99
    description: Chọn dải mạng được gán cho máy chủ
    title: Network
    default: net:vlan99
  hostname:
    type: string
    minLength: 4
    maxLength: 15
    pattern: '[a-z0-9A-Z]+'
    title: Server hostname
    description: 'Tên máy chủ được thiết lập trong phần Computer Name. Vd: DEMO-EOFFICE-FILE01'
  username:
    type: string
    enum:
      - devadmin
      - sysadmin
      - dbadmin
    default: devadmin
    title: Login Account
    description: Tài khoản đăng nhập máy chủ
  userpassword:
    type: string
    #pattern: '^[a-z0-9A-Z@#$]+$'
    pattern: (?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[^A-Za-z0-9])(?=.{8,})
    encrypted: true
    title: Login Password
    description: Thiết lập mật khẩu cho tài khoản đăng nhập máy chủ
    minLength: 6
  db_ver:
    type: string
    enum:
      - '10.6'
      - '11.4'
    description: Chọn Version MariaDB
    title: Select Version MariaDB
    default: '10.6'
  db_name:
    type: string
    default: database_name
    pattern: '[a-z0-9A-Z]+'
    title: Database Name
    description: Database Name
  db_user:
    type: string
    enum:
      - adminmysql
      - dbadmin
    default: dbadmin
    title: DB User Account
    description: DB User Account
  db_pass:
    type: string
    pattern: (?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[^A-Za-z0-9])(?=.{8,})
    title: DB User Password
    description: DB User Password
    encrypted: true
resources:
  Cloud_vSphere_Network_1:
    type: Cloud.vSphere.Network
    properties:
      networkType: existing
      constraints:
        - tag: ${input.network}
  Cloud_vSphere_Machine_1:
    type: Cloud.vSphere.Machine
    properties: #Tên được đặt cho host trên vcenter
      name: vRA-${to_upper(input.hostname)}
      #Tên được đặt cho hostname trong OS
      hostname: ${input.hostname}
      image: ${input.image}
      flavor: ${input.size}
      networks:
        - network: ${resource.Cloud_vSphere_Network_1.id}
          assignment: static
      tags:
        - key: requestedBy
          value: ${env.requestedBy}
        - key: projectName
          value: ${env.projectName}
        - key: requestedAt
          value: ${env.requestedAt}
        - key: deploymentName
          value: ${env.deploymentName}
      constraints:
        - tag: wd-solus
      cloneStrategy: FULL
      # customizeGuestOs: false
      cloudConfig: |
        users:
          - name: root
            lock_passwd: false
          - name: ${input.username}
            lock_passwd: false
            sudo: ['ALL=(ALL) NOPASSWD:ALL']
            groups: [sudo, admin]
            shell: '/bin/bash'
            ssh_pwauth: yes
        network:
          version: 1
          config:
            - type: physical
              name: eth0
              subnets: 
                - type: static
                  address: ${resource.Cloud_vSphere_Machine_1.networks.address[0]}/${resource.Cloud_vSphere_Network_1.prefixLength}
                  gateway: ${resource.Cloud_vSphere_Network_1.gateway}
        chpasswd:
          list: |
            ${input.username}:${input.userpassword}
            root:Solus2025!
          expire: false 
        # expire: false: Nếu muốn giữ password ban đầu, Comment: lần đăng nhập đầu tiên user phải đổi password
        write_files:
        - path: /var/tmp/initdisk.sh
          permissions: '0755'
          content: |
            #!/bin/bash
      snapshotLimit: 5
  Cloud_Ansible_1:
    type: Cloud.Ansible
    properties:
      account: vra-ansible1
      inventoryFile: /etc/ansible/hosts
      host: ${resource.Cloud_vSphere_Machine_1.*}
      # hostName: localhost
      username: root
      password: Solus2025!
      osType: linux
      # groups:
        # - vms
      maxConnectionRetries: 10
      playbooks:
        provision:
          - /opt/1.vra1.yml
          # - /opt/vra3.yml
      hostVariables:
        hostname: ${input.hostname}
        # NTP_srv: ${input.NTP_srv}
        IP: ${resource.Cloud_vSphere_Machine_1.networks.address[0]}/${resource.Cloud_vSphere_Network_1.prefixLength}
        projectName: ${env.projectName}
        os_user: ${input.username}
        os_pass: ${input.userpassword}
        RequestedAt: ${env.requestedAt}
        Log_srv: ${input.Log_srv}
        db_ver: ${input.db_ver}
        resource_name1: ${resource.Cloud_vSphere_Machine_1.name}/${resource.Cloud_vSphere_Machine_1.hostname}/${resource.Cloud_vSphere_Machine_1.providerId}/${resource.Cloud_vSphere_Machine_1.resourceName}
        resource_name: ${resource.Cloud_vSphere_Machine_1.resourceName}
        db_name: ${input.db_name}
        db_pass: ${input.db_pass}
        db_user: ${input.db_user}

```
Lưu ý: Cần sửa lại nội dung của thông báo tại file: `~/vra/notify/vars/main.yml`</br>![image1](/image/vra_3.png)

Khi Deloy sẽ có kết quả như sau: </br>![image1](/image/vra_2.png)

