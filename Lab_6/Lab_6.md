# Лабораторная работа №6 — Балансирование нагрузки и авто-масштабирование 

---

# 1. Создание VPC и подсетей

1. EC2 → VPC → Create VPC (или VPC Wizard).
2. Создай VPC с CIDR, например `10.0.0.0/16`.
3. Создай 2 публичные подсети в разных AZ: `10.0.1.0/24` (eu-central-1a), `10.0.2.0/24` (eu-central-1b).
4. Создай 2 приватные подсети: `10.0.11.0/24` и `10.0.12.0/24` в тех же AZ.
5. Create Internet Gateway → Attach to VPC.
6. Route Tables → для публичной route table добавь маршрут `0.0.0.0/0` → Internet Gateway и ассоциируй публичные подсети.

### Через AWS CLI
```bash
# создаём VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
# включаем name tag
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=lab6-vpc

# создаём публичные подсети
SUBNET_PUB1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone eu-central-1a --query 'Subnet.SubnetId' --output text)
SUBNET_PUB2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone eu-central-1b --query 'Subnet.SubnetId' --output text)

# интернет-шлюз
IGW=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC_ID

# route table
RTB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $RTB --subnet-id $SUBNET_PUB1
aws ec2 associate-route-table --route-table-id $RTB --subnet-id $SUBNET_PUB2
```

---

# 2. Создание и настройка виртуальной машины 

### UserData (init.sh)

```bash
#!/bin/bash
yum update -y
amazon-linux-extras install -y nginx1
systemctl enable nginx
systemctl start nginx
# простая страница для идентификации инстанса
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
echo "<html><body><h1>Instance: $INSTANCE_ID</h1><p>Private IP: $PRIVATE_IP</p></body></html>" > /usr/share/nginx/html/index.html
# добавим маршрут /load для теста
cat > /usr/share/nginx/html/load <<'EOF'
<html><body>
<h1>Simulating load</h1>
</body></html>
EOF
```

### Запуск инстанса через Console
1. EC2 → Launch instances
2. AMI: Amazon Linux 2
3. Instance type: `t3.micro`
4. Network: выбрать созданную VPC и публичную подсеть
5. Auto-assign Public IP: Enable
6. Security Group: создать новую — SSH (22) только на ваш IP; HTTP (80) 0.0.0.0/0
7. Advanced - User data
8. Включить Detailed CloudWatch monitoring 

---

# 3. Создание AMI

### В Console
1. EC2 → Instances → выбери инстанс → Actions → Image and templates → Create image.
2. Дай имя: `project-web-server-ami`.
3. Нажми Create image → дождись появления в AMIs.

### Теория — что такое AMI и snapshot
- **AMI** — шаблон инстанса: содержит конфигурацию.
- **Snapshot** — снимок EBS-тома. AMI может использовать snapshot'ы как хранилище образа.

**Варианты использования AMI:** быстрое создание одинаковых инстансов, бэкапы настроенных систем, шаблоны для ASG.

---

# 4. Launch Template

### Console
1. EC2 → Launch Templates → Create launch template
2. Name: `project-launch-template`
3. AMI: `project-web-server-ami`
4. Instance type: `t3.micro`
5. Security group: выбрать существующую
6. Advanced details: Detailed CloudWatch monitoring — Enable

### Теория
- **Launch Template** — современный способ хранить конфигурацию инстанса с версиями, можно указывать UserData, IAM role, network interfaces и пр.
- **Launch Configuration** — старый вариант, не поддерживает версионирование и некоторые возможности.

---

# 5. Target Group

### Console
1. EC2 → Target Groups → Create
2. Name: `project-target-group`
3. Type: Instances
4. Protocol: HTTP, Port: 80
5. VPC: выбрать
6. Create

### Роль Target Group
Target Group — набор целей (targets), к которым ALB отправляет трафик. Он хранит health checks (проверки работоспособности) и настройки протокола.

---

# 6. Application Load Balancer

### Console
1. EC2 → Load Balancers → Create → Application Load Balancer
2. Name: `project-alb`
3. Scheme: **Internet-facing** (доступен из интернета)
4. Select the 2 public subnets
5. Security group: использовать группу с правилом HTTP 80 (и SSH не нужен для ALB)
6. Listener: HTTP:80 → Default action: Forward → select `project-target-group`
7. Create

### Internet-facing vs Internal
- **Internet-facing**: получает публичный IP (через ELB) и доступен из интернета.
- **Internal**: доступен только внутри VPC (например, для внутреннего трафика между приложениями).

### Default action
Default action — действие, выполняемое, если ни одно правило listener'а не сработало. Типы: Forward (в target group), Redirect, Fixed response.

---

# 7. Auto Scaling Group 

### Console
1. EC2 → Auto Scaling Groups → Create Auto Scaling group
2. Name: `project-auto-scaling-group`
3. Choose launch template: `project-launch-template`
4. Network: выбрать VPC и приватные подсети (таким образом инстансы не имеют публичного IP и доступ к ним осуществляется через ALB).
5. Availability Zone distribution: Balanced — равномерно распределяет инстансы по AZ
6. Attach to existing load balancer → выбрать Target Group `project-target-group`
7. Configure group size:
   - Min: 2
   - Desired: 2
   - Max: 4
8. Create scaling policy: Target tracking — Average CPU utilization = 50%
   - Instance warm-up period = 60 seconds
9. Enable group metrics collection within CloudWatch
10. Create

### Вопрос: почему приватные подсети?
Ответ: безопасность и архитектура — инстансы не требуют публичного IP, весь входящий трафик идёт через балансировщик. Это снижает поверхность атаки и обеспечивает централизованный контроль.

### Availability Zone distribution
Это настройка для распределения инстансов по зонам доступности. При одной AZ падение затронет только часть инстансов, а распределение повышает отказоустойчивость.

### Что такое Instance warm-up period?
Время, в течение которого новые инстансы считаются "разогревающимися" и их метрики не учитываются сразу в правилах масштабирования. Это предотвращает ложное масштабирование до стабильного состояния.

---

# 8. Тестирование ALB

1. Копируем DNS name
2. Открывается в браузере — nginx страница.
3. При обновлении видим разные IP-адреса backend'ов. Это доказывает, что ALB балансирует трафик между экземплярами.

---

# 9. Тестирование Auto Scaling 

### Скрипт `curl.sh`
```bash
THREADS=${1:-10}
DURATION=${2:-60}
URL=${3:-http://project-alb-591803537.eu-central-1.elb.amazonaws.com/}

end=$((SECONDS + DURATION))
while [ $SECONDS -lt $end ]; do
  for i in $(seq 1 $THREADS); do
    curl -s $URL >/dev/null &
  done
  wait
done
```

```bash
./curl.sh 20 180 http://http://project-alb-591803537.eu-central-1.elb.amazonaws.com/load
```

### Альтернатива: `hey` или `ab` (лучше)
- `hey` (Go): `hey -z 60s -c 50 http://project-alb-591803537.eu-central-1.elb.amazonaws.com/`  
- `ab`: `ab -n 10000 -c 100 http://project-alb-591803537.eu-central-1.elb.amazonaws.com/`

### Результат
- Через 2 минуты CloudWatch показал рост CPU.
- Alarm сработал→ ASG увеличил количество инстансов до нужного уровня.

### Какую роль играет Auto Scaling?
ASG автоматически создаёт дополнительные инстансы (на основе Launch Template / AMI) при повышении нагрузки и удаляет их при снижении, тем самым поддерживая требуемую производительность и экономию средств.

---

# CloudWatch — настройка Alarm 

```bash
aws cloudwatch put-metric-alarm --alarm-name ASGHighCPU --metric-name CPUUtilization \
--namespace AWS/EC2 --statistic Average --period 60 --evaluation-periods 2 \
--threshold 50 --comparison-operator GreaterThanThreshold \
--dimensions Name=AutoScalingGroupName,Value=project-auto-scaling-group \
--alarm-actions <policy-arn-or-sns>
```

---

# 10. Завершение работы — удаление ресурсов

1. Остановить нагрузочный тест
2. Delete ALB (Load Balancer)
3. Delete Target Group
4. Delete Auto Scaling Group (удалить также инстансы)
5. Terminate EC2 Instances (если остались)
6. Deregister AMI и удалить связанные snapshots
7. Delete Launch Template
8. Delete Security Groups
9. Delete Subnets, Route Tables, Internet Gateway
10. Delete VPC

---

# Контрольные вопросы — ответы (готово для отчёта)

**1. Что такое image и чем он отличается от snapshot?**
- AMI — образ инстанса, включает конфигурацию и ссылки на snapshot'ы. Snapshot — копия EBS-тома (блок-устройство).

**2. Какие есть варианты использования AMI?**
- Шаблоны для Auto Scaling, резервные образы, быстрое клонирование окружений.

**3. Что такое Launch Template и зачем он нужен? Чем он отличается от Launch Configuration?**
- Launch Template хранит конфигурацию для создания инстансов (поддерживает версии, больше опций). Launch Configuration — устаревший формат.

**4. Зачем нужен Target Group?**
- Target Group хранит набор целей (инстансы / IP / lambda), health check и настройки маршрутизации трафика.

**5. В чем разница между Internet-facing и Internal?**
- Internet-facing даёт публичный доступ; Internal — только внутри VPC.

**6. Что такое Default action и какие есть типы?**
- Default action — действие listener'а если нет правил. Типы: Forward, Redirect, Fixed response.

**7. Почему для Auto Scaling Group выбираются приватные подсети?**
- Для безопасности: инстансы не требуют публичного IP; вход идёт только через ALB.

**8. Зачем нужна настройка Availability Zone distribution?**
- Для распределения инстансов по AZ и повышения отказоустойчивости.

**9. Что такое Instance warm-up period и зачем он нужен?**
- Время, за которое инстанс достигает боевой работоспособности. Не учитывать их метрики сразу — предотвращает перепиз масштабирования.

**10. Какую роль играет Auto Scaling?**
- Обеспечивает автоматическое добавление/удаление инстансов в зависимости от метрик, гарантируя required capacity.

**11. Какие IP-адреса вы видите при тестировании ALB и почему?**
- Видны приватные IP инстансов, так как ALB пробрасывает трафик к backend'ам внутри VPC.

---
