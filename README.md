# mdadm RAID — шпаргалка по диагностике и восстановлению

Сборник команд и сценариев для диагностики, восстановления и предотвращения проблем с программным RAID (mdadm) в Debian/Ubuntu. Основано на реальном кейсе восстановления RAID5

---

## Содержание

- [Базовая диагностика](#базовая-диагностика)
- [Запуск деградированного / inactive массива](#запуск-деградированного--inactive-массива)
- [Возврат диска в массив (rebuild)](#возврат-диска-в-массив-rebuild)
- [Загрузка системы при развале RAID](#загрузка-системы-при-развале-raid)
- [Типичные ошибки и их разбор](#типичные-ошибки-и-их-разбор)
- [Полезные команды на каждый день](#полезные-команды-на-каждый-день)
- [Чеклист перед перезагрузкой удалённого сервера](#чеклист-перед-перезагрузкой-удалённого-сервера)

---

## Базовая диагностика

```bash
# Общий статус всех массивов
cat /proc/mdstat

# Детальная информация по конкретному массиву
sudo mdadm --detail /dev/md54

# Метаданные на каждом физическом диске/разделе
sudo mdadm --examine /dev/sda /dev/sdb /dev/sdc

# Список блочных устройств и точек монтирования
lsblk -f

# Текущие точки монтирования конкретного массива
mount | grep md54

# UUID и тип ФС разделов внутри массива
sudo blkid /dev/md54p1 /dev/md54p2

# Последние сообщения ядра по RAID
dmesg | tail -n 40
dmesg | grep -i md
```

### На что смотреть в `--examine`

| Поле | Что означает |
|---|---|
| `Events` | Счётчик событий. У "живых" дисков должен совпадать (или быть очень близким). Диск с меньшим значением считается устаревшим (`non-fresh`) и выкидывается. |
| `Array State` | `A` — active, `.` — missing/failed, `R` — replacing. `AAA` = всё ок, `A.A` = один диск выпал. |
| `Update Time` | Время последней записи на диск. Сильное расхождение = диск отстал. |
| `Device Role` | Позиция диска в массиве (Active device 0/1/2...). |

---

## Запуск деградированного / inactive массива

### Главное правило
**Никогда не используйте `mdadm --create`** для "пересборки" существующего массива — это уничтожит метаданные и данные. Восстановление — это всегда `--stop` + `--assemble`.

### Шаг 1. Попытка мягкого запуска
```bash
sudo mdadm --run /dev/md54
```
Если статус стал `clean`/`degraded`/`active` — успех, дальше переходите к монтированию.

### Шаг 2. Если массив "заклинило" в `inactive`
```bash
# Остановить (сбросить состояние в памяти ядра)
sudo mdadm --stop /dev/md54

# Если "busy" — снять блокировку
sudo fuser -k /dev/md54
sudo mdadm --stop /dev/md54
```

### Шаг 3. Принудительная пересборка из живых дисков
```bash
# Указывайте только диски с одинаковым (или максимальным) Events
sudo mdadm --assemble --force /dev/md54 /dev/sda /dev/sdc
```

> Диск с устаревшим `Events` (например, на 20000+ меньше остальных) указывать **не нужно** — mdadm сам "выкинет" его как `non-fresh`, а если указать насильно, может заблокировать сборку вообще.

### Шаг 4. Если массив поднялся, но в `auto-read-only`
Это значит ядро собрало массив, но не пишет на диск — recovery не начнётся.
```bash
cat /proc/mdstat
# md54 : active (auto-read-only) raid5 ...

sudo mdadm --readwrite /dev/md54

cat /proc/mdstat
# (auto-read-only) должно пропасть
```

### Шаг 5. Проверка и монтирование данных
```bash
sudo mkdir -p /mnt/raid_data
sudo mount /dev/md54p1 /mnt/raid_data
ls -la /mnt/raid_data
# если не вышло — попробовать md54p2
```

**Сделайте резервную копию критичных данных сейчас**, пока массив работает без избыточности (N-1 дисков).

---

## Возврат диска в массив (rebuild)

Когда данные проверены и есть бэкап:

```bash
# Если диск числится как failed/нет в массиве — убрать старую запись
sudo mdadm /dev/md54 --remove /dev/sdb

# может быть read sudo mdadm --readwrite /dev/md54

# Добавить диск для полной ресинхронизации
sudo mdadm /dev/md54 --add /dev/sdb

# Следить за прогрессом recovery
cat /proc/mdstat
watch cat /proc/mdstat
```

### Про разбиение на партиции (sfdisk)
Если массив собран на **целых дисках** (`/dev/sda`, `/dev/sdb`, а не `/dev/sda1`) — `sfdisk`/копирование таблицы разделов **не требуется**. `mdadm --add` сам пишет суперблок RAID на чистый диск.

`sfdisk -d /dev/sda | sfdisk /dev/sdb` нужен только если:
- массив собран на партициях (`sda1`, `sdb1`, ...), **и**
- новый диск пришёл совсем чистым, без таблицы разделов.

Проверить, на чём собран массив:
```bash
sudo mdadm --examine /dev/sda     # если есть вывод — массив на целом диске
sudo mdadm --examine /dev/sda1    # если "No such file or directory" — партиций нет
```

---

## Загрузка системы при развале RAID

Критично для удалённых серверов без KVM — если этого не сделать, сервер может зависнуть в `BusyBox (initramfs)` или `systemd Emergency Mode`, требующем пароль root с локальной консоли.

### 1. Опция `nofail` в `/etc/fstab`

Для **не-корневых** массивов (хранилище данных, не `/` и не `/boot`):

```
UUID=<uuid_md54p1>  /mnt/raid_data1  ext4  defaults,nofail,noatime  0  0
```

- `nofail` — systemd не уйдёт в Emergency Mode, если устройство не появится при загрузке.
- последнее число (`pass`) = `0` — отключает fsck при загрузке для этого раздела (fsck на degraded-массиве может потребовать интерактивного подтверждения и подвесить загрузку).

> Если раздел вообще не прописан в `/etc/fstab` (монтируется руками) — systemd его не ждёт, и Emergency Mode по этой причине не возникнет независимо от состояния RAID.

### 2. Если корневой раздел (`/`) или `/boot` на RAID1

```bash
# Разрешить загрузку с деградированного массива
sudo nano /etc/initramfs-tools/conf.d/mdadm
# содержимое:
BOOT_DEGRADED=true

# Пересобрать initramfs
sudo update-initramfs -u -k all

# Для Debian 9/10 дополнительно — параметр в GRUB
sudo nano /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet bootdegraded=true"
sudo update-grub

# Установить GRUB на все диски массива
sudo grub-install /dev/sda
sudo grub-install /dev/sdb
```

### 3. Применить fstab без перезагрузки (проверка перед ребутом)

```bash
# Смонтирует всё новое из fstab — синтаксическая ошибка вылезет сразу
sudo mount -a

# Проверка корректности юнитов systemd, генерируемых из fstab
sudo systemd-analyze verify
```

### 4. mdadm.conf и initramfs — когда это нужно

| Ситуация | Нужно ли обновлять mdadm.conf / initramfs |
|---|---|
| Массив собирается через udev/incremental assembly по UUID на superblock'ах (mdadm.conf пустой, и всё работает) | **Нет** — ничего не трогать |
| md-массив — часть `/` или `/boot` | Да — initramfs должен знать актуальный UUID/состав |
| Изменился состав дисков массива данных, и он явно прописан в mdadm.conf | Желательно обновить запись (Array UUID не меняется при замене диска, поэтому старая запись обычно остаётся валидной) |

Если решили обновить — **не дублируйте** строки:
```bash
sudo cp /etc/mdadm/mdadm.conf /etc/mdadm/mdadm.conf.bak
grep -v "md54" /etc/mdadm/mdadm.conf > /tmp/mdadm.conf.new
sudo mdadm --detail --scan >> /tmp/mdadm.conf.new
sudo cp /tmp/mdadm.conf.new /etc/mdadm/mdadm.conf
```

---

## Типичные ошибки и их разбор

### `mdadm: failed to start array /dev/mdX: Input/output error`
Ядро не может прочитать суперблоки, либо дисков меньше минимума (для RAID5 нужно N-1 живых).

1. `sudo mdadm --stop /dev/mdX` (если "busy" → `fuser -k`)
2. `sudo mdadm --examine /dev/sd*` — сравнить `Events` и `Array State`
3. `sudo mdadm --assemble --force /dev/mdX <живые диски без устаревшего>`
4. Если снова ошибка — `dmesg | tail -n 40`, искать `I/O error`, `Buffer I/O error`, `Bad block` — это аппаратный сбой диска/кабеля/контроллера.

### `mdadm: /dev/mdX assembled from 1 drive - not enough to start the array`
Без `--force` mdadm не рискует собирать массив с расходящимися `Events`. Добавьте `--force` и указывайте только диски с максимальным/совпадающим `Events`.

### `md: kicking non-fresh sdX from array!`
Диск `sdX` имеет более старый `Events` и был исключён ядром автоматически — это нормально для безопасной деградированной сборки. Массив поднимется на оставшихся дисках.

### `mdadm: hot remove failed for /dev/sdX: No such device or address`
Диск уже не входит в массив (был выкинут как non-fresh) — `--remove` для него бессмысленен. Сначала `--stop`, затем `--assemble --force` без этого диска.

### Массив поднялся, но `(auto-read-only)` и recovery не идёт
```bash
sudo mdadm --readwrite /dev/mdX
```

### `used devices: <none>` в `/proc/mdstat`
Массив не собрался, несмотря на попытку. Проверить точные имена устройств через `lsblk`, остановить, собрать заново с `--force` и точным списком разделов/дисков.

### systemd Emergency Mode при загрузке (после отказа диска)
Добавить `nofail` в `/etc/fstab` для соответствующей строки, `update-initramfs -u -k all` если устройство в составе initramfs.

---

## Полезные команды на каждый день

```bash
# Сводный статус
cat /proc/mdstat

# Подробности по массиву (состояние, активные/failed диски, % recovery)
sudo mdadm --detail /dev/md54

# Мониторинг прогресса recovery в реальном времени
watch -n 5 cat /proc/mdstat

# Проверка SMART конкретного диска (выявление "умирающего" диска до его выпадения)
sudo smartctl -a /dev/sda | grep -E "Reallocated|Pending|Uncorrectable|Health"

# Принудительная проверка консистентности (без записи)
echo check | sudo tee /sys/block/md54/md/sync_action
cat /sys/block/md54/md/mismatch_cnt

# Остановить проверку
echo idle | sudo tee /sys/block/md54/md/sync_action

# Список всех md-устройств и их состояния через udev
ls -la /dev/md/
```

---

## Чеклист перед перезагрузкой удалённого сервера

- [ ] `cat /proc/mdstat` — все массивы `active`, без `(auto-read-only)`, без degraded-меток (`_`) — если только это не ожидаемо.
- [ ] Все non-root массивы из `/etc/fstab` имеют `nofail` и `pass=0`.
- [ ] `sudo mount -a` выполнен без ошибок.
- [ ] `sudo systemd-analyze verify` не выдал критичных ошибок по точкам монтирования.
- [ ] Если массив — часть `/` или `/boot`: `BOOT_DEGRADED=true` выставлен, `update-initramfs -u -k all` выполнен, GRUB установлен на все диски массива.
- [ ] Данные с проблемного массива скопированы/забэкаплены.
