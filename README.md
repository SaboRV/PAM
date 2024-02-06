## Цель домашнего задания
Научиться создавать пользователей и добавлять им ограничения

## Задача

Научиться создавать пользователей и добавлять им ограничения
1) Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

## Выполнение

Установим систему
Создадим Vagrantfile, в котором будут указаны параметры наших ВМ:
# Описание параметров ВМ
MACHINES = {
  # Имя DV "pam"
  :"pam" => {
              # VM box
              :box_name => "generic/centos8s",
              # Количество ядер CPU
              :cpus => 2,
              # Указываем количество ОЗУ (В Мегабайтах)
              :memory => 1024,
              # Указываем IP-адрес для ВМ
              :ip => "192.168.57.10",
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    # Отключаем сетевую папку
    config.vm.synced_folder ".", "/vagrant", disabled: true
    # Добавляем сетевой интерфейс
    config.vm.network "private_network", ip: boxconfig[:ip]
    # Применяем параметры, указанные выше
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      box.vm.provision "shell", inline: <<-SHELL
          #Разрешаем подключение пользователей по SSH с использованием пароля
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
          #Перезапуск службы SSHD
          systemctl restart sshd.service
  	  SHELL
    end
  end
end

### Настройка запрета для всех пользователей (кроме группы Admin) логина в выходные дни (Праздники не учитываются)

1. Подключаемся к нашей созданной ВМ: vagrant ssh
2. Переходим в root-пользователя: sudo -i
3. Создаём пользователя otusadm и otus: sudo useradd otusadm && sudo useradd otus
4. Создаём пользователям пароли:
[root@pam ~]# echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
Changing password for user otus.
passwd: all authentication tokens updated successfully.

Для примера мы указываем одинаковые пароли для пользователя otus и otusadm

5. Создаём группу admin: sudo groupadd -f admin
6. Добавляем пользователей vagrant,root и otusadm в группу admin:
  usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin

Обратите внимание, что мы просто добавили пользователя otusadm в группу admin. Это не делает пользователя otusadm администратором.

После создания пользователей, нужно проверить, что они могут подключаться по SSH к нашей ВМ. Для этого пытаемся подключиться с хостовой машины: 

Проверяем:

sabo@sabo-virtual-machine:~/LESSONS/PAM$ ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
RSA key fingerprint is SHA256:H0mDZ4cqvc0UutzpQklvr+TFUvyfW8HJLn8TnOuFuvo.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.57.10' (RSA) to the list of known hosts.
otus@192.168.57.10's password: 
[otus@pam ~]$ whoami 
otus
[otus@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.
sabo@sabo-virtual-machine:~/LESSONS/PAM$ ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
[otusadm@pam ~]$ whoami 
otusadm
[otusadm@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.
sabo@sabo-virtual-machine:~/LESSONS/PAM$ 

Мы можем подключиться по SSH под пользователем otus и otusadm

Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни:

7. Проверим, что пользователи root, vagrant и otusadm есть в группе admin:

[root@pam ~]# cat /etc/group | grep admin
admin:x:1003:otusadm,root,vagrant
[root@pam ~]# 

Выберем метод PAM-аутентификации, так как у нас используется только ограничение по времени, то было бы логично использовать метод pam_time, однако, данный метод не работает с локальными группами пользователей, и, получается, что использование данного метода добавит нам большое количество однообразных строк с разными пользователями. В текущей ситуации лучше написать небольшой скрипт контроля и использовать модуль pam_exec

8. Создадим файл-скрипт /usr/local/bin/login.sh

vi /usr/local/bin/login.sh
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


В скрипте подписаны все условия. Скрипт работает по принципу: 
Если сегодня суббота или воскресенье, то нужно проверить, входит ли пользователь в группу admin, если не входит — то подключение запрещено. При любых других вариантах подключение разрешено. 

9. Добавим права на исполнение файла: chmod +x /usr/local/bin/login.sh

10. Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:

[root@pam bin]# cat /etc/pam.d/sshd
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


Т.к. сегодня рабочий день недели, то для проверки в скрипте выставим сегодняшний день недели: вместо "Sat" поставим "Tue".

Пробуем зайти под пользователем otus и получаем ошибку:

sabo@sabo-virtual-machine:~/LESSONS/PAM$ ssh otus@192.168.57.10
otus@192.168.57.10's password: 
Permission denied, please try again.
otus@192.168.57.10's password: 

В логах видим необходимую информацию:

Feb  6 11:55:49 pam sudo[4195]: pam_unix(sudo-i:session): session opened for user root by vagrant(uid=0)
Feb  6 11:59:47 pam sshd[4111]: Received disconnect from 192.168.57.1 port 60026:11: disconnected by user
Feb  6 11:59:47 pam sshd[4111]: Disconnected from user otus 192.168.57.1 port 60026
Feb  6 11:59:47 pam sshd[4103]: pam_unix(sshd:session): session closed for user otus
Feb  6 11:59:54 pam sshd[4294]: pam_exec(sshd:auth): /usr/local/bin/login.sh failed: exit code 1
Feb  6 11:59:56 pam sshd[4294]: Failed password for otus from 192.168.57.1 port 48644 ssh2
Feb  6 11:59:58 pam systemd[4030]: pam_unix(systemd-user:session): session closed for user otus








































