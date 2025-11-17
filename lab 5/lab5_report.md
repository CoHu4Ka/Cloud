# Лабораторная работа №5 — Облачные базы данных: Amazon RDS и DynamoDB

## Описание лабораторной работы
Данная лабораторная работа посвящена работе с облачными реляционными и нереляционными базами данных AWS: Amazon RDS (MySQL) и Amazon DynamoDB.

## Цель
- Освоить создание и настройку RDS.
- Научиться использовать Read Replica.
- Подключаться к базе данных RDS с EC2.
- Выполнять CRUD‑операции.
- Освоить основы DynamoDB.

---

## Шаг 1. Подготовка среды (VPC/подсети/SG)
### Действия:
1. Создана VPC `project-vpc` с 2 публичными и 2 приватными подсетями в разных AZ.
2. Созданы Security Groups:
   - **web-security-group** — входящий HTTP/80, SSH/22.
   - **db-mysql-security-group** — входящий MySQL/3306 только от web‑security‑group.
3. Добавлено исходящее правило MySQL/3306 из web-security-group в db‑mysql‑security‑group.

---

## Шаг 2. Развертывание Amazon RDS

### Subnet Group
**Что такое Subnet Group?**
Группа подсетей, которые RDS использует для размещения инстанса.  
Нужно, чтобы база могла работать в приватных подсетях и обеспечивать отказоустойчивость.

Создана:
- **project-rds-subnet-group**
- 2 приватные подсети в разных AZ.

### Создание RDS MySQL
Использованы параметры:
- Engine: **MySQL 8.0**
- Free Tier
- Single-AZ
- Instance: **db.t3.micro**
- Storage: GP3, 20GB + autoscaling до 100GB
- Public Access: **No**
- Security Group: `db-mysql-security-group`
- Initial DB: `project_db`

Получен endpoint для подключения.

---

## Шаг 3. Создание EC2
- Развёрнута EC2 в публичной подсети.
- SG: `web-security-group`
- Установлен MySQL клиент:

```bash
dnf update -y
dnf install -y mariadb105
```

---

## Шаг 4. Подключение к RDS и CRUD‑операции

### Подключение:

```bash
mysql -h <RDS_ENDPOINT> -u admin -p
```

### Работа с базой:

```sql
USE project_db;

CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE todos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255),
    category_id INT,
    status VARCHAR(50),
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

INSERT INTO categories(name) VALUES ('Home'), ('Work'), ('Study');
INSERT INTO todos(title, category_id, status) VALUES
('Wash dishes', 1, 'pending'),
('Deploy app', 2, 'in-progress'),
('Read book', 3, 'done');

SELECT todos.id, title, status, categories.name
FROM todos
JOIN categories ON categories.id = todos.category_id;
```

---

## Шаг 5. Создание Read Replica

### Наблюдения:
**Какие данные видим на реплике?**  
Все данные с основного экземпляра, только для чтения — т.к. реплика синхронизируется.

**Можно ли выполнить INSERT/UPDATE?**  
Нет. Read Replica — read‑only.

**Появляется ли новая запись после добавления на основном инстансе?**  
Да. Из-за асинхронной репликации.

### Зачем нужны Read Replicas?
- Повышение производительности (разделение нагрузки: read‑heavy).
- Повышение отказоустойчивости.
- Возможность географического распределения.

---

## Шаг 6. Подключение приложения
Вариант: приложение на EC2.

- Записывает данные в master.
- Читает данные с read replica.
- Использует 2 разных endpoint’а.

---

## Шаг 7. DynamoDB

### Проектирование таблицы
Таблица **Todos**:
- Partition key: `id`
- Sort key: `created_at` (опционально)

### Преимущества DynamoDB:
+ Высокая производительность
+ Автоскейлинг
+ Гибкая структура

### Недостатки:
– Нет связей и JOIN  
– Нужно заранее продумывать ключи и паттерны доступа

### Сценарий использования RDS + DynamoDB
- RDS используется для транзакционных операций (CRUD, связи).
- DynamoDB — для быстрых операций чтения/кеширования, хранения логов, статистики, сессий.

---

## Вывод
В ходе лабораторной работы изучены:
- Развёртывание RDS MySQL.
- Подключение и выполнение CRUD‑операций.
- Работа с Read Replica.
- Использование DynamoDB и сравнение с реляционной моделью.

---

## Источники
- AWS Documentation  
- Amazon RDS Developer Guide  
- Amazon DynamoDB Guide  
