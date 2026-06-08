# Практическая работа «Disaster recovery и Keepalived»

Выполнил: Нестеренко Андрей

---

## Цель работы

В ходе практической работы были выполнены следующие задачи:

* настроено отслеживание состояния интерфейсов в протоколе HSRP;
* проверена отказоустойчивость маршрутизации в Cisco Packet Tracer;
* настроен сервис Keepalived на двух виртуальных машинах Linux;
* настроен веб-сервер `nginx` на двух виртуальных машинах;
* написан Bash-скрипт для проверки доступности веб-сервера;
* настроен перенос плавающего IP-адреса на резервный сервер при отказе основного.

---

## Задание 1

Дана схема для Cisco Packet Tracer, рассматриваемая в лекции.
На данной схеме уже было настроено отслеживание интерфейсов маршрутизаторов `Gi0/1` для нулевой HSRP-группы.
Необходимо было аналогично настроить отслеживание состояния интерфейсов `Gi0/0` для первой HSRP-группы.
Для проверки корректности настройки был разорван кабель между одним из маршрутизаторов и `Switch0`, после чего был запущен `ping` между `PC0` и `Server0`.

---

### Выполнение задания 1

На схеме используются два маршрутизатора:

| Устройство | Интерфейс | IP-адрес         |
| ---------- | --------- | ---------------- |
| Router0    | `Gi0/0`   | `192.168.0.2/24` |
| Router0    | `Gi0/1`   | `192.168.1.2/24` |
| Router1    | `Gi0/0`   | `192.168.0.3/24` |
| Router1    | `Gi0/1`   | `192.168.1.3/24` |

Виртуальные IP-адреса HSRP:

| HSRP-группа | Виртуальный IP | Назначение       |
| ----------- | -------------- | ---------------- |
| Group 0     | `192.168.0.1`  | Шлюз для PC0     |
| Group 1     | `192.168.1.1`  | Шлюз для Server0 |

На маршрутизаторах было настроено отслеживание интерфейса `GigabitEthernet0/0` для HSRP group 1.
Команды настройки на Router0:

```bash
enable
configure terminal
interface gigabitEthernet 0/1
standby 1 track gigabitEthernet 0/0
end
write memory
show standby brief
```

Команды настройки на Router1:

```bash
enable
configure terminal
interface gigabitEthernet 0/1
standby 1 track gigabitEthernet 0/0
end
write memory
show standby brief
```

Дополнительно для демонстрации отказоустойчивости был отключён кабель между Router1 и `Switch0`.
После отключения кабеля интерфейс `GigabitEthernet0/0` на Router1 перешёл в состояние `down/down`, а роль Active для виртуального IP-адреса `192.168.1.1` перешла на Router0.
Файл схемы Cisco Packet Tracer:

[hsrp_advanced_nesterenko_a.pkt](img/hsrp_advanced_nesterenko_a.pkt)

<img src = "img/Pasted image 20260608184637.png" width = 50%>

Скриншот состояния HSRP на Router0:

<img src="img/Pasted image 20260608184517.png" width="50%">

Скриншот состояния HSRP на Router1 после отключения кабеля:

<img src="img/Pasted image 20260608184552.png" width="50%">

Скриншот проверки доступности Server0 с PC0:

<img src="img/Pasted image 20260608184619.png" width="50%">

---

## Задание 2

Необходимо было запустить две виртуальные машины Linux, установить и настроить сервис Keepalived, используя пример конфигурационного файла.
Также требовалось:
* настроить веб-сервер на двух виртуальных машинах;
* написать Bash-скрипт, проверяющий доступность порта веб-сервера и существование файла `index.html`;
* настроить Keepalived так, чтобы он запускал данный скрипт каждые 3 секунды;
* обеспечить перенос виртуального IP-адреса на другой сервер, если скрипт завершается с кодом, отличным от нуля;
* использовать секцию `vrrp_script`;
* приложить Bash-скрипт, конфигурационные файлы Keepalived и скриншот демонстрации переезда плавающего IP-адреса.
---
### Выполнение задания 2

Для выполнения задания были подготовлены две виртуальные машины Ubuntu в VMware Workstation.
Использовались следующие параметры:

| Виртуальная машина | IP-адрес          | Роль               |
| ------------------ | ----------------- | ------------------ |
| VM1                | `192.168.229.137` | MASTER             |
| VM2                | `192.168.229.138` | BACKUP             |
| Floating IP        | `192.168.229.150` | Плавающий IP-адрес |

Сетевой интерфейс на обеих виртуальных машинах:

```text
ens33
```


На обеих виртуальных машинах были установлены пакеты:

```bash
sudo apt update
sudo apt install -y nginx keepalived curl
```

На VM1 был создан файл `/var/www/html/index.html` со следующим содержимым:

```text
Hello from VM1 MASTER
```

На VM2 был создан файл `/var/www/html/index.html` со следующим содержимым:

```text
Hello from VM2 BACKUP
```

---

### Bash-скрипт проверки веб-сервера

Был создан Bash-скрипт `/etc/keepalived/check_web.sh`.

Скрипт проверяет:
* существует ли файл `/var/www/html/index.html`;
* слушает ли веб-сервер порт `80`;
* при ошибке завершает работу с кодом `1`.

Файл скрипта:

[check_web.sh](files/check_web.sh)

Содержимое скрипта:

```bash
#!/bin/bash

WEB_PORT=80
INDEX_FILE="/var/www/html/index.html"

if [ ! -f "$INDEX_FILE" ]; then
    echo "ERROR: index.html not found"
    exit 1
fi

ss -ltn | awk '{print $4}' | grep -qE "(:|\.)${WEB_PORT}$"

if [ $? -ne 0 ]; then
    echo "ERROR: web server port ${WEB_PORT} is not listening"
    exit 1
fi

echo "OK: web server is working and index.html exists"
exit 0
```

Скрипту были выданы права на выполнение:

```bash
sudo chmod +x /etc/keepalived/check_web.sh
```

Скриншот содержимого Bash-скрипта:

<img src="img/Pasted image 20260608195742.png" width="50%">

---

### Конфигурация Keepalived на VM1

На VM1 был настроен Keepalived в роли `MASTER`.
Скриншот конфигурационного файла Keepalived на VM1:

<img src="img/Pasted image 20260608195818.png" width="50%">

Содержимое конфигурационного файла VM1:

```bash
vrrp_script check_web {
    script "/etc/keepalived/check_web.sh"
    interval 3
    fall 1
    rise 1
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 15
    priority 150
    advert_int 1

    unicast_src_ip 192.168.229.137
    unicast_peer {
        192.168.229.138
    }

    virtual_ipaddress {
        192.168.229.150/24
    }

    track_script {
        check_web weight -100
    }
}
```

### Конфигурация Keepalived на VM2

На VM2 был настроен Keepalived в роли `BACKUP`.
Скриншот конфигурационного файла Keepalived на VM2:

<img src="img/Pasted image 20260608195840.png" width="50%">

Содержимое конфигурационного файла VM2:

```bash
vrrp_script check_web {
    script "/etc/keepalived/check_web.sh"
    interval 3
    fall 1
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 15
    priority 100
    advert_int 1

    unicast_src_ip 192.168.229.138
    unicast_peer {
        192.168.229.137
    }

    virtual_ipaddress {
        192.168.229.150/24
    }

    track_script {
        check_web weight -100
    }
}
```


### Проверка работы Keepalived

После запуска Keepalived плавающий IP-адрес `192.168.229.150` находился на VM1.
Проверка выполнялась командой:

```bash
ip a | grep 192.168.229.150
curl 192.168.229.150
```

Результат:

```text
Hello from VM1 MASTER
```

Скриншот работы floating IP на VM1:

<img src="img/Pasted image 20260608195354.png" width="50%">

---

### Проверка отказа веб-сервера

Для проверки отказоустойчивости на VM1 был остановлен веб-сервер `nginx`:

```bash
sudo systemctl stop nginx
```

После этого был вручную запущен проверочный скрипт:

```bash
sudo /etc/keepalived/check_web.sh
```

Скрипт завершился с ошибкой:

```text
ERROR: web server port 80 is not listening
```

После срабатывания проверки плавающий IP-адрес был снят с VM1.
Скриншот ошибки проверки на VM1:

<img src="img/Pasted image 20260608195700.png" width="50%">

---

### Проверка переезда плавающего IP на VM2

После отказа веб-сервера на VM1 плавающий IP-адрес `192.168.229.150` был перенесён на VM2.
Проверка на VM2 выполнялась командами:

```bash
ip a | grep 192.168.229.150
curl 192.168.229.150
```

Результат:

```text
Hello from VM2 BACKUP
```

Скриншот переезда floating IP на VM2:

<img src="img/Pasted image 20260608195719.png" width="50%">

---
## Задание 3*

Дополнительно была изучена возможность Keepalived `track_file`.

Необходимо было написать Bash-скрипт, который меняет значение приоритета внутри файла в зависимости от нагрузки на виртуальную машину. В качестве показателя нагрузки использовался `Load average`.
Также требовалось настроить Keepalived на отслеживание данного файла и проверить, что при высокой нагрузке на сервере MASTER виртуальный IP-адрес переезжает на другой, менее нагруженный сервер.

---

### Выполнение задания 3*

Для выполнения задания использовались те же виртуальные машины:

| Виртуальная машина | IP-адрес          | Роль               |
| ------------------ | ----------------- | ------------------ |
| VM1                | `192.168.229.137` | MASTER             |
| VM2                | `192.168.229.138` | BACKUP             |
| Floating IP        | `192.168.229.150` | Плавающий IP-адрес |

На обеих виртуальных машинах был создан файл:

```text
/etc/keepalived/load_priority
```

В этот файл скрипт записывает значение, которое влияет на итоговый приоритет VRRP-инстанса.

Если нагрузка на сервер низкая, в файл записывается значение:

```text
0
```

Если нагрузка увеличивается, в файл записывается отрицательное значение:

```text
-30
-70
-120
```

Таким образом итоговый приоритет сервера уменьшается, и плавающий IP-адрес может перейти на другой сервер.

---

### Bash-скрипт расчёта нагрузки

Был создан скрипт:

```text
/etc/keepalived/update_load_priority.sh
```

Скриншот Bash-скрипта расчёта нагрузки:

<img src="img/Pasted image 20260608215837.png" width="50%">

---
Содержимое скрипта:

```bash
#!/bin/bash

TRACK_FILE="/etc/keepalived/load_priority"
LOG_FILE="/var/log/keepalived-load-priority.log"

LOAD=$(awk '{print $1}' /proc/loadavg)

if awk "BEGIN {exit !($LOAD < 1.0)}"; then
    VALUE=0
elif awk "BEGIN {exit !($LOAD < 2.0)}"; then
    VALUE=-30
elif awk "BEGIN {exit !($LOAD < 3.0)}"; then
    VALUE=-70
else
    VALUE=-120
fi

echo "$VALUE" > "$TRACK_FILE"
echo "$(date '+%F %T') load=$LOAD value=$VALUE" >> "$LOG_FILE"

exit 0
```

Скрипт анализирует первое значение из файла `/proc/loadavg`, после чего записывает рассчитанное значение в файл `/etc/keepalived/load_priority`.
Скрипту были выданы права на выполнение:

```bash
sudo chmod +x /etc/keepalived/update_load_priority.sh
```

Скрипт был добавлен в `cron` для запуска каждую минуту:

```bash
* * * * * /etc/keepalived/update_load_priority.sh
```


### Конфигурация Keepalived с track_file на VM1

На VM1 был настроен Keepalived с использованием `track_file`.
Скриншот конфигурации Keepalived с `track_file`:

<img src="img/Pasted image 20260608215901.png" width="50%">

---
Содержимое конфигурационного файла VM1:

```bash
track_file load_priority {
    file "/etc/keepalived/load_priority"
    weight 1
    init_file 0
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 15
    priority 150
    advert_int 1

    unicast_src_ip 192.168.229.137
    unicast_peer {
        192.168.229.138
    }

    virtual_ipaddress {
        192.168.229.150/24
    }

    track_file {
        load_priority
    }
}
```


### Конфигурация Keepalived с track_file на VM2
На VM2 был настроен Keepalived с использованием `track_file`.
Содержимое конфигурационного файла VM2:

```bash
track_file load_priority {
    file "/etc/keepalived/load_priority"
    weight 1
    init_file 0
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 15
    priority 100
    advert_int 1

    unicast_src_ip 192.168.229.138
    unicast_peer {
        192.168.229.137
    }

    virtual_ipaddress {
        192.168.229.150/24
    }

    track_file {
        load_priority
    }
}
```

---

### Проверка нормального состояния

В нормальном состоянии значение в файле `/etc/keepalived/load_priority` равно `0`.

Проверка на VM1 выполнялась командами:

```bash
ip a | grep 192.168.229.150
cat /etc/keepalived/load_priority
curl 192.168.229.150
```

Результат:

```text
192.168.229.150
0
Hello from VM1 MASTER
```

Скриншот нормального состояния до нагрузки:

<img src="img/Pasted image 20260608220133.png" width="50%">

---

### Проверка работы при высокой нагрузке

Для создания нагрузки на VM1 была установлена утилита `stress`:

```bash
sudo apt install -y stress
```

После этого была запущена нагрузка на CPU:

```bash
stress --cpu 4 --timeout 180
```

Скрипт `update_load_priority.sh` определил повышенную нагрузку и записал в файл `/etc/keepalived/load_priority` отрицательное значение.

Проверка выполнялась командами:

```bash
cat /proc/loadavg
cat /etc/keepalived/load_priority
tail -n 10 /var/log/keepalived-load-priority.log
ip a | grep 192.168.229.150
```

В логах скрипта было видно изменение значения приоритета:

```text
load=1.59 value=-30
load=2.38 value=-70
```

После уменьшения итогового приоритета VM1 плавающий IP-адрес был снят с VM1.

Скриншот высокой нагрузки на VM1:

<img src="img/Pasted image 20260608220528.png" width="50%">

---

### Проверка переезда floating IP на VM2

После снижения приоритета VM1 плавающий IP-адрес `192.168.229.150` был перенесён на VM2.

Проверка на VM2 выполнялась командами:

```bash
ip a | grep 192.168.229.150
curl 192.168.229.150
```

Результат:

```text
192.168.229.150
Hello from VM2 BACKUP
```

Скриншот переезда floating IP на VM2:

<img src="img/Pasted image 20260608220509.png" width="50%">

---

### Логи Keepalived

Для проверки переключения состояний Keepalived были просмотрены логи на VM2:

```bash
sudo journalctl -u keepalived -n 80 --no-pager
```

В логах было видно, что VM2 переходила в состояние `MASTER`:

```text
Entering MASTER STATE
```

А после восстановления приоритета VM1 возвращалась в состояние `BACKUP`:

```text
Entering BACKUP STATE
```

Скриншот логов Keepalived:

<img src="img/Pasted image 20260608220643.png" width="50%">

---

## Вывод

В ходе выполнения практической работы была настроена отказоустойчивость сетевой инфраструктуры.

В первой части работы был настроен HSRP в Cisco Packet Tracer с отслеживанием состояния интерфейса `GigabitEthernet0/0` для первой HSRP-группы. После отключения кабеля между маршрутизатором и `Switch0` роль Active была перенесена на резервный маршрутизатор, а связь между `PC0` и `Server0` сохранилась.

Во второй части работы был настроен Keepalived на двух виртуальных машинах Ubuntu. С помощью секции `vrrp_script` был подключён Bash-скрипт, который проверяет доступность веб-сервера и наличие файла `index.html`. При остановке `nginx` на VM1 скрипт завершился с ошибкой, после чего плавающий IP-адрес `192.168.229.150` был перенесён на VM2.

В дополнительном задании был настроен механизм `track_file` в Keepalived. Был создан Bash-скрипт, который рассчитывает значение приоритета на основании `Load average` и записывает его в файл `/etc/keepalived/load_priority`. При увеличении нагрузки на VM1 итоговый приоритет сервера снижался, после чего плавающий IP-адрес переходил на VM2. В логах Keepalived был зафиксирован переход VM2 в состояние `MASTER`.
