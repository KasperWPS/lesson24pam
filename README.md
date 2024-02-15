# Домашнее задание № 15 по теме: "PAM". К курсу Administrator Linux. Professional

## Задание

- Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников
- * Дать конкретному пользователю права работать с докером и возможность рестартить докер сервис

Для выполнения задания описаны виртуальные машины 'pam' и 'docker':

- config.json
```json
[
        {
                "name": "pam",
                "cpus": 2,
                "gui": false,
                "box": "centos/8",
                "ip_addr": "192.168.57.10",
                "memory": "1024",
                "no_share": true
        },
        {
                "name": "docker",
                "cpus": 2,
                "gui": false,
                "box": "centos/8",
                "ip_addr": "192.168.57.11",
                "memory": 1024,
                "no_share": true
        }
]
```

- Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby : vsa
Vagrant.require_version ">= 2.0.0"

require 'json'

f = JSON.parse(File.read(File.join(File.dirname(__FILE__), 'config.json')))
# Локальная переменная PATH_SRC для монтирования
$PathSrc = ENV['PATH_SRC'] || "."

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  # включить переадресацию агента ssh
  config.ssh.forward_agent = true
  # использовать стандартный для vagrant ключ ssh
  config.ssh.insert_key = false

  f.each do |g|
    config.vm.define g['name'] do |s|
      s.vm.box = g['box']
      s.vm.hostname = g['name']
      s.vm.network 'private_network', ip: g['ip_addr']
      s.vm.synced_folder $PathSrc, "/vagrant", disabled: g['no_share']

      s.vm.provider :virtualbox do |virtualbox|
        virtualbox.customize ["modifyvm", :id,
          "--audio", "none",
          "--cpus", g['cpus'],
          "--memory", g['memory'],
          "--graphicscontroller", "VMSVGA",
          "--vram", "64"
        ]
        virtualbox.gui = g['gui']
        virtualbox.name = g['name']
      end
      s.vm.provision "ansible" do |ansible|
        ansible.playbook = "provisioning/playbook.yml"
        ansible.become = "true"
      end
    end
  end
end
```

Для описания конфигурации виртуальных машин использован provisioner ansible

## Задание 1.

- Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

- Создать пользователей otusadm и otus

```bash
sudo useradd otusadm && sudo useradd otus
```

- Присвоить пароли вновь созданным пользователям

```bash
echo "Otus2023!" | sudo passwd --stdin otusadm && echo "Otus2023!" | sudo passwd --stdin otus
```

- Создать группу 'admin'

```bash
sudo groupadd -f admin
```

- Добавить пользователей otusadm, root, vagrant в группу admin

```bash
sudo usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
```

- Проверить возможность подключения пользователей к виртуальной машине по ssh:

```bash
ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
ED25519 key fingerprint is SHA256:ukEvWH0PsM4bg0ziL8AByKhNQoUuUM+RDCkxeUudnAg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.57.10' (ED25519) to the list of known hosts.
otus@192.168.57.10's password:
[otus@pam ~]$
```

- Проверить, что пользователи root, vagrant и otusadm есть в группе admin

```bash
[vagrant@pam ~]$ sudo -s
[root@pam vagrant]# cat /etc/group | grep admin
printadmin:x:994:
admin:x:1003:otusadm,root,vagrant
```

- Создать скрипт /usr/local/bin/login.sh

```bash
vi /usr/local/bin/login.sh
```

```bash
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
        #Второе условие: входит ли пользователь в группу admin
        if getent group admin | grep -qw "$PAM_USER"; then
                #Если пользователь входит в группу admin, то он может подключиться
                exit 0
        else
                #Иначе ошибка (не сможет подключиться)
                exit 1
        fi
        #Если день не выходной, то подключиться может любой пользователь
else
        exit 0
fi
```

- Внести изменения в /etc/pam.d/sshd

```bash
vi /etc/pam.d/sshd
```

```
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
auth       required     pam_exec.so /usr/local/bin/login.sh
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```

- Проверить работу можно изменив дату/время на виртуальной машине (17.02.2024 - суббота)

```
date 021716342024
```

- Попробовать залогиниться с хоста на виртуальную машину

```bash
ssh otus@192.168.57.10
otus@192.168.57.10's password:
Permission denied, please try again.
otus@192.168.57.10's password:
```

- Логин пользователя из группы admin

```bash
ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password:
[otusadm@pam ~]$
```

## Задание 2. * Дать конкретному пользователю права работать с докером и возможность рестартить докер сервис

- Поправить репозитории

```bash
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```

- Установить docker

```bash
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable docker --now
```

- Добавить пользователя 'otus' в группу 'docker'


```bash
usermod otus -a -G docker
```

- Выполнить вход на виртуальную машину docker под пользователем otus

```bash
ssh otus@192.168.57.11
```

- Выполнить для проверки docker

```bash
docker run hello-world
```

Вывод:

```
[otus@docker ~]$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

- Для предоставления возможности перезапускать сервис docker необходимо внести изменения в конфигурацию sudo /etc/sudoers.d/otus

```bash
visudo /etc/sudoers.d/otus
```

```
Cmnd_Alias HANDLE_DOCKER = \
    /bin/systemctl restart docker, \
    /bin/systemctl restart docker.service

otus ALL = (root) NOPASSWD: HANDLE_DOCKER
```

После выполнения вышеприведенных действий пользователь 'otus' сможет выполнять перезапуск службы коммандами

```bash
sudo systemctl restart docker
```

```bash
sudo systemctl restart docker.service
```

playbook:

```yaml
---
- hosts: all
  become: true
  gather_facts: false

  tasks:
  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify:
      - Restart sshd

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow

  - name: Add users
    ansible.builtin.user:
      name: "{{ item }}"
      password: "$1$dysHdksd$TNBJ6Qmbld4UX8qFqi5z5." # openssl passwd -salt "dysHdksd" -1 "Otus2023!"
      state: present
    loop:
      - otusadm
      - otus

  - name: Add group "admin"
    ansible.builtin.group:
      name: admin
      state: present

  - name: Add users to group "admin"
    ansible.builtin.user:
      name: "{{ item }}"
      groups: admin
      append: true
    loop:
      - otusadm
      - root
      - vagrant

  - name: Copy pam script
    ansible.builtin.copy:
      src: ./files/login.sh
      dest: /usr/local/bin
      owner: root
      group: root
      mode: '0750'

  - name: Insert plugin PAM (/etc/pam.d/sshd)
    ansible.builtin.lineinfile:
      path: /etc/pam.d/sshd
      insertafter: '^auth.*postlogin$'
      line: 'auth       required     pam_exec.so /usr/local/bin/login.sh'

  handlers:
  - name: Restart sshd
    service:
      name: sshd
      state: restarted

- hosts: docker
  become: true
  gather_facts: false

  tasks:

  - name: List all files in directory /etc/yum.repos.d/*.repo
    find:
      paths: "/etc/yum.repos.d/"
      patterns: "*.repo"
    register: repos

  - name: Comment mirrorlist /etc/yum.repos.d/CentOS-*
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^(mirrorlist=.+)'
      line: '#\1'
    with_items: "{{ repos.files }}"

  - name: Replace baseurl
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^#baseurl=http:\/\/mirror.centos.org(.+)'
      line: 'baseurl=http://vault.centos.org\1'
    with_items: "{{ repos.files }}"

  - name: Install yum-utils
    ansible.builtin.yum:
      name: yum-utils
      state: present

  - name: Add Docker repository
    shell: yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    args:
      creates: /etc/yum.repos.d/docker-ce.repo

  - name: Install docker packages
    ansible.builtin.yum:
      name:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        - docker-buildx-plugin
        - docker-compose-plugin
      state: present
    notify: Up docker service

  - name: Add users to group "docker"
    ansible.builtin.user:
      name: "{{ item }}"
      groups: docker
      append: true
    loop:
      - otus
      - vagrant

  - name: Copy sudo config for accept user "otus" restart service docker
    ansible.builtin.copy:
      src: ./files/sudoers_otus
      dest: /etc/sudoers.d/otus
      owner: root
      group: root
      mode: '0640'

  handlers:
  - name: Up docker service
    ansible.builtin.service:
      name: docker
      enabled: true
      state: started
```

