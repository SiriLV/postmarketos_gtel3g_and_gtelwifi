# PostmarketOS на Samsung Galaxy Tab E 9.6 (SM-T561 / SM-T560)

Гайд по настройке PostmarketOS с окружением **XFCE**, **i3** или **none** на планшете Samsung Galaxy Tab E.

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

## 🖥️ Настройка окружения XFCE

По умолчанию графическая оболочка может не запускаться.

### 1. Установка и настройка LightDM
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
autologin-user=siri
autologin-user-timeout=0
```

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

## 🔊 Настройка звука (Fix)

Выполните команды последовательно.

**1. Установка пакетов и групп:**
```bash
doas addgroup siri audio
doas apk add pulseaudio alsa-plugins-pulse pavucontrol alsa-utils
```

**2. Конфигурация PulseAudio:**

*   `/etc/pulse/daemon.conf`:
    ```ini
    enable-shm = no
    enable-memfd = no
    exit-idle-time = -1
    ```

*   `/etc/pulse/default.pa`:
    *Найти `load-module module-udev-detect` и заменить на:*
    ```ini
    load-module module-udev-detect tsched=0
    ```

*   `/etc/pulse/client.conf`:
    ```ini
    autospawn = yes
    ```

**3. Настройка микшера (ALSA):**
Выполните блок команд:
```bash
amixer -c 0 cset numid=108 0
amixer -c 0 cset numid=109 0
amixer -c 0 cset numid=32 15
amixer -c 0 cset numid=33 15
amixer -c 0 cset numid=43 7
amixer -c 0 cset numid=44 7
amixer -c 0 cset numid=73 1
amixer -c 0 cset numid=70 1
doas alsactl store
doas apk add xfce4-pulseaudio-plugin
```

Перезагрузитесь и проверьте звук:
```bash
speaker-test -t pink -c 2
```

---

## 🛠️ Другие окружения и софт

### i3 / None
Устанавливаются базовые пакеты. Конфигурация полностью ручная (через SSH). Рекомендуется устанавливать легкие инструменты.

### Полезный софт
*   **btop** — мониторинг ресурсов.
*   **neofetch** — информация о системе.
*   **surf** — легкий браузер (webkit).

**Запуск YouTube в surf (оптимизированный):**
```bash
JSC_useJIT=0 LIBGL_ALWAYS_SOFTWARE=1 WEBKIT_DISABLE_COMPOSITING_MODE=1 \
WEBKIT_FORCE_SANDBOX=0 WEBKIT_USE_SINGLE_WEB_PROCESS=1 DISPLAY=:0 \
surf https://m.youtube.com
```
