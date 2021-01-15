# PAM_01
PAM homework

**Ограничению доступа пользователей в систему  по  ssh**

**Подготовка стенда**

```
”day” - имеет удаленный доступ каждый день с 8 до 20;
“night” - с 20 до 8;
“friday” - в любое время, если сегодня пятница.
```

1. Создадим пользователей

`useradd day && useradd night && useradd friday`

2. Назначим пароли

```
echo "123qweQWE" | passwd --stdin day && \
echo "123qweQWE" | passwd --stdin night && \
echo "123qweQWE" | passwd --stdin friday

history -d1780
```

3. Проверим записи в pswd и shadow

```
cat /etc/passwd | grep -e day -e night -e friday

day:x:1002:1002::/home/day:/bin/bash
night:x:1003:1003::/home/night:/bin/bash
friday:x:1004:1004::/home/friday:/bin/bash
```

```
cat /etc/shadow | grep -e day -e night -e friday

day:$6$CaDTWC40BbkoCNZE$s/SdNicDTcSxNtWcS.EwDmJUVIGWt850cjsyRQZXEyWMm4V6qUd62HgmXju7H1YDbEAOj5E1uj2kggu3o55Lm0:18639:0:99999:7:::
night:$6$dK/.PyQ/Fx0SZ7.7$Pngs5TDN9xwo1RrF/8DEN3nzPfXSS0o/EasslP3160yQL6d2PcoJ7lh4JtwL4q3Xmw9x4kRqMIo6JGxisz7zH0:18639:0:99999:7:::
friday:$6$bvCY55x8A0P0ROof$GLxXQ8RE4FXFxfayYxj9v2oZMJiDnYV3TwxouNIFE7w3CH.Sz7z6sQWhsyUvoIdkKBhiHpdNgMLdttNnSzUDI1:18639:0:99999:7:::
```

4. Проверим конфиг sshd, вход по паролю должен быть включен

```
grep PasswordAuthentication /etc/ssh/sshd_config

PasswordAuthentication yes
```

**Модуль pam_time**

1. Создадим правила

```
vi /etc/security/time.conf

*;*;day;Al0800-2000
*;*;night;!Al0800-2000
*;*;friday;Fr
```

“*” сервис, к которому применяется правило

"*" имя терминала, к которому применяется правило

имя пользователя, для которого данное правило будет действовать

время, когда правило носит разрешающий характер

2. Включим модуль

```
vi /etc/pam.d/sshd

account    required     pam_nologin.so
account    required     pam_time.so
```

3. Проверим доступ по ssh для пользователей day, night, friday

**Модуль pam_exec.so**

Еще один способ реализовать задачу это выполнить при подключении пользователя скрипт, в котором мы сами обработаем необходимую информацию.

1. Приведем /etc/pam.d/sshd к виду (добавим модуль pam_exec.so и путь до скрипта)

```
vi /etc/pam.d/sshd

account    required     pam_nologin.so
account    required     pam_exec.so /usr/local/bin/test_login.sh
```

2. Создадим скрипт

`touch /usr/local/bin/test_login.sh`

`vi /usr/local/bin/test_login.sh`

`chmod 744 /usr/local/bin/test_login.sh`

```
#!/bin/bash

if [ $PAM_USER = "friday" ]; then
  if [ $(date +%a) = "Fri" ]; then
      exit 0
    else
      exit 1
  fi
fi

hour=$(date +%H)

is_day_hours=$(($(test $hour -ge 8; echo $?)+$(test $hour -lt 20; echo $?)))

if [ $PAM_USER = "day" ]; then
  if [ $is_day_hours -eq 0 ]; then
      exit 0
    else
      exit 1
  fi
fi

if [ $PAM_USER = "night" ]; then
  if [ $is_day_hours -eq 1 ]; then
      exit 0
    else
      exit 1
  fi
fi
```

**Модуль pam_script**

Ещё один способ, использовать модуль pam_script
1. Установим pam_script

`for pkg in epel-release pam_script; do yum install -y $pkg; done`

2. Приведем /etc/pam.d/sshd к виду (добавим модуль pam_script.so и путь до скрипта)

```
vi /etc/pam.d/sshd

account    required     pam_nologin.so
account    required     pam_script.so /usr/local/bin/test_login.sh
```


**Модуль pam_cap.so**

1. Для демонстрации работы модуля установим пакет nmap-ncat

`yum install -y nmap-ncat`

2. Попытаемся прослушать порт 80 пользователем day

```
[day@centos8 ~]$ ncat -l -p 80
Ncat: bind to :::80: Permission denied. QUITTING.
```

Неудачно из-за отсутсвия прав (capabilities)

3. На демостенде отключен SELINUX

```
[day@centos8 ~]$ sestatus
SELinux status:                 disabled
```

4. Приведем /etc/pam.d/sshd к виду (добавим модуль pam_cat.so)

```
vi /etc/pam.d/sshd

auth       include      postlogin
auth       required     pam_cap.so
```

5. Пропишем необходимые права (capabilities) пользователю day. Создадим файл /etc/security/capability.conf

`touch /etc/security/capability.conf`

`echo 'cap_net_bind_service     day' >> /etc/security/capability.conf`

6. Выдадим права программе (capabilities) ncat

`setcap cap_net_bind_service=ei /usr/bin/ncat`

7. Проверим, что пользователь day получил права (capabilities)

```
[day@centos8 ~]$ capsh --print
Current: = cap_net_bind_service+i
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
Ambient set =
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
 secure-no-ambient-raise: no (unlocked)
uid=1002(day)
gid=1002(day)
groups=1002(day)
```

8. Попытаемся прослушать порт 80 пользователем day

`[day@centos8 ~]$ ncat -l -p 80`

Программа удачно запустилась и слушает порт 80

9. Откроем еще одну консоль пользователя day и отправим сообщение

```
[day@centos8 ~]$ echo "Make Linux great again!" > \
/dev/tcp/127.0.0.7/80
```

10. Увидим сообщение в первой консоле

`Make Linux great again!`

**SUDO предоставление прав рута пользователю**

Несколько вариантов:

●пользователь заносится в группу wheel;

●для него создается отдельный файл в /etc/sudoers.d/;

●отдельная строка в  /etc/sudoers.

1. Добавим пользователя day в группу wheel

```
[root@centos8 ~]# usermod -G wheel day

[day@centos8 ~]$ cat /etc/passwd |grep day
day:x:1002:1002::/home/day:/bin/bash

[day@centos8 ~]$ cat /etc/group |grep day
wheel:x:10:day
```

2. Проверим

```
[day@centos8 ~]$ sudo -i

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for day:
[root@centos8 ~]#
```

3. Удалим пользователя day из группы wheel

```
[root@centos8 ~]# gpasswd -d day wheel
Removing user day from group wheel
```

4. Создадим файл в /etc/sudoers.d

`touch /etc/sudoers.d/day`

5. Добавим строку в файл /etc/sudoers.d/day

```
# без пароля
echo 'day        ALL=(ALL)       NOPASSWD: ALL' >> /etc/sudoers.d/day`
```

или

```
# с паролем
echo 'day        ALL=(ALL)       ALL' >> /etc/sudoers.d/day
```

6. Проверим

```
[day@centos8 ~]$ sudo su
[root@centos8 ~]#
```







