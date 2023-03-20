#  Автоматизация администрирования. Ansible
### Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:
* необходимо использовать модуль yum/apt
* конфигурационные файлы должны бýть взяты из шаблона jinja2 с переменными
* после установки nginx должен бýть в режиме enabled в systemd
* должен бýть использован notify для старта nginx после установки
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
Используя эти параметры создадим inventory файл  

