---
title: Автообновление устройств в NetBox
last_modified_at: 2023-06-16T13:43:00
excerpt: 'Хочу поделиться написанным на "коленке", но рабочим решением для онбординга новых и обновления уже добавленных в БД NetBox устройств - netbox_resolver. Это python-скрипт, который поможет опросить и добавить/обновить в NetBox одно, либо n-устройств последовательно или с использованием потоков.'
classes: wide
categories:
  - notes
tags:
  - code
  - netbox
  - python
  - bash
  - network
---
**UPD 2023-06-16:** [репозиторий](https://github.com/netdotwork/netbox_resolver){:target="_blank"} в архиве. Скрипт поддерживает только `Huawei VRP`, работает с `netbox 2.8.5`, `netmiko==3.1.0`, `pynetbox==5.0.5`, `textfsm==1.1.0`  и не тестировался с более поздними версиями.
{: .notice--info}

Хочу поделиться написанным "на коленке", но рабочим решением для онбординга новых и обновления уже добавленных в БД NetBox устройств - [netbox_resolver](https://github.com/netdotwork/netbox_resolver){:target="_blank"}. Это python-скрипт, который поможет опросить и добавить/обновить в NetBox одно, либо n-устройств последовательно или с использованием потоков.

Для регулярного обновления скрипт можно добавить в планировщик. Например, так:
- подготовим yaml-конфигуратор с параметрами, `devices.yaml`:

  ```yaml
  - netbox: http://netbox_domain_name_or_ip/
    token: netbox_token
    device_type: huawei
    # 10.1.1.1, 10.1.1.2, 10.1.1.3, 10.1.1.4, 10.1.1.5
    ip_list: ["10.1.1.1-5"]
    threads: True
    # кол-во потоков
    max_workers: 5
    # собираем Inventory для устройств
    inventory: True
    # логин для первой группы устройств
    username: user
    # пароль для первой группы устройств
    password: password
  - netbox: http://netbox_domain_name_or_ip/
    token: netbox_token
    device_type: huawei
    # 10.2.2.140, 10.2.2.148, 10.2.2.156
    ip_list: ["10.2.2.140,148,156"]
    threads: True
    # кол-во потоков
    max_workers: 3
    # собираем Inventory для устройств
    inventory: True
    # логин для второй группы устройств
    username: user1
    # пароль для второй группы устройств
    password: password1
  ```

- пишем `run.py` для работы с одним словарем из `devices.yaml`. `run.py` ждет номер словаря как аргумент:

  ```python
  # для примера используем python3.7 
  # путь до python3.7 в виртуальном окружении
  #!/home/user/virtenvs/py3.7/bin/python3.7
  import yaml
  from vendor_selector import VendorSelector
  # yaml-параметры будем передавать как аргументы
  from sys import argv

  # запускаем как run.py 0, где 0 - словарь из devices.yaml
  with open("devices.yaml") as f:
      # devices - список из словарей
      devices = yaml.safe_load(f)
      params_number = int(argv[1])
      obj = VendorSelector(**devices[params_number])
      obj.send_ip_list()
  ```

- пишем `run.sh` для запуска в crond. `run.sh` будет запускать `run.py` как отдельный фоновый процесс. Чтобы регулировать число таких процессов, добавим в bash-скрипт глобальную переменную (см. `BG_MAX_PROCESS`). Пусть `BG_MAX_PROCESS=2`. В итоге, `run.sh` создаст 2 фоновых процесса, каждый из которых будет опрашивать свою группу устройств из `devices.yaml` с использоанием потоков (см. параметры `threads` `max_workers`):

  ```bash
  #/bin/bash

  # переключаемся на виртуальное окружение с python3.7
  source /home/user/virtenvs/py3.7/bin/activate
  # устанавливаем максимальное число запущенных параллельно процессов
  BG_MAX_PROCESS=2
  BG_PROCESS=0
  MY_PID=$$
  # кол-во словарей с параметрами в devices.yaml = общее кол-во запущенных процессов
  PARAMS_COUNTER=$(( $(awk '/^- /{print $0}' devices.yaml | wc -l) - 1 ))

  for (( i = 0; i <= $PARAMS_COUNTER; i++ ))
  do
          BG_PROCESS=$((`ps ax -Ao ppid | grep $MY_PID | wc -l`))
          while [ $BG_PROCESS -gt $BG_MAX_PROCESS ]
          do
                  BG_PROCESS=$((`ps ax -Ao ppid | grep $MY_PID | wc -l`))
                  sleep 1
          done
          echo TASK NUMBER: $i FROM $PARAMS_COUNTER
          # запускаем run.py с соотв. аргументом как фоновый процесс
          /home/user/virtenvs/py3.7/bin/python /path_to_run.py/run.py $i &
  done
  printf "\nTASKS FINISHING...\n"
  BG_PROCESS=$((`ps ax -Ao ppid | grep $MY_PID | wc -l`))
  # ожидаем пока выполнятся последние 2 процесса
  # уведомляем о завершении всех процессов
  printf "LAST $BG_PROCESS PROCESSES\n"
  while [ $BG_PROCESS -gt 1 ]
  do
          BG_PROCESS=$((`ps ax -Ao ppid | grep $MY_PID | wc -l`))
          #clear
          #echo LAST $BG_PROCESS PROCESSES
          sleep 1
  done
  printf "\nALL TASKS HAVE DONE\n"
  ```

- добавляем `run.sh` в crontab, запускаем каждый день в 00:01, лог сохраняем в `/tmp/cronlog.txt`:

  ```bash
  chmod +x /path_to_run.sh/run.sh

  crontab -e

  # добавляем в crontab
  01      00      *       *       *       bash /path_to_run.sh/run.sh 2>&1 /tmp/cronlog.txt
  ```
