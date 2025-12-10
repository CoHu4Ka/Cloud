# Лабораторная работа: Создание виртуальной сети (VPC) в AWS

## Описание
Эта лабораторная работа посвящена настройке виртуальной частной облачной сети (VPC) в Amazon Web Services (AWS). Цель — создать изолированную сеть с публичной и приватной подсетями, интернет-шлюзом (IGW), NAT Gateway, таблицами маршрутов и EC2-инстансами для веб-сервера, сервера базы данных и bastion host. Работа демонстрирует принципы сетевой изоляции, маршрутизации и безопасности в облаке AWS.

## Цели
- Научиться создавать и настраивать VPC, подсети, таблицы маршрутов, IGW и NAT Gateway.
- Понять принципы изоляции сетей в AWS.
- Настроить взаимодействие между EC2-инстансами в разных подсетях.
- Изучить различия между публичными и приватными маршрутами.

## Требования
- Аккаунт AWS с доступом к консоли управления.
- Установленный SSH-клиент (OpenSSH в Windows или PuTTY).
- Браузер для проверки веб-сервера.

## Основные этапы
1. Подготовка среды и создание VPC.
2. Создание Internet Gateway (IGW).
3. Создание публичной и приватной подсетей.
4. Настройка таблиц маршрутов.
5. Создание NAT Gateway и Elastic IP.
6. Настройка Security Groups.
7. Запуск EC2-инстансов (web-server, db-server, bastion-host).
8. Проверка конфигурации.
9. Дополнительные задания: доступ через bastion host.
10. Завершение работы и удаление ресурсов.

## Параметры
- **VPC CIDR**: `10.12.0.0/16` (k=12).
- **Публичная подсеть**: `10.12.1.0/24` (eu-central-1a).
- **Приватная подсеть**: `10.12.2.0/24` (eu-central-1b).
- **Регион**: Frankfurt (eu-central-1).
- **EC2-инстансы**: Amazon Linux 2, t3.micro, ключ `student-key-k12`.

## Инструкция по выполнению

### Шаг 1. Подготовка среды
1. Войдите в AWS Management Console.
2. Установите регион: Frankfurt (eu-central-1).
3. Откройте консоль VPC (поиск "VPC").

### Шаг 2. Создание VPC
1. В разделе "Your VPCs" нажмите "Create VPC".
2. Параметры:
   - Name tag: `student-vpc-k12`.
   - IPv4 CIDR block: `10.12.0.0/16`.
   - Tenancy: Default.
3. Нажмите "Create VPC".

### Шаг 3. Создание Internet Gateway (IGW)
1. В разделе "Internet Gateways" нажмите "Create internet gateway".
2. Name: `student-igw-k12`.
3. Привяжите к VPC `student-vpc-k12` (Attach to VPC).

### Шаг 4. Создание подсетей
1. **Публичная подсеть**:
   - В разделе "Subnets" нажмите "Create subnet".
   - VPC: `student-vpc-k12`.
   - Subnet name: `public-subnet-k12`.
   - Availability Zone: `eu-central-1a`.
   - IPv4 CIDR block: `10.12.1.0/24`.
2. **Приватная подсеть**:
   - Аналогично, создайте подсеть:
   - Subnet name: `private-subnet-k12`.
   - Availability Zone: `eu-central-1b`.
   - IPv4 CIDR block: `10.12.2.0/24`.

### Шаг 5. Настройка таблиц маршрутов
1. **Публичная таблица маршрутов**:
   - В "Route Tables" создайте таблицу: `public-rt-k12`.
   - VPC: `student-vpc-k12`.
   - Добавьте маршрут: `0.0.0.0/0 → student-igw-k12`.
   - Привяжите к `public-subnet-k12`.
2. **Приватная таблица маршрутов**:
   - Создайте таблицу: `private-rt-k12`.
   - VPC: `student-vpc-k12`.
   - Привяжите к `private-subnet-k12`.
   - (Маршрут для NAT Gateway добавим позже.)

### Шаг 6. Создание NAT Gateway
1. **Elastic IP**:
   - В "Elastic IPs" нажмите "Allocate Elastic IP address".
2. **NAT Gateway**:
   - В "NAT Gateways" нажмите "Create NAT gateway".
   - Name: `nat-gateway-k12`.
   - Subnet: `public-subnet-k12`.
   - Connectivity: Public.
   - Выберите созданный Elastic IP.
3. **Обновите приватную таблицу маршрутов**:
   - В `private-rt-k12` добавьте: `0.0.0.0/0 → nat-gateway-k12`.

### Шаг 7. Настройка Security Groups
1. Создайте группы безопасности:
   - `web-sg-k12`: Inbound HTTP (80) и HTTPS (443) от `0.0.0.0/0`.
   - `bastion-sg-k12`: Inbound SSH (22) от вашего IP.
   - `db-sg-k12`: Inbound MySQL (3306) от `web-sg-k12`; SSH (22) от `bastion-sg-k12`.

### Шаг 8. Создание EC2-инстансов
1. **Общие параметры**:
   - AMI: Amazon Linux 2.
   - Тип: t3.micro.
   - Ключ: `student-key-k12` (скачайте .pem файл).
2. **web-server**:
   - VPC: `student-vpc-k12`, Subnet: `public-subnet-k12`.
   - Auto-assign Public IP: Enable.
   - Security Group: `web-sg-k12`.
   - User Data:
     ```bash
     #!/bin/bash
     dnf install -y httpd php
     echo "<?php phpinfo(); ?>" > /var/www/html/index.php
     systemctl enable httpd
     systemctl start httpd
     ```
   - Тег: `Name: web-server`.
3. **db-server**:
   - VPC: `student-vpc-k12`, Subnet: `private-subnet-k12`.
   - Auto-assign Public IP: Disable.
   - Security Group: `db-sg-k12`.
   - User Data:
     ```bash
     #!/bin/bash
     dnf install -y mariadb105-server
     systemctl enable mariadb
     systemctl start mariadb
     mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongPassword123!'; FLUSH PRIVILEGES;"
     ```
   - Тег: `Name: db-server`.
4. **bastion-host**:
   - VPC: `student-vpc-k12`, Subnet: `public-subnet-k12`.
   - Auto-assign Public IP: Enable.
   - Security Group: `bastion-sg-k12`.
   - User Data:
     ```bash
     #!/bin/bash
     dnf install -y mariadb105
     ```
   - Тег: `Name: bastion-host`.

### Шаг 9. Проверка работы
1. **Веб-сервер**:
   - Откройте публичный IP `web-server` в браузере (например, `http://<public-ip>`).
   - Ожидаемый результат: страница `phpinfo()`.
2. **Bastion-host**:
   - Подключитесь по SSH:
     ```powershell
     ssh -i <путь_к_ключу> ec2-user@<bastion-host-public-ip>
     ```
   - Выполните: `ping -c 4 google.com` (должен быть успех).
3. **MySQL на db-server**:
   - С `bastion-host` выполните:
     ```bash
     mysql -h <db-server-private-ip> -u root -p
     ```
   - Пароль: `StrongPassword123!`.
   - Если возникает ошибка `ERROR 1130`, настройте права:
     ```sql
     GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.12.1.0/24' IDENTIFIED BY 'StrongPassword123!';
     FLUSH PRIVILEGES;
     ```

### Шаг 10. Дополнительные задания
1. **На локальной машине (Windows)**:
   - Запустите SSH Agent:
     ```powershell
     Start-Service ssh-agent
     ssh-add C:\путь\к\student-key-k12.pem
     ```
   - Подключитесь к `db-server` через `bastion-host`:
     ```powershell
     ssh -A -J ec2-user@<bastion-host-public-ip> ec2-user@<db-server-private-ip>
     ```
2. **На db-server**:
   - Обновите систему:
     ```bash
     sudo dnf update -y
     ```
   - Установите htop:
     ```bash
     sudo dnf install -y htop
     ```
   - Подключитесь к MySQL:
     ```bash
     mysql -u root -p
     ```
     Пароль: `StrongPassword123!`.
3. **Завершение**:
   - Остановите SSH Agent:
     ```powershell
     Stop-Service ssh-agent
     ```

### Шаг 11. Завершение работы
1. Удалите ресурсы для экономии:
   - EC2-инстансы: `web-server`, `db-server`, `bastion-host`.
   - NAT Gateway и Elastic IP.
   - (Опционально) Security Groups, IGW (detach затем delete), VPC.

## Ответы на контрольные вопросы
1. **Почему нельзя использовать CIDR /8?**
   - Маска /16 предоставляет 65,536 адресов, что достаточно для VPC. AWS ограничивает CIDR от /16 до /28, чтобы избежать конфликтов с глобальными сетями. CIDR /8 слишком велик и может пересекаться с другими сетями.
2. **Когда подсеть становится публичной/приватной?**
   - Публичная: когда связана с таблицей маршрутов, содержащей маршрут `0.0.0.0/0 → IGW`.
   - Приватная: когда использует таблицу маршрутов без прямого доступа к IGW (обычно с NAT Gateway).
3. **Зачем нужен NAT Gateway?**
   - Позволяет приватным инстансам (например, `db-server`) выходить в интернет для обновлений через публичный IP, блокируя входящий трафик.
4. **Зачем нужен bastion-host?**
   - Это защищенный сервер в публичной подсети, используемый для SSH-доступа к приватным ресурсам. Минимизирует поверхность атаки, так как прямой доступ к приватной подсети невозможен.
5. **Что делают опции -A и -J в SSH?**
   - `-A`: Включает SSH Agent Forwarding для передачи ключа через `bastion-host`.
   - `-J`: Указывает промежуточный сервер (`bastion-host`) для подключения к конечному серверу.

## Устранение ошибок
- **Ошибка `ERROR 1130 (HY000): Host '...' is not allowed`**:
  - На `db-server` выполните:
    ```sql
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.12.1.0/24' IDENTIFIED BY 'StrongPassword123!';
    FLUSH PRIVILEGES;
    ```
- **Ошибка SSH `Connection timed out`**:
  - Проверьте Security Groups (`bastion-sg-k12`, `db-sg-k12`).
  - Убедитесь, что инстансы в статусе `Running`.
  - Проверьте таблицы маршрутов и NAT Gateway.
- **Ошибка `Start-SshAgent` в Windows**:
  - Используйте:
    ```powershell
    Start-Service ssh-agent
    ```

## Источники
- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ug.pdf)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
- [SSH Documentation](https://man.openbsd.org/ssh)