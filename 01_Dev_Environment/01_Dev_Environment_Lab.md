# Тема 1: Нам потрібна безпечна пісочниця — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `01_Dev_Environment_Theory.md`.

---

## 🎯 Мета роботи

Створити локальну Dev Environment — ізольовану Ubuntu Server VM — за принципом **Infrastructure as Code** за допомогою Vagrant та VirtualBox. Після виконання роботи ви матимете повноцінне «безпечне середовище», з яким будете працювати протягом всього курсу.

> **Що потрібно:** VirtualBox та Vagrant встановлені на вашому комп'ютері. Git. Інтернет для завантаження Vagrant Box (~500 MB).

---

## 🛠 Покрокова інструкція

> ⚠️ **Важливо: де виконувати команди?**
>
> У цій роботі є **два різних термінали**:
>
> | Термінал | Де відкритий | Які команди |  Підказка у рядку |
> | -------- | ------------ | ----------- | ----------------- |
> | 🖥️ **Хост** | Ваш комп'ютер, у папці `training-project/` | `vagrant up`, `vagrant ssh`, `vagrant halt`, `vagrant snapshot` | `user@your-pc` |
> | 🐧 **VM** | Всередині Ubuntu VM після `vagrant ssh` | `uname`, `free`, `ip addr` та інші Linux-команди | `vagrant@devops-sandbox` |
>
> `vagrant` — це програма на **вашому комп'ютері**. Вона не встановлена у VM, тому команди `vagrant halt`, `vagrant snapshot` тощо потрібно виконувати **тільки у терміналі хоста**.

### Крок 1: Встановіть VirtualBox та Vagrant

#### Windows

1. Завантажте та встановіть **VirtualBox**: [virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)

   - Оберіть «Windows hosts»
   - Запустіть інсталятор і пройдіть усі кроки (залишити налаштування за замовчуванням)
2. Завантажте та встановіть **Vagrant**: [developer.hashicorp.com/vagrant/downloads](https://developer.hashicorp.com/vagrant/downloads)

   - Оберіть «Windows» → AMD64
   - Після встановлення — **перезавантажте комп'ютер**
3. Перевірте інсталяцію у PowerShell або Command Prompt:

```bash
virtualbox --version
vagrant --version
```

**Очікуваний результат:** виводяться версії обох програм (наприклад, `7.0.x` та `2.4.x`).

#### macOS

```bash
# Встановіть Homebrew, якщо ще немає: brew.sh
brew install --cask virtualbox
brew install --cask vagrant
vagrant --version
```

#### Linux (Ubuntu/Debian)

```bash
# VirtualBox
sudo apt-get update && sudo apt-get install -y virtualbox

# Vagrant
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y vagrant
vagrant --version
```

---

### Крок 2: Клонуємо репозиторій курсу

```bash
# Клонуємо публічний студентський репозиторій
git clone https://github.com/sobol-mo/devops-course.git

# Переходимо в папку training-project — ваш робочий простір на весь курс
cd devops-course/training-project/
```

**Очікуваний результат:** У поточній теці є файл `Vagrantfile`. Перегляньте його:

```bash
cat Vagrantfile
```

Ви побачите декларативний опис нашої майбутньої VM. Зверніть увагу:

- `ubuntu/jammy64` = Ubuntu 22.04 LTS (той самий дистрибутив, що на реальних VPS)
- `memory = 2048` = 2 GB RAM
- `cpus = 2` = 2 ядра процесора
- `ip: "192.168.56.10"` = статична IP-адреса в приватній мережі хоста

---

### Крок 3: Запускаємо VM одною командою

```bash
# Ця команда читає Vagrantfile і:
# 1. Завантажує образ ubuntu/jammy64 (перший раз — ~500 MB)
# 2. Створює VM у VirtualBox з потрібними параметрами
# 3. Запускає провізіонування (apt-get update, встановлення інструментів)
vagrant up
```

> ⏳ Перший запуск займає **3-10 хвилин** — очікується завантаження образу. Наступного разу `vagrant up` займе 30-60 секунд.

**Очікуваний результат:** В кінці виводу ви побачите:

```text
✅ VM is ready! Useful commands:
   vagrant ssh   — connect to this VM
   vagrant halt  — stop the VM
   uname -a      — check OS version
```

Відкрийте VirtualBox Manager — ви побачите VM з ім'ям `devops-sandbox` зі статусом **Running**.

---

### Крок 4: Підключаємось до VM

```bash
# SSH-підключення до VM (без паролю — Vagrant налаштовує ключі автоматично)
vagrant ssh
```

**Очікуваний результат:** Ви всередині VM! Запрошення командного рядка зміниться на:

```text
vagrant@devops-sandbox:~$
```

> 📌 Зверніть увагу на `devops-sandbox` — це hostname нашої VM, заданий у Vagrantfile.

---

### Крок 5: Перевіряємо нашу Dev Environment

Виконайте наступні команди **всередині VM** та занотуйте результати:

```bash
# Перевірка операційної системи
uname -a

# Перевірка версії Ubuntu
cat /etc/os-release | grep PRETTY_NAME

# Перевірка доступної пам'яті
free -h

# Перевірка дискового простору
df -h /

# Перевірка кількості процесорів
nproc

# Перевірка мережевих інтерфейсів та IP-адрес
ip addr show
```

**Очікуваний результат:**

| Команда | Що ви маєте побачити                                      |
| -------------- | -------------------------------------------------------------------------- |
| `uname -a`   | `Linux devops-sandbox 5.15.x ... x86_64 GNU/Linux`                       |
| `os-release` | `Ubuntu 22.04.x LTS`                                                     |
| `free -h`    | Загальна пам'ять: ~2.0 Gi                                    |
| `df -h /`    | Розмір диска: ~20 GB                                            |
| `nproc`      | `2`                                                                      |
| `ip addr`    | Адреса `192.168.56.10` на одному з інтерфейсів |

---

### Крок 6: Встановлюємо зв'язок з хостом

Перевіримо, що VM доступна з вашого хост-комп'ютера. **Відкрийте новий термінал на хост-машині** (не у VM):

```bash
# Пінг VM за статичною IP-адресою
ping 192.168.56.10
```

**Очікуваний результат:** Відповіді `64 bytes from 192.168.56.10` — VM доступна з хоста.

---

### Крок 7: Робимо перший знімок стану (Snapshot)

Виконайте **на хост-машині** (поверніться з VM командою `exit`):

```bash
# Виходимо з VM
exit

# Зберігаємо поточний стан VM як "чисту установку"
vagrant snapshot save clean-install
```

**Чому це важливо?** Якщо у наступних темах ви щось зламаєте — одна команда поверне все назад:

```bash
# Відновлення до чистого стану (якщо щось пішло не так)
vagrant snapshot restore clean-install
```

Це `git commit` для вашої VM — ще один практичний приклад принципу IaC.

---

### Крок 8: Зупиняємо VM

Після роботи завжди зупиняйте VM, щоб не навантажувати комп'ютер.

> 🖥️ **Цей крок — на хост-машині.** Якщо ви зараз всередині VM — спочатку вийдіть:

```bash
# Якщо ви всередині VM — виходимо назад на хост
exit
```

Тепер ваш термінал знов на хості (рядок знову показує `user@your-pc`). Виконайте:

```bash
# Зупинити VM (зберігає стан, можна продовжити потім командою vagrant up)
vagrant halt
```

**Переконайтесь**, що VirtualBox Manager показує VM зі статусом **Powered Off**.

---

## ✅ Результат виконання роботи

Після виконання всіх кроків ви повинні мати:

- [X] VirtualBox та Vagrant встановлені та перевірені
- [X] VM `devops-sandbox` (Ubuntu 22.04 LTS, 2 vCPU, 2 GB RAM) створена з `Vagrantfile`
- [X] Успішне підключення по `vagrant ssh`
- [X] Перевірені системні параметри (`uname`, `free`, `df`, `ip addr`)
- [X] VM доступна з хоста по IP `192.168.56.10`
- [X] Snapshot `clean-install` збережено

```bash
# Фінальна перевірка: список знімків VM
vagrant snapshot list
```

**Очікуваний результат:** `clean-install` у списку.

---

## ❓ Контрольні питання

1. Поясніть різницю між Dev, Staging та Production середовищами. Яка головна причина, чому їх три — а не одне?
2. Чому для імітації реального VPS ми обрали VM, а не Docker-контейнер?
3. Що відбувається при `vagrant up`? Перелічіть кроки, які Vagrant виконує «під капотом».
4. Чому збереження конфігурації VM у `Vagrantfile` та Git — це Infrastructure as Code? Чим це краще за ручне налаштування через GUI VirtualBox?
5. Чим корисна функція Snapshot? Наведіть реальний сценарій, де вона рятує ситуацію.

---

## 📚 Додаткові матеріали

- [Vagrant Documentation — Getting Started](https://developer.hashicorp.com/vagrant/tutorials/getting-started) — офіційний туторіал
- [VirtualBox User Manual](https://www.virtualbox.org/manual/) — повна документація
- [Ubuntu 22.04 LTS Release Notes](https://ubuntu.com/blog/ubuntu-22-04-lts-jammy-jellyfish-is-now-available) — що нового у Jammy Jellyfish
- [«Pets vs. Cattle» — Randy Bias](https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/) — концепція, яка лежить в основі IaC
