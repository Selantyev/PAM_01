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

