# Лабораторная работа: Введение в AWS и развёртывание статического сайта на EC2

## Постановка задачи

Лабораторная работа посвящена освоению базовых сервисов Amazon Web Services (AWS): создание аккаунта, настройка IAM для управления доступом, контроль расходов через Budgets, запуск виртуальной машины EC2 с Nginx, мониторинг, подключение по SSH и развёртывание статического веб-сайта (задание 6a). В конце инстанс останавливается, чтобы избежать расходов. Все действия выполнялись в Free Tier для минимизации затрат.

## Цель и основные этапы работы

**Цель:** Освоить ключевые инструменты AWS для управления инфраструктурой, обеспечения безопасности и развёртывания приложений. Развёрнут статический сайт на EC2 с Nginx, включая HTML, CSS, JS и изображения.

**Основные этапы:**
1. Подготовка среды (регистрация в AWS, выбор региона).
2. Создание IAM группы и пользователя.
3. Настройка Zero-Spend Budget.
4. Создание и запуск EC2-инстанса.
5. Логирование и мониторинг.
6. Подключение по SSH.
7. Развёртывание статического сайта (задание 6a).
8. Завершение работы (остановка инстанса).

## Практическая часть

### Задание 0: Подготовка среды
Зарегистрировался в AWS Free Tier по ссылке https://aws.amazon.com/. Создал аккаунт под root-пользователем. В консоли выбрал регион EU (Frankfurt) eu-central-1.


### Задание 1: Создание IAM группы и пользователя
В сервисе IAM создал группу "Admins" с политикой AdministratorAccess. Создал пользователя "cloudstudent", добавил в группу Admins, разрешил доступ к консоли. Вошёл под новым пользователем.

**Ответ на вопрос: Что делает политика AdministratorAccess?**  
Политика AdministratorAccess предоставляет полный доступ ко всем сервисам и ресурсам AWS (аналог root), позволяя выполнять любые действия, включая создание/удаление инстансов. Рекомендуется только для админов.


### Задание 2: Настройка Zero-Spend Budget
В Billing and Cost Management создал бюджет "ZeroSpend" с шаблоном Zero spend budget, указав email для уведомлений о расходах > $0.


### Задание 3: Создание и запуск EC2 экземпляра
В EC2 запустил инстанс:
- Name: webserver.
- AMI: Amazon Linux 2023.
- Type: t3.micro.
- Key pair: Создал "yournickname-keypair" (.pem файл скачан, права ограничены `icacls /grant:r`).
- Security group: "webserver-sg" с правилами HTTP (80, anywhere) и SSH (22, my IP).
- User Data:
  ```bash
  #!/bin/bash
  dnf -y update
  dnf -y install htop
  dnf -y install nginx
  systemctl enable nginx
  systemctl start nginx
  ```
- Запустил, дождался Running и 2/2 checks. Проверил Nginx по `http://<Public-IP>` (стандартная страница).

**Ответ на вопрос: Что такое User Data и роль скрипта? Nginx?**  
User Data — скрипт, выполняемый при запуске инстанса для автоматизации настройки (обновление системы, установка htop для мониторинга, nginx как веб-сервера). Nginx — высокопроизводительный веб-сервер для обслуживания статического контента, reverse proxy.


### Задание 4: Логирование и мониторинг
- Status checks: 2/2 passed (system и instance reachability).
- Monitoring: Базовые метрики CloudWatch (CPU, Network); enlarge для детального просмотра.
- System Log: Нашёл записи об установке nginx из User Data.
- Instance Screenshot: Консольный вывод без ошибок.

**Ответ на вопрос: Когда включать детализированный мониторинг?**  
Для высоконагруженных приложений, Auto Scaling или отладки (метрики каждую минуту вместо 5 мин), чтобы быстро выявлять проблемы и минимизировать downtime.


### Задание 5: Подключение по SSH
В CMD: `cd` к папке ключа, `icacls` для прав `R` только для пользователя. Подключился:
```cmd
cd C:\Users\CoHu4Ka\OneDrive\Desktop\Folders\University\HTML
icacls yournickname-keypair.pem /inheritance:d
icacls yournickname-keypair.pem /grant:r "DESKTOP-B7P35ED\CoHu4Ka:R"
ssh -i yournickname-keypair.pem ec2-user@<Public-IP>
```
Проверил: `systemctl status nginx` (active).

**Ответ на вопрос: Почему SSH без пароля в AWS?**  
Ключи безопаснее паролей (нет brute-force атак, приватный ключ локально), обеспечивают шифрование и аутентификацию без передачи по сети.


### Задание 6a: Развёртывание статического веб-сайта
Локальная структура: `Lab6` (index.html, css/style.css, script/script.js, img/*.png/jpg).
- По SSH:
  ```bash
  sudo mkdir -p /usr/share/nginx/html/Lab6/{css,script,img}
  sudo chown -R ec2-user:ec2-user /usr/share/nginx/html/Lab6
  ```
- В CMD копировал файлы:
  ```cmd
  cd C:\Users\CoHu4Ka\OneDrive\Desktop\Folders\University\HTML
  scp -i yournickname-keypair.pem Lab6\index.html ec2-user@<Public-IP>:/usr/share/nginx/html/Lab6/
  scp -i yournickname-keypair.pem Lab6\css\style.css ec2-user@<Public-IP>:/usr/share/nginx/html/Lab6/css/
  scp -i yournickname-keypair.pem Lab6\script\script.js ec2-user@<Public-IP>:/usr/share/nginx/html/Lab6/script/
  scp -i yournickname-keypair.pem -r Lab6\img\* ec2-user@<Public-IP>:/usr/share/nginx/html/Lab6/img/
  ```
- На сервере:
  ```bash
  sudo chmod -R 644 /usr/share/nginx/html/Lab6/*
  sudo chmod 755 /usr/share/nginx/html/Lab6 /usr/share/nginx/html/Lab6/css /usr/share/nginx/html/Lab6/script /usr/share/nginx/html/Lab6/img
  sudo systemctl restart nginx
  ```
- Проверил: `ls -l /usr/share/nginx/html/Lab6`, сайт по `http://<Public-IP>/Lab6/index.html`.

**Ответ на вопрос: Что делает scp?**  
`scp` (secure copy) копирует файлы между хостами по SSH с шифрованием, аналог `cp`, но для удалённых серверов (например, загрузка сайта на EC2).


### Задание 7: Завершение работы
Остановил инстанс:
```cmd
aws ec2 stop-instances --instance-ids <instance-id>
```
**Ответ на вопрос: Stop vs Terminate?**  
Stop выключает инстанс (сохраняет EBS-данные, billing только за storage). Terminate удаляет инстанс навсегда (освобождает ресурсы, данные потеряны, если не snapshot).


### Проблемы и решения
- Права на .pem: Исправлены `icacls /grant:r "DESKTOP-B7P35ED\CoHu4Ka:R"`.
- `mkdir: Permission denied`: Решено с `sudo mkdir` и `sudo chown`.
- `No such file or directory`: Переименовал папку на `Lab6` (без пробела), создал структуру заново.
- Сайт: Проверены пути в HTML, порт 80 в Security Group, логи Nginx (`/var/log/nginx/error.log`).

## Ответы на контрольные вопросы
(Интегрированы в шаги выше для удобства.)

## Список использованных источников
- AWS Documentation: https://docs.aws.amazon.com/
- AWS Free Tier: https://aws.amazon.com/free/
- Теоретический материал курса: 02_AWS_Introduction.
- Nginx: https://nginx.org/en/docs/
- SSH/SCP: https://man.openbsd.org/scp

## Дополнительные важные аспекты
- Все ресурсы в Free Tier (t3.micro, S3 как альтернатива).
- Безопасность: Ограничены права на ключ, SSH только с IP, Zero-Spend для уведомлений.
- Репозиторий Git: [Ссылка на репозиторий с HTML/CSS/JS файлами и отчётом, например, https://github.com/CoHu4Ka/aws-lab].
- Рекомендация: Для продакшена S3 лучше для статических сайтов (экономичнее EC2).
- Представление: Готов к защите; вопросы по шагам/ошибкам решены.