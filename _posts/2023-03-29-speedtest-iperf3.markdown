---
layout: post
title:  "iperf3 - если нужен speedtest до своей машины"
date:   2023-03-29 09:00 +0300
---

Speedtest всем хорош, кроме того что он позволяет измерить скорость от "себя" до некоего рандомного сервера, который обычно располагается еще и относительно близко к машине, скорость с которой измеряется. Это удобно, если надо оценить своего провайдера, или покрытие сотовой сети в определенном месте.
Но у меня стояла другая задача - надо было измерить скорость между двумя конкретными узлами сети (на самом деле еще и построить мониторинг для этого!). Кажется, что можно было бы быстренько соорудить какой нибудь велосипед, перегоняя трафик из `/dev/random` в `/dev/null` через tcp-сокет, но зачем, если есть iperf3. Он умеет работать в клиент-серверном режиме, поддерживает авторизацию, поэтому можно не опасаться, что скрипт-киддисы будут заливать терабайты sql-инъекций и прочих `Q-injection` в наш сервак.

В этой небольшой заметке я расскажу как поднять iperf3 в режиме сервера, настроить аутентификацию клиентов и запускать iperf3-клиент на локальной машине для измерений точка-точка.

# Генерим аутентификационные данные
Аутентификация iperf3 состояит из двух компонентов.
Во первых, потребуется RSA-ключи для обеспечения шифрования передаваемых кредов пользователя, во вторых нужны эти самые креды.
Ключи легко генерятся при помощи `openssl` (я рискну предположить, что если запускается всякий linux-софт, то есть доступ до консоли). Креды на сервере хранятся в хешированном виде, так что все довольно безопасно и утечки пароля опасаться не стоит, но тем не менее, пароль должен быть длинным, случайным, храниться в менеджере паролей. Имя пользователя - тоже, ну а че бы нет, не руками же его вводить.

## Генерим RSA-ключи
Можно открыть man page, а можно скопирнуть команды ниже.

{% highlight shell %}
$ openssl genrsa -des3 -out private.pem 2048
$ openssl rsa -in private.pem -outform PEM -pubout -out public.pem
$ openssl rsa -in private.pem -out private_not_protected.pem -out-form PEM
{% endhighlight %}

Что теперь есть в каталоге где выполнялись команды: `private.pem` - приватный ключ, зашифрованный, `public.pem` - публичный ключ (он нужен клиентам для подключения), `private_not_protected.pem` - приватный ключ, незашифрованный (он понадобится серверу для работы в автоматическом режиме). Прихраним полученные артефакты в надежном месте до поры до времени.

## Генерим креды пользователя
Вторым компонентом защиты от залива половины интернета в сервак iperf3 являются креды пользователя. Сервер употребляет их в хешированном и соленом виде, клиент требует ввод через stdin или можно передать в переменной окружения, если клиент работает в составе автоматизированного решения (запускается без участия человека).

Опять же, все описано в man, а можно скопирнуть отсюда:

{% highlight shell %}
$ S_USER=P3oUpiPcZ9hgeKRfsyVM S_PASSWD=q7m$H^R3dXuVT3(URy754bY9~q)T^Kut
$ echo -n "{$S_USER}$S_PASSWD" | sha256sum | awk '{ print $1 }'
9c37242bb7f83fef32689cb2005cc341dce3b424c48aa34c6a1cbc9d51c2b44b
{% endhighlight %}

Полученный хешик (`9c37242bb7f83fef32689cb2005cc341dce3b424c48aa34c6a1cbc9d51c2b44b`) вместе с именем пользователя надо прихранить в `csv` файле, который будет использоваться сервером.

{% highlight plaintext %}
P3oUpiPcZ9hgeKRfsyVM,9c37242bb7f83fef32689cb2005cc341dce3b424c48aa34c6a1cbc9d51c2b44b
{% endhighlight %}

Пользователей может быть несколько или даже много. 
Я очень надеюсь, что никто не догадается использовать логопасс из статьи, несмотря на то что они достаточно длинные и надежные :-)

# Поднимаем сервер
А вот сейчас будет контент которого нет в man pages, так что... копируйте отсюда.
Сервак будем крутить systemd юнитом. Для этого напишем вот такой незамысловатый файлик:

{% highlight ini %}
[Unit]
Description=Network speed test server
After=syslog.target network.target auditd.service

[Service]
ExecStart=/usr/bin/iperf3 -s --rsa-private-key-path /etc/iperf3/private_not_protected.pem --authorized-users-path /etc/iperf3/credentials.csv

[Install]
WantedBy=multi-user.target
{% endhighlight %}

И сложим его в `/etc/systemd/system/iperf3.service`.
Внимательный читатель уже мог догадаться, куда надо сложить приватный ключ и файл с кредами. Ну конечно же в `/etc/iperf3/`! Можно сложить конечно куда то еще, в этом случае надо указать правильные пути к файлам через опции `--rsa-private-key-path` (для приватного ключа) и `--authorized-users-path` (для `csv` файлов с солеными и хешированными кредами).
Chmod для этих файлов можно сделать побезопаснее:

{% highlight shell %}
-r--------  1 root root   71 Feb 11 10:24 credentials.csv
-r--------  1 root root 1.7K Feb 11 10:24 private_not_protected.pem
{% endhighlight %}
Приемлемо!

Запустить сервак проще простого:
{% highlight shell %}
# systemctl enable iperf3.service
# systemctl start iperf3.service
{% endhighlight %}
Готовенько! Можно запускать клиента.

# Стартуем клиента
С клиентом все будет проще. Для запуска понадобится публичный ключ и логопасс, которые были созданы ранее. Для запуска "руками" пароль будет запрошен при старте приложения:
{% highlight shell %}
$ iperf3 --username P3oUpiPcZ9hgeKRfsyVM --rsa-public-key-path ./public.pem -c my.test.host.com
Password: Connecting to host my.test.host.com, port 5201
[  5] local 192.168.0.1 port 45540 connected to 42.69.666.777 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  10.4 MBytes  87.0 Mbits/sec    0   1.00 MBytes       
[  5]   1.00-2.00   sec  11.2 MBytes  94.4 Mbits/sec   33    834 KBytes       
[  5]   2.00-3.00   sec  11.2 MBytes  94.4 Mbits/sec  180    639 KBytes       
[  5]   3.00-4.00   sec  10.0 MBytes  83.9 Mbits/sec  105    488 KBytes       
[  5]   4.00-5.00   sec  7.50 MBytes  62.9 Mbits/sec    4    375 KBytes       
[  5]   5.00-6.00   sec  6.25 MBytes  52.4 Mbits/sec    0    403 KBytes       
[  5]   6.00-7.00   sec  8.75 MBytes  73.4 Mbits/sec    0    420 KBytes       
[  5]   7.00-8.00   sec  7.50 MBytes  62.9 Mbits/sec    0    427 KBytes       
[  5]   8.00-9.00   sec  8.75 MBytes  73.4 Mbits/sec    0    428 KBytes       
[  5]   9.00-10.00  sec  8.75 MBytes  73.4 Mbits/sec    0    434 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  90.4 MBytes  75.8 Mbits/sec  322             sender
[  5]   0.00-10.05  sec  88.2 MBytes  73.6 Mbits/sec                  receiver

iperf Done.
{% endhighlight %}

Пароль можно передать через переменную среды, это может понадобиться при запуске из другой софтины или скрипта:
{% highlight shell %}
$ export IPERF3_PASSWORD=q7m$H^R3dXuVT3(URy754bY9~q)T^Kut
$ iperf3 --username P3oUpiPcZ9hgeKRfsyVM --rsa-public-key-path ./public.pem -c my.test.host.com
Connecting to host my.test.host.com, port 5201
[  5] local 192.168.0.1 port 56308 connected to 42.69.666.777 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  9.12 MBytes  76.5 Mbits/sec   67    526 KBytes       
[  5]   1.00-2.00   sec  10.0 MBytes  83.9 Mbits/sec    0    597 KBytes       
[  5]   2.00-3.00   sec  11.2 MBytes  94.4 Mbits/sec    0    648 KBytes       
[  5]   3.00-4.00   sec  8.75 MBytes  73.4 Mbits/sec  111    488 KBytes       
[  5]   4.00-5.00   sec  7.50 MBytes  62.9 Mbits/sec    1    477 KBytes       
[  5]   5.00-6.00   sec  7.50 MBytes  62.9 Mbits/sec    0    403 KBytes       
[  5]   6.00-7.00   sec  7.50 MBytes  62.9 Mbits/sec    0    428 KBytes       
[  5]   7.00-8.00   sec  7.50 MBytes  62.9 Mbits/sec    0    439 KBytes       
[  5]   8.00-9.00   sec  7.50 MBytes  62.9 Mbits/sec    1    328 KBytes       
[  5]   9.00-10.00  sec  6.25 MBytes  52.4 Mbits/sec    0    354 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  82.9 MBytes  69.5 Mbits/sec  180             sender
[  5]   0.00-10.06  sec  80.2 MBytes  66.9 Mbits/sec                  receiver

iperf Done.
{% endhighlight %}

Это если из консоли, а если надо вызвать из дотнета, то переменную надо засунуть в коллекцию `EnvironmentVariables` объекта `ProcessStartInfo`:

{% highlight csharp %}
var start = new ProcessStartInfo()
{
    FileName = "iperf3",
    Arguments = args.ToString(),
    RedirectStandardOutput = true
};
start.EnvironmentVariables.Add("IPERF3_PASSWORD", password);
var proc = new Process { StartInfo = start };
proc.Start();
{% endhighlight %}

Ну вот и все. Это оказалось проще чем казалось:-)