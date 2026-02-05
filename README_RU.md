# PostmarketOS на Samsung Galaxy Tab E 9.6 (SM-T561 / SM-T560)

Универсальный гайд по настройке PostmarketOS на планшете Samsung Galaxy Tab E.

> [!NOTE]
> **Синий экран при загрузке:** Если вы видите синий экран с "битым" логотипом Samsung — это нормально. Система загружается, доступ по SSH открыт.

## 📱 Состояние оборудования

| Функция | Статус | Примечание |
| :--- | :---: | :--- |
| **Экран / Тачскрин** | ✅ | Требуется калибровка (см. ниже) |
| **WiFi** | ✅ | |
| **USB (OTG/ADB)** | ✅ | Клавиатура обязательна для первичной настройки |
| **Звук** | ⚠️ | Работает после патча (только T561?) |
| **Аккумулятор** | ✅ | |
| **Хард-кнопки** | ✅ | |
| **Сон (Suspend)** | ✅ | |
| **Bluetooth** | ❌/❓ | Требует тестов |
| **Камера** | ❌/❓ | |
| **GPS / 3G** | ❌ | |

---

## 🚀 Начало работы (SSH)

По умолчанию доступ к устройству осуществляется через USB-сеть.

**Данные для входа:**
*   **IP:** `172.16.42.1`
*   **User:** `user`
*   **Password:** `1`

```bash
ssh user@172.16.42.1
```

### Подключение к WiFi
```bash
doas nmcli device wifi list
doas nmcli device wifi connect "SSID_NAME" password "YOUR_PASSWORD"
doas apk update && doas apk upgrade
```

---

## 🖥️ Рекомендуемые окружения рабочего стола

Выбор окружения влияет на производительность и удобство использования:

1. **LXQt** — для тех, кому лень настраивать. Легкое, работает из коробки
2. **i3wm** — для тех, кто желает эталон. Плиточный менеджер, максимальная производительность
3. **MATE** — не плохой вариант, баланс простоты и функционала
4. **XFCE4** — работает, но требует фикс запуска + нет отображения приложений
5. **Openbox** — минималистичный оконный менеджер
6. **none** — для опытных пользователей, ручная настройка

---

## 🖥️ Настройка графического окружения (MATE / XFCE)

По умолчанию графическая оболочка может не запускаться.

### 1. Установка и настройка LightDM (для MATE / XFCE)

> **Примечание:** LightDM необходим для автозапуска графической оболочки в MATE и XFCE.

```bash
doas apk add lightdm lightdm-gtk-greeter
doas rc-update add lightdm default
```

Редактируем конфиг `/etc/lightdm/lightdm.conf`.
Внесите изменения в секции `[LightDM]` и `[Seat:*]`:

```ini
[LightDM]
logind-check-graphical=false

[Seat:*]
user-session=xfce
autologin-user=user
autologin-user-timeout=0
```

> Для MATE замените `user-session=xfce` на `user-session=mate`

Перезапустите сервис:
```bash
doas service lightdm restart
```

### 2. Поворот экрана и тачскрина
Создаем директорию для конфигов Xorg:
```bash
doas mkdir -p /etc/X11/xorg.conf.d
```

**Файл `/etc/X11/xorg.conf.d/10-monitor.conf` (Поворот картинки):**
```xorg
Section "Device"
    Identifier "LCD"
    Driver "fbdev"
    Option "Rotate" "CW"
EndSection
```

**Файл `/etc/X11/xorg.conf.d/99-calibration.conf` (Калибровка тача):**
```xorg
Section "InputClass"
    Identifier "calibration"
    MatchProduct "sec_touchscreen"
    Option "TransformationMatrix" "0 1 0 -1 0 1 0 0 1"
EndSection
```

Примените изменения перезагрузкой:
```bash
doas reboot
```

---

## 🔊 Настройка звука

### Установка и настройка аудиосистемы

**Шаг 1. Установка пакетов и добавление в группы:**
```bash
doas addgroup user audio
doas addgroup user video
doas addgroup user input
doas apk add pulseaudio alsa-plugins-pulse pavucontrol alsa-utils
```

**Шаг 2. Конфигурация PulseAudio**

Перезапишите `/etc/pulse/daemon.conf`, и вставьте этот код:
```ini
enable-shm = no
enable-memfd = no
exit-idle-time = -1
resample-method = trivial
flat-volumes = no
allow-module-loading = yes
```

Также файл `/etc/pulse/default.pa`:
```ini
# Протоколы
load-module module-native-protocol-unix auth-anonymous=1 socket=/run/user/10000/pulse/native
# load-module module-dbus-protocol # Отключено во избежание ошибок

# Восстановление
load-module module-device-restore
load-module module-stream-restore
load-module module-card-restore
load-module module-augment-properties
load-module module-switch-on-port-available

# ЗАГРУЗКА ДРАЙВЕРА SPREADTRUM (по имени)
load-module module-alsa-sink device=hw:sprdphone tsched=0 mmap=0 sink_name=sprd_out
load-module module-alsa-source device=hw:sprdphone tsched=0 mmap=0 source_name=sprd_in

# Умолчания
set-default-sink sprd_out
set-default-source sprd_in

# Дополнительно
load-module module-always-sink
load-module module-position-event-sounds
load-module module-role-cork
```

И файл `/etc/pulse/client.conf`:
```ini
enable-shm = no
enable-memfd = no
autospawn = yes
```

**Шаг 4. Перезапуск**

```bash
doas killall -9 pulseaudio 2>/dev/null
rm -rf /run/user/10000/pulse/*
pulseaudio -D
```

**Шаг 5. Узнать номер карты**
Вам нужно знать номер карты (-c X), чтобы применять настройки микшера.
Введите:

```bash
aplay -l | grep sprdphone
```
Цифра после слова card — это ваш номер.
Если там card 0 -> используйте в командах -c 0
Если там card 2 -> используйте в командах -c 2

Далее в инструкции я использую -c 2, так как это мой последний вариант. Если карта изменится, поменяйте цифру.

**Шаг 6. Настройка Микшера (Amixer)**
Это полная последовательность включения всего тракта для Spreadtrum SC7730/8830. Выполняйте блоками.

Блок 1: Включаем "Рубильники" питания
```bash
amixer -c 2 sset 'Speaker Function' on
amixer -c 2 sset 'Speaker2 Function' on
amixer -c 2 sset 'VBC DACL DG' on
amixer -c 2 sset 'VBC DACR DG' on
```

Блок 2: Маршрутизация (Соединяем провода). Без этого звук теряется внутри чипа.
```bash
amixer -c 2 sset 'SPKL Mixer DACLSPKL' on
amixer -c 2 sset 'SPKR Mixer DACRSPKR' on
amixer -c 2 sset 'VBC DA IIS Mux' 'sprd-codec'
amixer -c 2 sset 'VBC' 'ap'
```

Блок 3: Громкость
```bash
amixer -c 2 sset 'SPKL' 100%
amixer -c 2 sset 'SPKR' 100%
amixer -c 2 sset 'DACL' 100%
amixer -c 2 sset 'DACR' 100%
```

Блок 4: Заглушение (Mute) - Самый хитрый момент
Здесь пробуем два варианта.

Вариант А (Стандартный):
```bash
amixer -c 2 sset 'Speaker Mute' off
amixer -c 2 sset 'Speaker2 Mute' off
#Проверьте звук командой
paplay /usr/share/sounds/alsa/Front_Center.wav
```

Вариант Б (Если звука всё ещё нет):
```bash
amixer -c 2 sset 'Speaker Mute' on
amixer -c 2 sset 'Speaker2 Mute' on
#Проверьте звук командой
paplay /usr/share/sounds/alsa/Front_Center.wav
```

**Шаг 7. Сохранение**
После того как звук заработал, обязательно сохраните настройки:
```bash
doas alsactl store
doas rc-update add alsa default
```

> **Важно!** Без сохранения все настройки сбросятся после перезагрузки.

---

## 💾 Монтирование microSD карты и USB-накопителей

### Ручное монтирование через терминал

**Шаг 1. Определяем имя диска**

Вставьте карту памяти или USB-накопитель и выполните:
```bash
lsblk
```

> Если команда не найдена: `doas apk add util-linux`

**Как различить устройства на Galaxy Tab E:**
- `mmcblk0` — внутренняя память планшета (не трогать)
- `mmcblk1p1` — обычно SD-карта
- `sda1` (или `sdb1`) — USB-флешка через OTG

**Шаг 2. Создаем точку монтирования**
```bash
doas mkdir -p /mnt/usb
```

**Шаг 3. Монтируем устройство**

Для USB-накопителя:
```bash
doas mount /dev/sda1 /mnt/usb
```

Для SD-карты:
```bash
doas mount /dev/mmcblk1p1 /mnt/usb
```

Теперь файлы доступны в `/mnt/usb`

**Шаг 4. Безопасное извлечение**

Перед отключением устройства выполните:
```bash
doas umount /mnt/usb
```

### Поддержка exFAT и NTFS

Для работы с Windows-форматами установите драйверы:
```bash
doas apk add exfat-fuse ntfs-3g
```

Для exFAT можно использовать:
```bash
doas mount.exfat-fuse /dev/sda1 /mnt/usb
```

### Автоматическое монтирование при загрузке

**Шаг 1. Узнайте UUID устройства:**
```bash
doas blkid
```

Найдите строку для вашего устройства (например, `/dev/mmcblk1p1`) и скопируйте UUID.

**Шаг 2. Отредактируйте `/etc/fstab`:**
```bash
doas nano /etc/fstab
```

Добавьте строку (замените UUID и тип файловой системы):
```
UUID=ВАШ-UUID-ЗДЕСЬ  /mnt/usb  exfat  defaults,nofail  0  0
```

Примеры для разных файловых систем:
- **exFAT:** `exfat  defaults,nofail  0  0`
- **NTFS:** `ntfs-3g  defaults,nofail  0  0`
- **ext4:** `ext4  defaults,nofail  0  0`
- **FAT32:** `vfat  defaults,nofail  0  0`

> Параметр `nofail` предотвращает ошибки загрузки, если карта не вставлена.

**Шаг 3. Проверьте монтирование:**
```bash
doas mount -a
```

Если ошибок нет, устройство будет монтироваться автоматически при каждой загрузке.

---

## 🛠️ Полезное ПО

### Рекомендуемые приложения

Установка основных инструментов:
```bash
doas apk add firefox-esr btop neofetch fish micro ranger qterminal
```

**Описание:**
- **firefox-esr** — браузер (стабильная версия Firefox)
- **btop** — мониторинг системных ресурсов
- **neofetch** — красивый вывод информации о системе
- **fish** — удобная оболочка командной строки
- **micro** — простой текстовый редактор для кодинга
- **ranger** — файловый менеджер в терминале
- **qterminal** — легкий эмулятор терминала

### Смена оболочки на Fish (опционально)

Для более комфортной работы в терминале:
```bash
chsh -s /usr/bin/fish
```

> После этого перезагрузитесь.