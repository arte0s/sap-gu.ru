# Установка SAP

1. Скачать OpenSUSE версии Leap 15.3 (**Важно:** 15.4 пока не подходит)
1. Отключить виртуализацию в VirtualBox (иначе может отваливаться мышь)
1. При установке
   * Настроить параметры диска
      * Выбрать `EXT4`
      * Установить галочку `Propose Separate Swap Partition`
   * Выполнить настройки безопасности
      * Отключение файрвола активировать (`enable`)
      * Отключение SSH деактивировать (`disable`)

1. Проверить свободное место на диске (должно быть в разделе `/dev/sda2` минимум 33 Gb свободного места)

   ```bash
   df -h
   ```

1. Настроить `hostname`

   ```bash
   vi /etc/hostname
   ```

   Для входа в режим изменения нажать `i` и добавить следующий текст (изначально файл должен быть пустой)

   ```bash
    vhcalnplci
   ```

   Для сохранения изменений нажать `ESC`, набрать `:wq` и нажать `Enter`

   ```bash
    cat /etc/hostname
    rcnetwork restart
    hostname
   ```

1. Настроить `hosts`

   ```bash
   ip a #Найти IP-адрес интерфейса eth0 (в примере 10.0.2.15)
   vi /etc/hosts 
   ```

   Для входа в режим изменения нажать `i` и после строки `127.0.0.1` добавить

   ```bash
   10.0.2.15 vhcalnplci vhcalnplci.dummy.nodomain
   ```

   Для сохарнения измененийи нажать `ESC`, набрать `:wq` и нажать `Enter`

1. Настроить виртуальную машину Virtual Box

   * В разеделе `Shared folders` настроить общую папку для доступа к дистрибутву

       * Выбрать Путь к дистрибутиву
       * Выбрать имя папки (если, например, выбрать имя `sap_distr`, то в системе путь будет `/media/sf_sap_distr/`)
       * Установить флаг и `Auto-Mount` и `Make permanent`

   * В разеделе `Network` > `Advanced` > `Port forwarding` настроить проброс портов

   (вместо `10.0.2.15` указать адрес из п.8)

   |Имя|Протокол|Адрес|Порт|Адрес|Порт|
   |---|--------|-----|----|-----|----|
   | AiE | TCP | 127.0.0.1 | 3300 | 10.0.2.15 | 3300 |
   | HTTP | TCP | 127.0.0.1 | 8000 | 10.0.2.15 | 8000 |
   | HTTPS | TCP | 127.0.0.1 | 44300 | 10.0.2.15 | 44300 |
   | SAPGUI | TCP | 127.0.0.1 | 3200 | 10.0.2.15 | 3200 |
   | SCC | TCP | 127.0.0.1 | 8443 | 10.0.2.15 | 8443 |
   | SSH | TCP | 127.0.0.1 | 22 | 10.0.2.15 | 22 |

1. Установить пакет `uuidd`

   ```bash
   sudo zypper install uuidd
   ```

   Запустить сервис UUIDD (**Важно:** В случае перезагрузки до завешения установки, повторить запуск сервиса)

   ```bash
   sudo service uuidd start
   sudo service --status-all | grep uuidd
   ```

1. Сделать файл скрипта запуска установки исполняемым

   ```bash
   sudo -i
   cd /media/sf_SAPD
   chmod +x install.sh
   ls -l #Проверить, что изменение полномочий применилось
   ```

1. Запустить скрипт установки и задать сложный пароль (буквы, символы, цифры и не менее 9? символов)

   ```bash
   ./install.sh
   :q
   yes
   ...password...
   ...password...
   ```

1. Первый запуск установка прервётся с ошибкой необходимо обновить файл лицензии (который идёт в отдельном архиве) и заново запуститить скрипт установки

   ```bash
   cat /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
   cd /media/sf_sap_distr/License/SYBASE_ASE_TestDrive/ #Путь куда был распакован архив с новой лицензией
   cat SYBASE_ASE_TestDrive.lic
   cp SYBASE_ASE_TestDrive.lic /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
   cat /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
   ```

1. В случае успеха скрипт завершится с сообщениями

   ```bash
   Instance on host vhcalnplci started
   Installation of NPL successful
   ```

   Теперь можно сразу подключиться через SAPGUI, после установки сервер уже запущен

## Работа с сервером

1. Настройка подключения в SAPGUI (подключение из хост системы, не виртуальной)

   |Параметр|Значение|
   |-|-|
   | Сервер приложений | 127.0.0.1 |
   | Номер инстанции | 00 |
   | Ид. Системы | NPL |

1. Параметры входа

   (см. readme.html в корне каталога дистрибутива)

   |Параметр|Значение|
   |-|-|
   | Логин | DEVELOPER |
   | Мандант | 001 |
   | Пароль | Down1oad |

1. Запуск и остановка сервиса SAP

   ```bash
   su npladm
   startsap all
   ps -ef #Проверить, что процессы запущены
   ss -ltp #Проверить, что порты прослушиваются
   stopsap all
   ```

1. Получение лицензии (необходимо для начала разработки)

   Изначально номер инсталляциивас будет `INTERN`, с ним пользователь `DEVELOPER` не может писать код

   Необходимо [отсюда](https://go.support.sap.com/minisap/#minisap) скачать новую демо-лицензию

   Установить лицензию через тр. `SLICENSE`, кнопка Import

   После этого ключ разработчика не запрашивается

## Установка и обновление пакетов

   ```bash
   sudo zypper ref
   zypper list-updates --all
   sudo zypper update
   ```

*Источник: [linuxhint.com](https://linuxhint.com/update_all_packages_opensuse/)*

## Задать комбинацию горячих клавиш для открытия терминала

* Перейти `Settings` > `Keyboard`, затем вкладка `Shorcuts`
* Hit the `+` to add a new custom shortcut
* Under Name enter `Terminal` in `Command` enter 'gnome-terminal' and then press `Add`
* Click on `Disabled` to add a new accelerator, then press `CTRL+ALT+t` (or your preferred combination)

*Источник: [forums.opensuse.org](https://forums.opensuse.org/showthread.php/514520-Launch-Gnome-Terminal-With-A-Hot-Key)*

## Для OpenSUSE версии Leap 15.4

(но этого всего для успешной установки, похоже, недосточно; используейте версию Leap 15.3)

Установить пакет `tcsh`

   ```bash
   sudo zypper install tcsh
   ```

Если установка зависает на строке

   ```text
   INFO 2021-06-03 16:29:44.619 (root/sapinst) (startInstallation) [iaxxbfile.cpp:594] id=syslib.filesystem.aclSetSucceeded CIaOsFile::chmod_impl(3,0755)
   Authorizations set for /sybase/NPL/startase_reset_sa.
   ```

   то добавить в файл `/sybase/NPL/ASE-16_0/install/RUN_NPL` в строку `/NPL.cfg \` параметр `-T11889`

   ```text
   sed -i 's/NPL.cfg \\/NPL.cfg -T11889 \\/g' /sybase/NPL/ASE-16_0/install/RUN_NPL
   cat /sybase/NPL/ASE-16_0/install/RUN_NPL
   ```

PS: В видео формате хорошая инструкция есть [тут](https://youtu.be/zAbgkt3ibYc)