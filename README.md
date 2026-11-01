# Сессионный проект МИФИ — 2026

**Тема:** установка и настройка защищённого Linux-сервера для публикации результатов ГИС  
**Номер зачётной книжки:** `375425`  
**Среда:** Fedora Server без графического интерфейса в виртуальной машине UTM (Mac, Apple Silicon)

> В учебном проекте использована **Fedora Server** вместо РЕД ОС 8 — в соответствии с разрешённой преподавателем заменой дистрибутива. Данная конфигурация демонстрирует выполнение учебных требований и не является подтверждением сертификации Fedora для эксплуатации в государственной информационной системе.

## Цель проекта

Развернуть Linux-сервер, настроить сеть, разделы дисков, системные службы, дискреционное и мандатное управление доступом, ограничения аутентификации и политику паролей. Опубликовать тестовую страницу через Nginx и предоставить артефакты выполненных работ.

## Конфигурация стенда

| Параметр | Значение |
|---|---|
| Виртуализация | UTM |
| Операционная система | Fedora Server, без GUI |
| Имя хоста | `mephi-2026.domain.local` |
| Системный диск | `/dev/vda`, 15 ГБ; корневая ФС — XFS на LVM |
| Дополнительный диск | `/dev/vdb`, 10 ГБ |
| Раздел данных | `/dev/vdb1`, ext4, метка `MEPHI_WEB` |
| Точка монтирования | `/mephi-web` |
| Веб-сервер | Nginx |
| Управление доступом | POSIX ACL, Linux capabilities, SELinux, PAM |
| SELinux | `Enforcing` |

## Выполнение задания

| Раздел | Реализация | Подтверждение |
|---|---|---|
| **1. Установка и сеть** | Установлена Fedora Server без GUI; задан hostname; настроена сетевая связность; выполнен `ping` до `8.8.8.8` | [`ping.out`](ping.out), [`history.out`](history.out) |
| **2. Программное обеспечение** | Выполнено обновление через `dnf upgrade`; установлены Nginx и `libcap-ng-utils`; RPM пакета `tcpdump` скачан командой `dnf download` и установлен через `rpm` | [`dnf.out`](dnf.out), [`history.out`](history.out) |
| **3. Файловые системы** | На `/dev/vdb` создан раздел `/dev/vdb1`, отформатирован в ext4 с меткой `MEPHI_WEB`, смонтирован в `/mephi-web`; добавлена запись для автоматического монтирования | [`fstab`](fstab), [`stat.out`](stat.out), [`history.out`](history.out) |
| **4. Системные службы** | Nginx запущен и добавлен в автозагрузку (`systemctl enable --now nginx`); сохранён журнал текущей загрузки | [`journalctl.out`](journalctl.out), [`history.out`](history.out) |
| **5.1. Дискреционный доступ (DAC)** | Созданы разработчики `user1–user3` с UID 5501–5503 и группа `developers`; кураторы `curator1–curator2` входят в группу `curators` с GID 4444. На `/data/mephi-2026` настроены SGID и обычные/default ACL: разработчики — чтение/запись, кураторы — чтение, остальные — без доступа | [`passwd`](passwd), [`group`](group), [`stat.out`](stat.out), [`acl.out`](acl.out) |
| **5.2. Привилегии** | Для `/usr/sbin/tcpdump` назначены `cap_net_raw` и `cap_net_admin`; доступ к исполняемому файлу ограничен группой `pcap`. Захват ICMP от `user1` проверен без запуска от root | [`getcap.out`](getcap.out), [`history.out`](history.out) |
| **5.3. SELinux (MAC)** | Сохранён режим `Enforcing`; для каталога `/mephi-web` настроен постоянный контекст `httpd_sys_content_t` через `semanage fcontext` и `restorecon` | [`getenforce.out`](getenforce.out), [`stat.out`](stat.out), [`history.out`](history.out) |
| **6. Аутентификация** | Для службы локального входа `login` подключён `pam_access.so`, запрещающий вход членам группы `curators`; ограничение проверено через `pamtester`. Для пользователей установлен максимальный срок пароля 90 дней; в `pwquality.conf` задано `minlen = 12` | [`pam-login.conf`](pam-login.conf), [`console-access.out`](console-access.out), [`pam-check.out`](pam-check.out), [`password-aging.out`](password-aging.out), [`pwquality.conf`](pwquality.conf) |
| **7. Тестирование веб-сервера** | В `/mephi-web/index.html` размещена страница с номером зачётной книжки; Nginx настроен на нестандартный корневой каталог; успешный ответ проверен через `curl` | [`curl.out`](curl.out), [`journalctl.out`](journalctl.out) |
| **8. Публикация** | В репозитории размещаются результаты команд, системные конфигурации и подтверждения выполненных настроек | Файлы проекта и данный README |

## Ключевые настройки

### Автомонтирование второго диска

Запись в `/etc/fstab`:

```fstab
LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 2
```

Использование метки тома позволяет монтировать файловую систему без привязки к изменяемому имени устройства.

### Права на каталог проекта

Каталог: `/data/mephi-2026`.

```text
Владелец: root
Группа: developers
Режим: 2770 (SGID)

ACL каталога:
  developers (группа каталога): rwx
  curators:                      r-x
  other:                         ---

Default ACL:
  developers (группа каталога): rwx
  curators:                      r-x
  other:                         ---
```

Default ACL наследуются вновь созданными объектами с учётом типа объекта и маски прав. Проверено создание файла пользователем `user1`, запись пользователем `user2`, чтение пользователем `user3` и чтение без возможности изменения пользователем `curator1`.

### Запуск tcpdump без root

Назначенные привилегии:

```text
/usr/sbin/tcpdump cap_net_admin,cap_net_raw=eip
```

Проверка выполнялась от имени `user1` с захватом ICMP на интерфейсе `lo`.

### SELinux и Nginx

Для публикации страницы используется каталог `/mephi-web`. Для него и его содержимого задан тип SELinux `httpd_sys_content_t`; режим SELinux — `Enforcing`.

Результат проверки:

```console
$ curl http://localhost/
Hello from Student: 375425
```

### Ограничение локального входа

В PAM-конфигурацию `/etc/pam.d/login` добавлено подключение модуля `pam_access.so` к файлу правил `/etc/security/mephi-console-access.conf`.

```text
-:(curators):ALL
+:ALL:ALL
```

Проверка `pamtester` для операции `acct_mgmt`: разрешение для `user1`, отказ для `curator1`. Данный тест проверяет PAM-правила для службы `login`, но не является записью отдельного интерактивного входа в консоль.

## Проверка после перезагрузки

После `sudo reboot` дополнительно выполнялись проверки:

```bash
hostnamectl
findmnt /mephi-web
getenforce
systemctl status nginx --no-pager
curl http://localhost/
getcap /usr/sbin/tcpdump
```

Имеющиеся артефакты подтверждают успешную работу Nginx после загрузки, наличие записи автомонтирования, режим SELinux и назначенные capabilities. Последовательность команд представлена в `history.out`.

## Артефакты

**Обязательные файлы:**

| Файл | Содержание |
|---|---|
| `mephi-screenshot.png` | Визуальное подтверждение ответа веб-сервера |
| `history.out` | История команд |
| `ping.out` | Проверка сетевой связности |
| `dnf.out` | История транзакций DNF5 |
| `stat.out` | Права и контексты каталогов |
| `journalctl.out` | Журнал сервиса Nginx |
| `getcap.out` | Capabilities исполняемого файла tcpdump |
| `getenforce.out` | Режим SELinux |
| `curl.out` | HTML-ответ веб-сервера |
| `fstab` | Копия `/etc/fstab` |
| `passwd` | Копия `/etc/passwd` |
| `shadow` | Артефакт учётных записей `/etc/shadow` |
| `group` | Копия `/etc/group` |
| `pwquality.conf` | Копия `/etc/security/pwquality.conf` |

**Дополнительные файлы:**

| Файл | Содержание |
|---|---|
| `acl.out` | ACL каталога проекта и тестового файла |
| `console-access.out` | Правила доступа кураторов |
| `pam-login.conf` | Конфигурация PAM-службы локального входа |
| `pam-check.out` | Результат проверки правил через pamtester |
| `password-aging.out` | Параметры срока действия пароля пользователя |

> **Замечание по безопасности:** `/etc/shadow` содержит хеши паролей. При публикации его оригинальной копии в общедоступном репозитории хеши становятся публичными; для реального проекта необходима обезличенная версия.

## Результат

На тестовой Fedora Server выполнены настройки файловой системы, Nginx, Linux capabilities, ACL, SELinux и PAM. Веб-сервер успешно отдаёт страницу из дополнительного раздела по адресу `http://localhost/`. Конфигурация и результаты проверок представлены в приложенных артефактах.
