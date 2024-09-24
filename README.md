# Hisense A7cc
Здесь собрана некоторая полезная информация по данному девайсу

> [!WARNING]
> ### Все что вы делаете - вы делаете на свой страх и риск! Я ни за что не отвечаю! :no_mobile_phones:
---
- [Получение Root из-под Windows](README.md#root-из-под-windows)
- [Обновление Magisk из-под Windows](README.md#обновление-magisk-из-под-windows)

# Root из-под Windows
> [!IMPORTANT]
> ### Для получения root, загрузчик должен быть разблокирован.

Включаем режим разработчика:\
Переходим в настройки телефона Settings -> About Phone -> и тапаем несколько раз подряд на Kernel version\
Далее идем в Settings -> System & Updates -> Developer options и включаем USB Debug\
После этого нужно подключить телефон по проводу к ПК и подтвердить на телефоне, что нужно всегда(поставить галку) доверять этому устройству

На ПК нам понадобится установить
- [ADB](https://developer.android.com/tools/adb?hl=ru)
- [OpenSSL](https://slproweb.com/products/Win32OpenSSL.html)
- [Python](https://www.python.org/downloads/)
- [Драйвера для ADB(если, вдруг, телефон не виден)](https://developer.android.com/studio/run/win-usb)
- Изменение настроек USB в рееестре, в случае использования USB 3.0 

Для облегчения задачи рекомендую установить пакетный менеджер [choco](https://chocolatey.org/install).\
Вся установка **choco** сводится к запуску PowerShell от имени Администратора и выполнению в ней команды, с последующим нажатием клавиши enter
```PowerShell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
Import-Module $env:ChocolateyInstall\helpers\chocolateyProfile.psm1
``` 
Затем, чтобы установить софт перечисленный выше, в том же окне вводим и жмем enter
```PowerShell
choco install adb openssl curl -y 
choco install -y python312 --override --install-arguments '/quiet InstallAllUsers=1 PrependPath=1 TargetDir=""C:\Program Files\Python3""'
refreshenv
``` 
Далее фикс fastboot под USB 3.0, все в том же окне PowerShell
```PowerShell
$RegistryPath = "HKLM:\SYSTEM\CurrentControlSet\Control\usbflags\18D1D00D0100"
New-ItemProperty -Path $RegistryPath -Name "osvc" -Value ([byte[]](0x00,0x00)) -PropertyType Binary -Force
New-ItemProperty -Path $RegistryPath -Name "SkipContainerIdQuery" -Value ([byte[]](0x01,0x00,0x00,0x00)) -PropertyType Binary -Force
New-ItemProperty -Path $RegistryPath -Name "SkipBOSDescriptorQuery" -Value ([byte[]](0x01,0x00,0x00,0x00)) -PropertyType Binary -Force
``` 
#### Устанавливаем Magisk
Копируем свой boot.img(оригинал обязателно сохраните на ПК) в телефон.\
Я советую использовать [Kitsune Magisk](https://github.com/HuskyDG/magisk-files/releases). Устанавливаем и запускаем на телефоне\
На главной в блоке **Magisk** тапаем **Установка**, затем выбираем пропатчить boot-образ и выьираем скопированный ранее boot.img\
После завершения, в загрузках должнен появиться пропатченный образ(подобное имя magisk_patched-27001_ez3YV.img), его коируем на ПК.

Далее скачиваем скрипт подписи и ключ
```PowerShell
New-Item -Type Directory "$([Environment]::GetFolderPath("Desktop"))\A7cc_root"
$rootDir = $([Environment]::GetFolderPath("Desktop"))\A7cc_root
$avb = curl https://android.googlesource.com/platform/external/avb/+/refs/heads/main/avbtool.py?format=text
[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($avb)) | Out-File "$rootDir\avbtool.py" -Encoding ascii
(curl https://raw.githubusercontent.com/unisoc-android/unisoc-android.github.io/refs/heads/master/subut/assets/rsa4096_vbmeta.pem).content | Out-File $rootDir\rsa4096_vbmeta.pem -Encoding ascii
``` 

В папку **A7cc_root** на рабочем столе нужно положить свой boot.img и пропатченный Magisk'ом образ, затем выполнить в том же окне PowerShell 
```PowerShell
cd $rootDir
python .\avbtool.py info_image --image .\boot.img
``` 

На выходе получим что-то вроде
```
Footer version:           1.0
Image size:               36700160 bytes
Original image size:      19232768 bytes
VBMeta offset:            19234816
VBMeta size:              2176 bytes
--
Minimum libavb version:   1.0
Header Block:             256 bytes
Authentication Block:     576 bytes
Auxiliary Block:          1344 bytes
Public key (sha1):        545f96af476584c35500f8b2651a052ee9431f1e
Algorithm:                SHA256_RSA4096
Rollback Index:           0
Flags:                    0
Rollback Index Location:  0
Release String:           'avbtool 1.1.0'
Descriptors:
    Hash descriptor:
      Image Size:            19232768 bytes
      Hash Algorithm:        sha256
      Partition Name:        boot
      Salt:                  44cde590d6afefb4e84281b8bdcaea9500ed30c17643a28174b5f29078907afa
      Digest:                7701ddd4725b1768d60299034b4fa0c0cd13470f9007076bb6eaf998149cfc7a
      Flags:                 0
    Prop: com.android.build.boot.os_version -> '10'
```
Для "красоты" можно скопировать свое значение поля Salt. У меня оно 44cde590d6afefb4e84281b8bdcaea9500ed30c17643a28174b5f29078907afa, у вас будет другое.\
Затем подставляем его в команду ниже + меняем имя своего пропатченного Magisk'ом образа
```PowerShell
cd $rootDir
python .\avbtool.py add_hash_footer --image .\magisk_patched-27001_ez3YV.img --partition_name boot --partition_size 36700160 --key rsa4096_vbmeta.pem --algorithm SHA256_RSA4096 --salt 44cde590d6afefb4e84281b8bdcaea9500ed30c17643a28174b5f29078907afa
``` 

Проверяем, что телефон виден
```PowerShell
abd devices
```

Перезагружем телефон в fastboot
```PowerShell
adb reboot fastboot
```

Дожидаемся черного экрана с меню fastboot и проверяем, что телефон виден
```PowerShell
fasboot devices
```

Прошиваем наш пропатченный boot(имя файла нужно заменить на свое)
```PowerShell
fastboot flash boot .\magisk_patched-27001_ez3YV.img
```

Дожилаемся окончания процесса и перезагружаемся в систему командой
```PowerShell
fastboot reboot
```

# Обновление Magisk из-под Windows
Все шаги полностью аналогичны разделу [Root из-под Windows](README.md#root-из-под-windows)\
Если используется тот же ПК и софт не удаляли, то можно начать с пункта [Устанавливаем Magisk](README.md#устанавливаем-magisk)
