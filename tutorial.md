
Test thử Ansible với cơ chế dùng password để kết nối với target node

```bash
root@Ansible:~# cat /etc/ansible/hosts
[tao]
172.16.99.131

[tao:vars]
ansible_user=root
ansible_password='Solus2025!'
```

Tao file `/etc/ansible/ansible_vault_password` 

```bash
echo 'Solus2025!' > /etc/ansible/vault-pass.txt
chmod 600 /etc/ansible/vault-pass.txt
```

Sửa file cấu hình của ansible nhu sau

```bash
nano /etc/ansible/ansible.cfg
```

```bash
root@Ansible:~# cat /etc/ansible/ansible.cfg
[defaults]
host_key_checking = False
vault_password_file = /etc/ansible/vault-pass.txt
```

Thực hiện test:

```bash
root@Ansible:~# ansible tao -m ping
172.16.99.131 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

```

Tao plabook `hello-world.yml`
```bash
cat << EOF > /opt/hello-world.yml
- name: Echo hello
  hosts: all
  connection: ssh
  become: no
  tasks:
    - name: "Echo hello string"
      shell: echo "Hello-world" >> /root/echo.txt
EOF
```
Tạo blueprint trên VRA

```bash
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
  NTP_srv:
    type: string
    enum:
      - 172.16.99.160
      - 0.0.0.0
      - localhost
    description: Chọn IP Server NTP
    title: Select IP NTP
    default: 172.16.99.160
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
    pattern: ^[a-z0-9A-Z]+$
    title: Login Account
    description: Tài khoản đăng nhập máy chủ
    minLength: 6
    maxLength: 20
  userpassword:
    type: string
    #pattern: '^[a-z0-9A-Z@#$]+$'
    pattern: (?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[^A-Za-z0-9])(?=.{8,})
    encrypted: true
    title: Login Password
    description: Thiết lập mật khẩu cho tài khoản đăng nhập máy chủ
    minLength: 6
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
      account: vRA-Ansible
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
          - /opt/hello-world.yml
      hostVariables:
        hostname: localhost
        NTP_srv: ${input.NTP_srv}

```