# Первые шаги с Ansible
Описание/Пошаговая инструкция выполнения домашнего задания:
Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:

- необходимо использовать модуль yum/apt;
- конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными;
- после установки nginx должен быть в режиме enabled в systemd;
- должен быть использован notify для старта nginx после установки;
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible.

Сделать все это с использованием Ansible роли

Критерии оценки:
Статус "Принято" ставится, если:
- предоставлен Vagrantfile и готовый playbook/роль (инструкция по запуску стенда, если посчитаете необходимым);
- после запуска стенда nginx доступен на порту 8080;
- при написании playbook/роли соблюдены перечисленные в задании условия.

## Установка и настройка Ansible
На своей виртуальной машине c Ubuntu 20.04.3 LTS установил ansible:
```
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt-get install ansible
```
Убедился, что версия python и ansible соответсвуют нужным:
```
sam@yarkozloff:/otus/ansible$ ansible --version
ansible [core 2.12.4]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/sam/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/sam/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Mar 15 2022, 12:22:08) [GCC 9.4.0]
  jinja version = 2.10.1
  libyaml = True
```

## Подготовка окружения
В связи с недоступностью Vagrant cloud, бокс centos/7 был загружен локально на виртуальную машину с Ubuntu 20.04.3 как это и делалось ранее.
Сам бокс дополнительно сохранил в свой локальный репозиторий (который создал на прошлом дз на другом сервере): http://93.185.166.183/repo/CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box
Создал каталог ansible, разместил бокс и добавил его в vagrant:
```
sam@yarkozloff:/otus/ansible$ vagrant box add centos7 CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'centos7' (v0) for provider:
    box: Unpacking necessary files from: file:///otus/ansible/CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box
==> box: Successfully added box 'centos7' (v0) for 'virtualbox'!
```
Также создал Vagrantfile с указанием названия загруженного бокса, дополнительно выпонлены настройки:
```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :nginx => {
        :box_name => "centos7",
        :ip_addr => '192.168.11.150'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "200"]
          end
          
          box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
          SHELL

      end
  end
end
```
Машина поднялась командой vagrant up, по vagrant ssh есть доступ. Теперь, чтобы каждый раз явно указывать инвентори
файл и вписывать в него много информации, сразу создаем inventory файл в каталоге staging/hosts с нужными параметрами (убрав информацию о пользователе):
```
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```
и создать в каталоге ansible конфиг ansible.cfg:
```
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```
Итого имеем рабочий каталог ansible с содержимым:
```
sam@yarkozloff:/otus/ansible$ ls
ansible.cfg  CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box staging  Vagrantfile
```
Убедимся, что управляемый хост доступен:
```
sam@yarkozloff:/otus/ansible$ ansible nginx -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
Теперь, когда мы убедились, что у нас все подготовлено - установлен Ansible, поднят хост для теста и Ansible имеет к нему доступ, можно конфигурировать хост. Для начала воспользуемся Ad-Hoc командами и выполним некоторые удаленные команды на хосте.
Проверим ядро на хосте:
```
sam@yarkozloff:/otus/ansible$ ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-862.2.3.el7.x86_64
```
Проверим статус сервиса firewalld
```
sam@yarkozloff:/otus/ansible$ ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "name": "firewalld",
    "status": {
        "ActiveEnterTimestampMonotonic": "0",
        "ActiveExitTimestampMonotonic": "0",
        "ActiveState": "inactive", --НЕ АКТИВЕН
```
Установим пакет epel-release на хост:
```
sam@yarkozloff:/otus/ansible$ ansible nginx -m yum -a "name=epel-release state=present" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "installed": [ -- УСТАНОВИЛСЯ
            "epel-release", 
```
Напишем playbook, который будет также устанавливать epel-release. Создаем файл с содержимым:
```
---

- name: Install EPEL Repo
  hosts: nginx
  become: true

  tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
        name: epel-release
        state: present
```
Запускаем выпонлением Playbook:
```
sam@yarkozloff:/otus/ansible$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] ***************************************************************************

TASK [Gathering Facts] *****************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standard repo] ************************************************
ok: [nginx]

PLAY RECAP *****************************************************************************************
nginx                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Выполняем команду для установки epel-release и проверяем разницу:
```
sam@yarkozloff:/otus/ansible$ ansible nginx -m yum -a "name=epel-release state=absent" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [ -- НЕ УСТАНОВЛЕНО
            "epel-release"
```
Напишем Playbook для дополнительной установки NGINX. Создаем файл nginx.yml с содержимым:
```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  
  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages
```
Выведем в консоль все теги:
```
sam@yarkozloff:/otus/ansible$ ansible-playbook nginx.yml --list-tags

playbook: nginx.yml

  play #1 (nginx): NGINX | Install and configure NGINX  TAGS: []
      TASK TAGS: [epel-package, nginx-package, packages]
```
Установим только NGINX:
```
sam@yarkozloff:/otus/ansible$ ansible-playbook nginx.yml -t nginx-package
...
PLAY RECAP *****************************************************************************************
nginx                      : ok=2 --УСТАНОВЛЕНО   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Далее добавим шаблон для конфига NGINX, для этого создаем в каталоге templates файл nginx.conf.j2 с содержимым:
```
events {
 worker_connections 1024;
}
http {
 server {
 listen {{ nginx_listen_port }} default_server;
 server_name default_server;
 root /usr/share/nginx/html;
 location / {
 }
 }
}
```
Внесем изменения в файл nginx.yml. Добавим шаблон для конфига NGINX и модуль, который будет копировать этот шаблон на хост (NGINX | Create NGINX config file from template). Добавим секцию vars для того чтобы nginx слушал на порту 8080. Добавим handler для рестарта и включения сервиса при загрузке, в name: reload nginx - перечитываем конфиг. 
* Handlers - это специальные задачи. Они вызываются из других задач ключевым словом notify. Эти задачи срабатывают после выполнения всех задач в сценарии (play). При этом, если несколько задач вызвали одну и ту же задачу через notify, она выполниться только один раз. Handlers описываются в своем подразделе playbook - handlers, так же, как и задачи. Для них используется такой же синтаксис, как и для задач.

Теперь можно запустить playbook и проверить curl-ом что сайт доступен:
```
sam@yarkozloff:/otus/ansible$ ansible-playbook nginx.yml
...
sam@yarkozloff:/otus/ansible$  curl http://192.168.11.150:8080
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">

        html {
...
```
