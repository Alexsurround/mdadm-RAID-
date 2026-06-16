# Areca ARC-1210 + mdadm RAID + Zabbix — полное руководство

Реальный кейс: диагностика, восстановление и мониторинг программного RAID5 (mdadm) на сервере с двумя контроллерами Areca ARC-1210 под управлением Debian 9/10. Контроллер с умирающей кэш-памятью (DRAM Fatal Error) периодически роняет диски и в итоге начинает валить весь сервер.

---

## Содержание

- [Архитектура системы](#архитектура-системы)
- [Диагностика через Areca CLI (cli64)](#диагностика-через-areca-cli-cli64)
- [Интерпретация event log контроллера](#интерпретация-event-log-контроллера)
- [SMART через cli64 (обход FCP-2 проблемы)](#smart-через-cli64-обход-fcp-2-проблемы)
- [mdadm — восстановление inactive/degraded массива](#mdadm--восстановление-inactivedegraded-массива)
- [Определение какой диск на каком контроллере](#определение-какой-диск-на-каком-контроллере)
- [Программное отключение умирающего контроллера](#программное-отключение-умирающего-контроллера)
- [Загрузка системы при развале RAID (удалённый сервер)](#загрузка-системы-при-развале-raid-удалённый-сервер)
- [Zabbix мониторинг Areca контроллера](#zabbix-мониторинг-areca-контроллера)
- [Замена контроллера — рекомендации](#замена-контроллера--рекомендации)
- [Чеклист после неожиданной перезагрузки](#чеклист-после-неожиданной-перезагрузки)

---

## Архитектура системы

```
Сервер Debian 9/10
├── ARC-1210 #1 (03:0e.0) — УМИРАЮЩИЙ (DRAM Fatal Error)
│   ├── Ch1: sda — WD40EFAX 4TB (JBOD)
│   ├── Ch2: sdb — WD40EFAX 4TB (JBOD)
│   ├── Ch3: sdc — WD40EFAX 4TB (JBOD)
│   └── Ch4: sdd — WD1001FALS 1TB (JBOD)
│       └── md54: RAID5 (sda+sdb+sdc), md51: RAID5
│
├── ARC-1210 #2 (06:0e.0) — ЖИВОЙ
│   ├── Ch1: sde
│   ├── Ch2: sdf
│   ├── Ch3: sdg
│   └── Ch4: sdh
│       └── md51: RAID5, md10: RAID10
│
├── Другой контроллер
│   └── sdi → LVM (root, var, swap, tmp) ← система загружается отсюда
│
└── Другие контроллеры
    └── sdj,sdk,sdl,sdm → md10: RAID10
```

> **Важно:** имена дисков (`sda`, `sdb`...) **не фиксированы** и могут меняться после перезагрузки. Всегда проверяйте `lsblk -o NAME,HCTL` для определения реального расположения дисков по контроллерам.

---

## Диагностика через Areca CLI (cli64)

### Запуск

```bash
cd /путь/к/cli64/
sudo ./cli64                    # интерактивный режим
sudo ./cli64 ctrl=1 disk info   # одноразовая команда для контроллера 1
sudo ./cli64 ctrl=2 event info  # одноразовая команда для контроллера 2
```

### Основные команды

```bash
# Список всех контроллеров (показывается при старте)
sudo ./cli64

# Переключение между контроллерами в интерактивном режиме
CLI> set curctrl=2

# Информация о контроллере
sudo ./cli64 ctrl=1 sys info
sudo ./cli64 ctrl=2 sys info

# Состояние физических дисков
sudo ./cli64 ctrl=1 disk info
sudo ./cli64 ctrl=2 disk info

# Лог событий контроллера (самое важное для диагностики)
sudo ./cli64 ctrl=1 event info
sudo ./cli64 ctrl=2 event info

# RAID-сеты на контроллере
sudo ./cli64 ctrl=1 rsf info
sudo ./cli64 ctrl=1 vsf info

# SMART конкретного диска (drv= номер из disk info)
sudo ./cli64 ctrl=1 disk smart drv=1
sudo ./cli64 ctrl=1 disk smart drv=2
sudo ./cli64 ctrl=1 disk smart drv=3
sudo ./cli64 ctrl=1 disk smart drv=4

# Аппаратный мониторинг (температуры, напряжения)
sudo ./cli64 ctrl=1 hw info
```

### Статусы дисков в disk info

| Status | Значение |
|---|---|
| `JBOD` | Диск передаётся ОС напрямую (passthrough) — норма для mdadm |
| `Failed` | Контроллер потерял связь с диском |
| `Free` | Диск есть, но не задействован |
| `N.A.` | Физически не отвечает (кабель/порт/диск) |
| `RaidSet Member` | Диск в аппаратном RAID контроллера |

---

## Интерпретация event log контроллера

```bash
sudo ./cli64 ctrl=1 event info | head -30
```

### Критические паттерны

**Норма:**
```
2026-06-13 06:34:03  H/W MONITOR  Raid Powered On
```

**Тревога — умирающая DRAM:**
```
2026-06-16 11:30:06  H/W MONITOR  DRAM Fatal Error    ← память контроллера
2026-06-16 11:32:33  H/W MONITOR  Raid Powered On     ← контроллер перезагрузился
```

**Катастрофа — связка DRAM Error + Device Failed:**
```
2026-06-14 20:14:38  H/W MONITOR      DRAM Fatal Error   ← ошибка памяти
2026-06-14 20:14:42  H/W MONITOR      DRAM Fatal Error
2026-06-14 20:14:42  IDE Channel #02  Device Failed      ← контроллер "потерял" диск
2026-06-14 20:14:42  Raid Set # 01    RaidSet Degraded   ← массив деградировал
2026-06-14 20:14:42  WD40EFAX-68JH4N0 Volume Failed
```

### Диагноз по event log

| Паттерн | Диагноз | Действие |
|---|---|---|
| Единичный `DRAM Fatal Error` раз в несколько месяцев | Норма для старых карт | Мониторинг |
| `DRAM Fatal Error` несколько раз подряд → `Device Failed` | Деградация кэш-памяти контроллера | Планировать замену |
| Разные каналы (Ch1, Ch2, Ch4) падают в разное время | Подтверждение DRAM, не диски | Срочная замена |
| `DRAM Fatal Error` → перезагрузка всего сервера | Критический отказ, контроллер валит систему | Немедленная замена/отключение |

---

## SMART через cli64 (обход FCP-2 проблемы)

### Проблема

Areca ARC-1210 представляет диски ОС как **Fibre Channel (FCP-2)** устройства. Стандартный `smartctl` показывает бессмыслицу:

```
Transport protocol:  Fibre channel (FCP-2)   ← неверно, реально SATA
Rotation Rate:       10000 rpm                ← неверно, реально 5400
Manufactured in week 30 of year 2002          ← неверно
Device does not support Self Test logging      ← поэтому smartctl -t long не работает
```

### Решение — SMART через cli64

```bash
sudo ./cli64 ctrl=1 disk smart drv=2
```

Вывод показывает реальные атрибуты SMART диска через firmware контроллера.

### Ключевые атрибуты для мониторинга

| ID | Атрибут | Raw value | Тревога |
|---|---|---|---|
| 5 | Reallocated Sector Count | 0 = OK | >0 → диск умирает |
| 197 | Current Pending Sector Count | 0 = OK | >0 → нечитаемые секторы |
| 198 | Offline Uncorrectable | 0 = OK | >0 → Disaster |
| 199 | UltraDMA CRC Error Count | 0 = OK | >0 → проблема кабеля/порта |
| 194 | Temperature | текущая °C | >45 → Warning |
| 9 | Power-on Hours | часы работы | >87600 (10 лет) → плановая замена |

### Пример здорового диска

```
  5 Reallocated Sector Count   200  200  140     0    OK   ← 0 = идеально
197 Current Pending Sector Count 200 200   0     0    OK   ← 0 = идеально
198 Off-line Scan Uncorrectable  100  253   0     0    OK   ← 0 = идеально
199 Ultra DMA CRC Error Count   200  200   0     0    OK   ← 0 = идеально
```

### Пример диска с проблемами кабеля/порта

```
199 Ultra DMA CRC Error Count   200  200   0   131   OK   ← 131 ненулевое!
  9 Power-on Hours Count          1    1   0  100623 OK   ← 11.5 лет работы
```

---

## mdadm — восстановление inactive/degraded массива

### Главное правило

**НИКОГДА не используйте `mdadm --create`** для "пересборки" — это уничтожит данные. Восстановление только через `--stop` + `--assemble`.

### Диагностика состояния

```bash
# Общий статус
cat /proc/mdstat

# Метаданные на каждом физическом диске
sudo mdadm --examine /dev/sda /dev/sdb /dev/sdc

# Детали массива
sudo mdadm --detail /dev/md54
```

### На что смотреть в --examine

```
Events : 128041    ← счётчик событий. У "живых" дисков должен совпадать
Array State : A.A  ← A=active, .=missing. A.A = один диск выпал
Device Role : Active device 0  ← позиция диска в массиве
```

**Диск с меньшим `Events` считается устаревшим (`non-fresh`)** и выкидывается при сборке. В данном кейсе:

```
sda: Events = 128041  ← актуальный
sdb: Events = 107803  ← устарел на 20238 событий → non-fresh → исключается
sdc: Events = 128041  ← актуальный
```

### Пошаговое восстановление

**Шаг 1. Мягкий запуск:**
```bash
sudo mdadm --run /dev/md54
cat /proc/mdstat
```

**Шаг 2. Если inactive — принудительная пересборка:**
```bash
sudo mdadm --stop /dev/md54
# Если "busy":
sudo fuser -k /dev/md54 && sudo mdadm --stop /dev/md54

# Собрать только на актуальных дисках (с одинаковым Events)
# НЕ указывать устаревший диск (sdb в данном примере)
sudo mdadm --assemble --force /dev/md54 /dev/sda /dev/sdc
```

**Шаг 3. Убрать auto-read-only:**
```bash
cat /proc/mdstat
# Если видите (auto-read-only):
sudo mdadm --readwrite /dev/md54
cat /proc/mdstat
# (auto-read-only) должно исчезнуть
```

**Шаг 4. Проверка данных:**
```bash
sudo mkdir -p /mnt/raid_data
sudo mount /dev/md54p1 /mnt/raid_data
ls -la /mnt/raid_data
# Если не вышло — попробовать md54p2
```

**Шаг 5. Бэкап данных** — ОБЯЗАТЕЛЬНО пока массив работает без избыточности (N-1 дисков):
```bash
rsync -av /mnt/raid_data/ /путь/к/бэкапу/
```

**Шаг 6. Возврат диска (rebuild) — только после бэкапа:**
```bash
# Если диск числится как failed:
sudo mdadm /dev/md54 --remove /dev/sdb
# Добавить для полной ресинхронизации:
sudo mdadm /dev/md54 --add /dev/sdb
# Следить за прогрессом:
watch -n 10 cat /proc/mdstat
```

### Про sfdisk при добавлении диска

`sfdisk` (копирование таблицы разделов) **НЕ нужен** если массив собран на целых дисках (не на партициях):

```bash
# Если есть вывод — массив на целом диске, sfdisk не нужен
sudo mdadm --examine /dev/sda

# Если "No such file or directory" — партиций нет, подтверждение
sudo mdadm --examine /dev/sda1
```

`sfdisk -d /dev/sda | sfdisk /dev/sdb` нужен только если массив на партициях (`sda1`, `sdb1`...) и новый диск пришёл чистым.

### Типичные ошибки

**`failed to start array: Input/output error`**
```bash
# 1. Остановить
sudo mdadm --stop /dev/md54
# 2. Проверить Events на всех дисках
sudo mdadm --examine /dev/sda /dev/sdb /dev/sdc
# 3. Собрать только диски с максимальным Events
sudo mdadm --assemble --force /dev/md54 /dev/sda /dev/sdc
```

**`assembled from 1 drive - not enough to start`**

Без `--force` mdadm боится собирать массив с расходящимися Events. Добавьте `--force` и исключите устаревший диск.

**`hot remove failed: No such device or address`**

Диск уже не в массиве (выкинут как non-fresh). `--remove` бессмысленен, просто используйте `--add` для добавления.

**`(auto-read-only)` и recovery не идёт**
```bash
sudo mdadm --readwrite /dev/mdX
```

---

## Определение какой диск на каком контроллере

Имена `sda/sdb/sdc` **меняются при каждой перезагрузке**. Для надёжной идентификации используйте HCTL:

```bash
lsblk -o NAME,HCTL
```

Формат HCTL: `HOST:CHANNEL:TARGET:LUN`

```
sda   0:0:0:0   ← host 0 = контроллер #1 (arcmsr0)
sde   1:0:0:0   ← host 1 = контроллер #2 (arcmsr1)
sdi   2:0:0:0   ← host 2 = другой контроллер (системный диск)
sdj   4:0:0:0   ← host 4 = ещё один контроллер
```

```bash
# PCIe адреса контроллеров
lspci | grep -i areca
# 03:0e.0 ARC-1210  ← контроллер #1 (host 0)
# 06:0e.0 ARC-1210  ← контроллер #2 (host 1)

# Соответствие host → PCIe адрес
ls /sys/class/scsi_host/
cat /sys/class/scsi_host/host0/proc_name  # arcmsr
```

---

## Программное отключение умирающего контроллера

Когда DRAM-ошибки контроллера начали валить весь сервер, но физически вытащить карту невозможно:

### Предварительные условия

Убедитесь что на контроллере #1 нет активно используемых данных:
```bash
lsblk -o NAME,HCTL
mount | grep -E "sda|sdb|sdc|sdd"
cat /proc/mdstat
```

### Последовательность отключения

**Шаг 1. Остановить все массивы на контроллере #1:**
```bash
sudo umount /mnt/raid_data 2>/dev/null
sudo mdadm --stop /dev/md54

# Если есть LVM на дисках контроллера #1:
sudo vgchange -an <имя_vg>
```

**Шаг 2. Убрать диски из ОС:**
```bash
# Диски контроллера #1 (host 0: sda,sdb,sdc,sdd в данном примере)
echo 1 | sudo tee /sys/block/sda/device/delete
echo 1 | sudo tee /sys/block/sdb/device/delete
echo 1 | sudo tee /sys/block/sdc/device/delete
echo 1 | sudo tee /sys/block/sdd/device/delete
```

**Шаг 3. Отключить PCIe устройство:**
```bash
# Адрес контроллера #1 из lspci | grep -i areca
echo 1 | sudo tee /sys/bus/pci/devices/0000:03:0e.0/remove
```

**Шаг 4. Проверить:**
```bash
lsblk
cat /proc/mdstat
dmesg | tail -n 20
# Контроллер #2, системный диск и md10 должны работать нормально
```

> **Важно:** после следующей перезагрузки контроллер снова появится. Это временная мера до физической замены карты. Если в BIOS/UEFI есть опция отключить PCIe слот — используйте её.

---

## Загрузка системы при развале RAID (удалённый сервер)

Критично для серверов без KVM/iDRAC — одна ошибка в fstab = Emergency Mode = недоступный сервер.

### Правило nofail в /etc/fstab

Для всех не-корневых массивов:
```
UUID=<uuid>  /mnt/raid_data  ext4  defaults,nofail,noatime  0  0
```

- `nofail` — systemd продолжит загрузку если устройство не появится
- последнее `0` — отключить fsck (fsck на degraded-массиве может зависнуть интерактивно)

```bash
# Проверить fstab без перезагрузки
sudo mount -a
sudo systemd-analyze verify
```

### Если массив не в fstab (монтируется руками)

Тогда systemd его не ждёт — Emergency Mode по этой причине не возникнет. Это самый безопасный вариант для ненадёжного железа.

### Загрузка с degraded корневым RAID

```bash
# Если / или /boot на RAID:
sudo nano /etc/initramfs-tools/conf.d/mdadm
# BOOT_DEGRADED=true

sudo update-initramfs -u -k all

# Для Debian 9/10 дополнительно:
sudo nano /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet bootdegraded=true"
sudo update-grub

# GRUB на все диски массива:
sudo grub-install /dev/sda
sudo grub-install /dev/sdb
```

### mdadm.conf — когда нужен, когда нет

Если сервер загружается нормально с пустым `/etc/mdadm/mdadm.conf` — **не трогать**. Массивы собираются через udev по UUID на superblock'ах дисков.

`update-initramfs -u -k all` нужен только если массив входит в состав `/` или `/boot`.

---

## Zabbix мониторинг Areca контроллера

### Проблема со стандартным SMART мониторингом

Стандартный Zabbix SMART template использует `smartctl`, который через Areca даёт бессмысленные данные (FCP-2 протокол). Нужен кастомный мониторинг через `cli64`.

### Скрипт мониторинга событий

`/usr/local/bin/areca_check.sh`:

```bash
#!/bin/bash
CTRL=${1:-1}
METRIC=${2:-dram_errors}
CLI=/путь/к/cli64/cli64
EVENTS=$($CLI ctrl=$CTRL event info 2>/dev/null)

case $METRIC in
    dram_errors)
        echo "$EVENTS" | grep -c "DRAM Fatal Error"
        ;;
    device_failed)
        echo "$EVENTS" | grep -c "Device Failed"
        ;;
    raidset_degraded)
        echo "$EVENTS" | grep -c "RaidSet Degraded"
        ;;
    volume_failed)
        echo "$EVENTS" | grep -c "Volume Failed"
        ;;
    all_errors_count)
        echo "$EVENTS" | grep -v "Raid Powered On" \
                        | grep -v "^Date" \
                        | grep -v "^===" \
                        | grep -v "^$" \
                        | grep -c "."
        ;;
    last_event)
        echo "$EVENTS" | grep -v "Raid Powered On" \
                        | grep -v "^Date" \
                        | grep -v "^===" \
                        | grep -v "^$" \
                        | grep -v "GuiErr" \
                        | head -1
        ;;
    last_5_events)
        echo "$EVENTS" | grep -v "Raid Powered On" \
                        | grep -v "^Date" \
                        | grep -v "^===" \
                        | grep -v "^$" \
                        | grep -v "GuiErr" \
                        | head -5 \
                        | tr '\n' '|'
        ;;
    disk_info)
        $CLI ctrl=$CTRL disk info 2>/dev/null \
                        | grep -v "^  #" \
                        | grep -v "^===" \
                        | grep -v "^$" \
                        | grep -v "GuiErr" \
                        | grep -v "Copyright"
        ;;
    disk_failed_count)
        $CLI ctrl=$CTRL disk info 2>/dev/null | grep -c "Failed"
        ;;
esac
```

### Скрипт мониторинга SMART

`/usr/local/bin/areca_smart.sh`:

```bash
#!/bin/bash
# $1 = номер контроллера (1 или 2)
# $2 = номер диска (1,2,3,4...)
# $3 = ID атрибута SMART (5, 197, 198, 199, 194, 9...)

CTRL=${1:-1}
DRV=${2:-1}
ATTR=${3:-5}
CLI=/путь/к/cli64/cli64

$CLI ctrl=$CTRL disk smart drv=$DRV 2>/dev/null \
    | awk -v attr="$ATTR" '$1==attr {print $8}'
```

### Установка

```bash
sudo chmod +x /usr/local/bin/areca_check.sh
sudo chmod +x /usr/local/bin/areca_smart.sh

# Sudo без пароля для zabbix
sudo nano /etc/sudoers.d/zabbix-areca
```

```
zabbix ALL=(ALL) NOPASSWD: /usr/local/bin/areca_check.sh
zabbix ALL=(ALL) NOPASSWD: /usr/local/bin/areca_smart.sh
```

### UserParameter в Zabbix

`/etc/zabbix/zabbix_agentd.d/areca.conf`:

```
UserParameter=areca[*],sudo /usr/local/bin/areca_check.sh $1 $2
UserParameter=areca.smart[*],sudo /usr/local/bin/areca_smart.sh $1 $2 $3
```

```bash
sudo systemctl restart zabbix-agent
```

### Items в Zabbix

**Мониторинг событий контроллера:**

| Key | Type | Interval | Описание |
|---|---|---|---|
| `areca[1,dram_errors]` | Numeric | 5m | Счётчик DRAM Fatal Error, ctrl 1 |
| `areca[1,device_failed]` | Numeric | 5m | Счётчик Device Failed, ctrl 1 |
| `areca[1,raidset_degraded]` | Numeric | 5m | Счётчик RaidSet Degraded, ctrl 1 |
| `areca[1,volume_failed]` | Numeric | 5m | Счётчик Volume Failed, ctrl 1 |
| `areca[1,all_errors_count]` | Numeric | 5m | Все события кроме PowerOn, ctrl 1 |
| `areca[1,disk_failed_count]` | Numeric | 5m | Дисков со статусом Failed сейчас |
| `areca[1,last_event]` | Text | 15m | Последнее событие-ошибка (история) |
| `areca[1,last_5_events]` | Text | 15m | Последние 5 событий через `\|` |
| `areca[2,dram_errors]` | Numeric | 5m | То же для контроллера 2 |
| `areca[2,disk_failed_count]` | Numeric | 5m | То же для контроллера 2 |

**Мониторинг SMART дисков:**

| Key | Type | Interval | Описание |
|---|---|---|---|
| `areca.smart[1,1,5]` | Numeric | 1h | Reallocated Sectors, ctrl1 drv1 |
| `areca.smart[1,1,197]` | Numeric | 1h | Pending Sectors, ctrl1 drv1 |
| `areca.smart[1,1,198]` | Numeric | 1h | Uncorrectable, ctrl1 drv1 |
| `areca.smart[1,1,199]` | Numeric | 1h | CRC Errors, ctrl1 drv1 |
| `areca.smart[1,1,194]` | Numeric | 30m | Temperature, ctrl1 drv1 |
| `areca.smart[1,1,9]` | Numeric | 6h | Power-on Hours, ctrl1 drv1 |
| То же для drv=2,3,4 | | | |

> **Интервал:** для SMART используйте `1h` или дольше — не нагружайте умирающий контроллер частыми опросами. Для `disk_failed_count` и счётчиков событий — `5m` достаточно.

### Триггеры в Zabbix

```
# Новая DRAM-ошибка (счётчик увеличился)
change(/host/areca[1,dram_errors])>0
Severity: High

# Новый Device Failed
change(/host/areca[1,device_failed])>0
Severity: Disaster

# Есть диск Failed прямо сейчас
last(/host/areca[1,disk_failed_count])>0
Severity: Disaster

# Новый RaidSet Degraded
change(/host/areca[1,raidset_degraded])>0
Severity: High

# Температура диска
last(/host/areca.smart[1,1,194])>45
Severity: Warning

# Появились переназначенные секторы
last(/host/areca.smart[1,1,5])>0
Severity: High

# Появились нечитаемые секторы
last(/host/areca.smart[1,1,197])>0
Severity: High
```

### Отключение стандартного smartd для дисков за Areca

```bash
# Smartd через smartctl даёт неверные данные для дисков за Areca
sudo systemctl stop smartd
sudo systemctl disable smartd

# Проверить нет ли Zabbix SMART template прилинкованного к хосту
# Configuration → Hosts → Templates → убрать "Template App Smart" если есть
```

### Проверка работы скриптов

```bash
sudo /usr/local/bin/areca_check.sh 1 dram_errors
sudo /usr/local/bin/areca_check.sh 1 device_failed
sudo /usr/local/bin/areca_check.sh 1 disk_failed_count
sudo /usr/local/bin/areca_check.sh 1 last_event
sudo /usr/local/bin/areca_check.sh 1 last_5_events
sudo /usr/local/bin/areca_smart.sh 1 1 5    # Reallocated, ctrl1, drv1
sudo /usr/local/bin/areca_smart.sh 1 2 197  # Pending, ctrl1, drv2
sudo /usr/local/bin/areca_smart.sh 1 1 194  # Temperature, ctrl1, drv1
```

---

## Замена контроллера — рекомендации

### Почему не Areca снова

ARC-1210 2010 года с firmware V1.49 — хороший контроллер своего времени, но:
- Представляет SATA-диски как FCP-2 (Fibre Channel) → стандартный SMART не работает
- Аппаратный RAID с кэш-памятью = дополнительная точка отказа
- DRAM на карте деградирует после 10+ лет работы

### Что выбрать для замены (mdadm/Linux)

Для Linux software RAID нужен **HBA (Host Bus Adapter)** — контроллер в режиме passthrough, без собственного RAID и кэша.

| Вариант | Цена | Примечание |
|---|---|---|
| IBM M1015 б/у (прошить в LSI IT mode) | 10-25 EUR | Лучшее соотношение цена/качество |
| LSI SAS9211-8i б/у (уже IT mode) | 15-30 EUR | Готов к работе сразу |
| AliExpress LSI SAS3008 новый (IT mode) | 30-50 EUR | Современный, проверять отзывы |
| Adaptec HBA 1100-8i новый | 80-120 EUR | Официальная гарантия |
| Broadcom HBA 9500-8i новый | 150-200 EUR | Топ, избыточно для большинства случаев |

**Не брать:** noname на JMB58x, ASM1061, Marvell 88SE9xxx — нестабильны под нагрузкой.

### Кабели

К LSI SAS контроллерам нужны:
```
SFF-8087 (mini-SAS) to 4x SATA — 2 штуки на 8 портов
```

### Преимущества LSI HBA в IT mode

- SMART работает нативно: `smartctl -a /dev/sda` — реальные данные
- `smartctl -t long /dev/sda` — реальный extended test
- Нет кэш-памяти = нет этой точки отказа
- Отличная поддержка в Debian/Ubuntu десятилетиями
- После замены можно удалить кастомные скрипты areca и использовать стандартные Zabbix SMART templates

---

## Чеклист после неожиданной перезагрузки

```bash
# 1. Причина перезагрузки
last reboot
journalctl -b -1 | tail -n 50
dmesg | grep -E "reboot|panic|oops|watchdog" | tail -n 20

# 2. Ошибки контроллера — ПЕРВЫМ ДЕЛОМ
cd /путь/к/cli64/
sudo ./cli64 ctrl=1 event info | head -20
sudo ./cli64 ctrl=2 event info | head -10

# 3. Какой диск на каком контроллере СЕЙЧАС (имена меняются после ребута!)
lsblk -o NAME,HCTL

# 4. Состояние RAID массивов
cat /proc/mdstat

# 5. Ошибки ядра
dmesg | grep -E "arcmsr|I/O error|abort|fatal|oom" | tail -n 30

# 6. Восстановить массивы если нужно
sudo mdadm --assemble --force /dev/md54 /dev/sdX /dev/sdY
sudo mdadm --readwrite /dev/md54
sudo mdadm --readwrite /dev/md51

# 7. Проверить монтирование
sudo mount -a
df -h
```

### Хронология типичного инцидента с ARC-1210

```
T+0:00  DRAM Fatal Error на контроллере
T+0:01  DRAM Fatal Error (повтор, нарастает)
T+0:02  Device Failed (канал X потерял диск)
T+0:02  RaidSet Degraded / Volume Failed
T+0:02  Контроллер перезагружается → возможно валит весь сервер
T+2:00  Raid Powered On (контроллер снова живой)
T+2:00  Диски снова JBOD (физически целые)
T+2:00  mdadm собирает массив в degraded (N-1 дисков)
```

**После восстановления:** сразу проверить `lsblk -o NAME,HCTL` — имена могли измениться, и массив, который был на контроллере #1, теперь может оказаться на контроллере #2 и наоборот.
