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
Машина поднялась командой vagrant up, по vagrant ssh есть доступ.
