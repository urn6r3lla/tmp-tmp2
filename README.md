# Лабораторная работа №2

**Дисциплина:** Современные технологии ИБ  
**Тема:** Развертывание и первичная настройка SIEM Wazuh. Анализ событий ИБ.  
**Вариант:** 4  
**Выполнили:** Фролкин Никита, Григорий Котенков, Трофимов Кирилл  
---

### Состав стенда

| Машина | ОС | IP-адрес (Host-Only) | Роль |
|--------|----|-----------------------|------|
| wazuh-server | Amazon Linux 2023 (OVA) | 192.168.56.31 | Сервер Wazuh (менеджер, индексатор, дашборд) |
| ubuntu-agent | Ubuntu Server 24.04 | 192.168.56.30 | Агент Wazuh, сбор логов auditd, syslog |
| windows-agent | Windows Server 2022 (Desktop Experience) | 192.168.56.32 | Агент Wazuh, сбор логов Security |

Все машины подключены к виртуальной сети **Internal network** (`intnet`) и имеют второй адаптер **Сетевой мост** для доступа в интернет.

![alt text](screenshots/0.png)

![alt text](screenshots/1.png)
---

## Этап 1: Развертывание Wazuh‑сервера

1. Импортировал OVA-образ Wazuh в VirtualBox. https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html

2. Настроил сеть: Адаптер 1 – Сетевой мост, Адаптер 2 – Internal network (intnet).  
3. Назначил статический IP `192.168.56.31/24` через утилиту `ip`.  
4. Открыл веб-интерфейс `https://10.132.58.63`, задал новый пароль администратора.

> **Скриншот 1** – окно VirtualBox с настройками сети сервера Wazuh.
![alt text](screenshots/4.png)  
> **Скриншот 2** – веб-интерфейс Wazuh со страницей входа.
![alt text](screenshots/2.png)

---

## Этап 2: Создание и настройка агентов

### 2.1 Ubuntu‑агент

1. Установил Wazuh‑агента согласно документации:
   ```bash
   curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
   echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
   sudo apt update
   sudo WAZUH_MANAGER="192.168.56.31" apt install wazuh-agent
   ```
![alt text](screenshots/3.png)
![alt text](screenshots/5.png)

2. Настроил мониторинг `audit.log` в `/var/ossec/etc/ossec.conf`:
   ```xml
   <localfile>
     <log_format>syslog</log_format>
     <location>/var/log/audit/audit.log</location>
   </localfile>
   ```
   ![alt text](screenshots/6.png)

3. Активировал правила auditd для записи команд, входа и изменений учётных записей
![alt text](screenshots/15.png)

4. Убедился, что агент подключён (статус `Active`).

![alt text](screenshots/7.png)
![alt text](screenshots/8.png)
![alt text](screenshots/23.png)
![alt text](screenshots/9.png)

### 2.2 Windows‑агент

1. Установил Wazuh‑агента согласно документации:
   ```powershell
   wazuh-agent-4.7.5-1.msi /q WAZUH_MANAGER="192.168.56.31"
   ```
2. Настроил сбор журнала `Security`.
3. Включил расширенный аудит через `auditpol` для записи входов, управления учётными записями и создания процессов.

> **Скриншот 6** – окно PowerShell с командами `auditpol` и результатами.  
![alt text](screenshots/10.png)
![alt text](screenshots/11.png)

---

## Этап 3: Генерация событий для Варианта 4

### Действия на Ubuntu‑агенте
- Успешные входы по SSH под пользователями `user`, `test8`.
- Неудачные входы для `test8` (попытоки с неверным паролем).
- Запуск команд: `whoami`, `ls`, `grep`, `find`, `echo`.
- Создание учётных записей `testuserN`.

![alt text](screenshots/12.png)
![alt text](screenshots/13.png)
![alt text](screenshots/14.png)

### Действия на Windows‑агенте
- Успешные входы (`test8`).
- Неудачные входы для `test8` (попытоки с неверным паролем).
- Запуск процессов: `whoami.exe`, `ipconfig.exe`, `dir`, `echo`.
- Создание пользователя `labuser`.

---

## Этап 4: Анализ событий (Вариант 4)

Все фильтры вводились в **Wazuh Discover** (индекс `wazuh-alerts-*`).  
Ниже приведены формулировки заданий, использованные KQL-фильтры, визуализация и скриншоты.

### 4.1 Успешные аутентификации под пользовательскими УЗ, сгруппированные по активной УЗ

> **Скриншот 8** – Discover с фильтром.
![alt text](screenshots/16.png)
![alt text](screenshots/17.png)
![alt text](screenshots/18.png)
![alt text](screenshots/24.png)

### 4.2 Количество неуспешных аутентификаций для УЗ test8 за последние сутки

> **Скриншот 9** – Discover, фильтр, счётчик хитов.
![alt text](screenshots/19.png)
![alt text](screenshots/25.png)

### 4.3 Процессы командной строки, сгруппированные по командной строке

> **Скриншот 10** – Discover с фильтром и таблицей (команды `whoami`, `ls` и др.).
![alt text](screenshots/20.png)
![alt text](screenshots/26.png)

### 4.4 События создания новых УЗ

> **Скриншот 11** – Discover, событие `ADD_USER`.

![alt text](screenshots/21.png)
> **Скриншот 12** – Discover, событие `4720`.
![alt text](screenshots/27.png)

### 4.5 Все события журнала Auditd (и эквивалент Windows)

> **Скриншот 13** – общий список событий auditd
![alt text](screenshots/22.png)
> **Скриншот 14** – общий список событий Security Windows.
![alt text](screenshots/28.png)

---
