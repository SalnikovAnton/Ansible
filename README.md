#  Автоматизация администрирования. Ansible
### Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:
* необходимо использовать модуль yum/apt
* конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными
* после установки nginx должен быть в режиме enabled в systemd
* должен быть использован notify для старта nginx после установки
* сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible


#### Запускаем вертуальную машину используя [Vagrant-file](https://gist.github.com/lalbrekht/f811ce9a921570b1d95e07a7dbebeb1e "Vagrant-file").  
После поднятия машины узнаем необходимые параметры  
```
anton@anton-VirtualBox:~/Ansible$ vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/anton/Ansible/.vagrant/machines/nginx/virtualbox/private_key
```
Используя эти параметры создадим inventory файл и сохраняем его в папку /Ansible/staging/hosts
```
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=/home/anton/Ansible/.vagrant/machines/nginx/virtualbox/private_key
```
Проверяем что Ansible может управлāть нашим хостом
```
anton@anton-VirtualBox:~/Ansible$ ansible nginx -i staging/hosts -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
Для того, чтобы уйти от постоянной настройки inventory файла необходимо в текущем каталоге создать файл ansible.cfg со следующим
содержанием: 
```
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```
После этого из inventory  файла можно убрать информацию пользователя
```
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```
Еще раз убедимся, что управляемый хост доступен, только теперь без явного указания inventory файла:
```
anton@anton-VirtualBox:~/Ansible$ ansible nginx -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
Посмотрим какое ядро установлено на хосте
```
anton@anton-VirtualBox:~/Ansible$ ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-1127.el7.x86_64
```
Проверим статус сервиса firewalld
```
anton@anton-VirtualBox:~/Ansible$ ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "name": "firewalld",
    "status": {
        "ActiveEnterTimestampMonotonic": "0",
        "ActiveExitTimestampMonotonic": "0",
        "ActiveState": "inactive",      ----->  не активен
...
```
Установим пакет epel-release на наш хост
```
anton@anton-VirtualBox:~/Ansible$ ansible nginx -m yum -a "name=epel-release state=present" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,           ----->  установился
        "changes": {
        "installed": [
            "epel-release"
        ]
    },
...
```
Напишем простой Playbook который будет делать одно действие - а именно: установку пакета epel-release. Создаем файл epel.yml со следующим содержимым
```
---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
    - name: Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
```
Запускаем выполнение Playbook
```
anton@anton-VirtualBox:~/Ansible$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] *********************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] ******************************************************************
ok: [nginx]

PLAY RECAP ***********************************************************************************************************
nginx                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
Затем выполняем 
```
anton@anton-VirtualBox:~/Ansible$ ansible nginx -m yum -a "name=epel-release state=absent" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "epel-release"
        ]
    },
...
```
Затем опять запускаем Playbook и видем разницу в выполнении 
```
anton@anton-VirtualBox:~/Ansible$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] *********************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] ******************************************************************
changed: [nginx]

PLAY RECAP ***********************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
                                        /\
```
Теперь приступим к написанию Playbook-а для установки NGINX. В файле добавились tags для выполнения отдельных действий по Playbook-у
```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080 

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
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template           ==> шаблон для конфига NGINX и модуль, который будет копировать этот шаблон на хост
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx                                            ==> Так же создадим handler для рестарта и включения сервиса при загрузке. 
      systemd:                                                           Теперь каждый раз когда конфиг будет изменяться - сервис перезагрузится.
        name: nginx
        state: restarted
        enabled: yes
    
    - name: reload nginx                                             ==> Перечитываем конфиг
      systemd:
        name: nginx
        state: reloaded
```
Запускаем Playbook nginx.yml
```
anton@anton-VirtualBox:~/Ansible$ ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] ***************************************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [nginx]

TASK [NGINX | Install EPEL Repo package from standart repo] **********************************************************
ok: [nginx]

TASK [NGINX | Install NGINX package from EPEL Repo] ******************************************************************
ok: [nginx]

TASK [NGINX | Create NGINX config file from template] ****************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] ***************************************************************************************
changed: [nginx]

PLAY RECAP ***********************************************************************************************************
nginx                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
Проверим доступность через curl
```
anton@anton-VirtualBox:~/Ansible$ curl http://192.168.56.150:8080
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css"> 

        html {
        background-image:url(img/html-background.png);
        background-color: white;
        font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
        font-size: 0.85em;
        line-height: 1.25em;
        margin: 0 4% 0 4%;
        }

        body {
```






```

```



