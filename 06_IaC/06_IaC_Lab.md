# Тема 6: Ліквідація ручної праці — Infrastructure as Code — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `06_IaC_Theory.md`.

---

## 🎯 Мета роботи

Написати свій перший Ansible playbook, який автоматизує налаштування VM для `training-project`: створення користувача, встановлення залежностей, підготовка каталогів, клонування коду з GitHub та розгортання мінімального веб-додатку. Після виконання роботи студент отримає файл `playbook.yml`, який за одну команду приводить порожній сервер до готового стану — замість десятків ручних кроків з Тем 1–5.

**Контекст:** У Темах 1–5 ми налаштовували сервер вручну: створювали користувача, встановлювали пакети, налаштовували SSH, працювали з Git. Тепер ми опишемо цей самий результат у коді — і зможемо відтворити його за одну команду. Реальний приклад такого підходу — Ansible playbook у `devops-ai-assistant/10_Implementation/ansible/`, який автоматизує налаштування Production-сервера.

---

## 🛠 Покрокова інструкція

### Крок 1: Перевірка передумов

Перш ніж писати playbook, переконаємось, що VM працює і доступна.

Виконайте на **хості** (вашому ноутбуці), з директорії `training-project/` — це та сама папка `~/devops-course/training-project/`, де лежить ваш `Vagrantfile` з Теми 1:

```bash
cd ~/devops-course/training-project/

# Перевіряємо, що VM запущена
vagrant status
# Очікуваний результат: devops-sandbox running

# Перевіряємо SSH-доступ (з Теми 4)
ssh devvm-openclaw "echo 'SSH працює'"
# Очікуваний результат: SSH працює
```

Якщо VM не запущена — виконайте `vagrant up`. Якщо SSH не працює — поверніться до Теми 4.

---

### Крок 2: Встановлення Ansible на хості

Ansible встановлюється на **ваш комп'ютер** (control node), а не на VM. Він керує VM через SSH.

```bash
# Встановлюємо Ansible (Ubuntu/Debian на хості)
sudo apt update
sudo apt install -y ansible

# Перевіряємо версію
ansible --version
```

**Очікуваний результат:** Вивід версії Ansible, наприклад `ansible [core 2.16.x]`.

**Якщо у вас macOS:**

```bash
brew install ansible
```

**Якщо у вас Windows:** використовуйте WSL (Windows Subsystem for Linux) — ті самі команди, що для Ubuntu.

---

### Крок 3: Створення структури Ansible-проєкту

Створимо директорію для наших IaC-файлів усередині `training-project`.

На **хості**, у директорії вашого `training-project`:

```bash
# Створюємо каталог для Ansible
mkdir -p ansible

# Створюємо мінімальну структуру
cd ansible
touch ansible.cfg inventory.ini playbook.yml
```

**Очікуваний результат:** У каталозі `ansible/` три порожніх файли.

Перевіримо:

```bash
ls -la
# Очікуваний результат:
# ansible.cfg
# inventory.ini
# playbook.yml
```

---

### Крок 4: Конфігурація Ansible — `ansible.cfg`

Цей файл каже Ansible, де шукати inventory і як підключатися.

Відкрийте `ansible.cfg` у редакторі та впишіть:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

**Що означає кожен параметр:**

| Параметр        | Навіщо                                                                                                                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `inventory`           | Де шукати список серверів                                                                                                                                                                     |
| `host_key_checking`   | Не питати «чи довіряєте серверу?» при підключенні (для лабораторної VM це безпечно; у Production завжди перевіряйте host keys) |
| `retry_files_enabled` | Не створювати файли-«спроби» після невдалих запусків                                                                                                                  |
| `become`              | За замовчуванням виконувати через `sudo`                                                                                                                                            |
| `become_ask_pass`     | Не питати пароль sudo                                                                                                                                                                                 |

---

### Крок 5: Inventory — опис сервера

Inventory повідомляє Ansible, на якому сервері працювати.

Спочатку дізнаємося, де Vagrant зберігає SSH-ключ для нашої VM. Виконайте з каталогу, де лежить ваш `Vagrantfile` (з Теми 1):

```bash
vagrant ssh-config
```

У виводі знайдіть рядок `IdentityFile` — це шлях до приватного ключа. Наприклад:

```text
IdentityFile /home/student/devops-sandbox/.vagrant/machines/default/virtualbox/private_key
```

Відкрийте `inventory.ini` та вкажіть цей шлях:

```ini
[devservers]
devvm ansible_host=192.168.56.10 ansible_user=vagrant ansible_ssh_private_key_file=/шлях/до/.vagrant/machines/default/virtualbox/private_key
```

> **Важливо:** замініть `/шлях/до/...` на реальний шлях з виводу `vagrant ssh-config`.

**Що означає кожне поле:**

| Поле                         | Значення                                                              |
| -------------------------------- | ----------------------------------------------------------------------------- |
| `[devservers]`                 | Група серверів                                                   |
| `devvm`                        | Псевдонім хоста                                                 |
| `ansible_host`                 | IP-адреса VM (`192.168.56.10` з Теми 1)                          |
| `ansible_user`                 | Користувач для SSH (`vagrant` — має `sudo`)              |
| `ansible_ssh_private_key_file` | Шлях до SSH-ключа Vagrant (з виводу `vagrant ssh-config`) |

> **Чому `vagrant`, а не `openclaw`?** Playbook створює користувача та налаштовує каталоги — для цього потрібні права `sudo`. Користувач `vagrant` має ці права, а `openclaw` — ні.

Перевіримо зв'язок:

```bash
# З каталогу ansible/ на хості:
ansible devvm -m ping
```

**Очікуваний результат:**

```text
devvm | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

`ping`-модуль перевіряє, що Ansible може підключитися до сервера через SSH і виконувати команди. Якщо бачите `SUCCESS` — можна продовжувати.

---

### Крок 6: Створення training-project додатку

Створимо мінімальний веб-додаток, який наш playbook буде розгортати.

**Flask** — це мікрофреймворк для Python, який дозволяє за кілька рядків коду створити веб-сервер. Нам він потрібен не заради веб-розробки, а щоб мати реалістичний додаток з `/health`-ендпоінтом, який ми будемо використовувати у наступних темах (systemd, Docker, моніторинг).

На **хості**, у корені `training-project`, створіть файл `app.py`:

```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route("/")
def index():
    return "<h1>My Training App</h1><p>Status: running</p>"

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="127.0.0.1", port=port)
```

Створимо файл залежностей:

```bash
echo "flask" > requirements.txt
```

Переконайтесь, що у `.gitignore` є рядки для файлів, які не повинні потрапити в Git:

```bash
# Додайте до .gitignore, якщо ще немає:
echo -e ".env\nvenv/" >> .gitignore
```

**Очікуваний результат:** Структура `training-project`:

```text
training-project/
├── .gitignore
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── playbook.yml
├── app.py
└── requirements.txt
```

Тепер зафіксуємо додаток у Git і відправимо на GitHub — саме звідти playbook буде отримувати код (як у реальних проєктах):

```bash
git add app.py requirements.txt .gitignore
git commit -m "Add training Flask app"
git push
```

> 💡 **Чому через Git, а не копіювання файлів?** У реальних командах ніхто не деплоїть код з ноутбука — це ненадійно і неповторювано. Код живе в Git (Single Source of Truth з Теми 5), і сервер отримує його звідти. Наш playbook буде робити те саме: клонувати репозиторій з GitHub.

---

### Крок 7: Написання playbook — створення користувача

Тепер головне — пишемо playbook. Почнемо зі створення системного користувача для додатку.

Відкрийте `ansible/playbook.yml`:

```yaml
---
- name: Налаштування сервера для training-project
  hosts: devservers
  become: yes

  vars:
    app_user: training
    app_home: /home/training
    app_dir: /opt/training-app
    app_repo: "https://github.com/<ваш-username>/training-project.git"

  tasks:
    - name: Створити користувача для додатку
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: yes
        home: "{{ app_home }}"
        groups: []
        state: present
```

**Розберемо, що ми написали:**

| Елемент     | Призначення                                                                 |
| ------------------ | -------------------------------------------------------------------------------------- |
| `hosts`          | Для яких серверів (група `devservers` з inventory)              |
| `become: yes`    | Виконувати через `sudo`                                               |
| `vars`           | Змінні — ім'я користувача, шляхи, URL репозиторію |
| `groups: []`     | Порожній список груп — Principle of Least Privilege                 |
| `state: present` | Декларативний опис: «користувач має існувати»  |

> 💡 **Чому не `openclaw`?** `openclaw` — це персональний акаунт інженера для щоденної роботи: SSH-ключі, Git-доступ, VS Code. Якщо запустити додаток під цим акаунтом і він буде скомпрометований — зловмисник отримає доступ до всього: ваших SSH-ключів, репозиторіїв, налаштувань. Ізольований сервісний акаунт `training` (без `sudo`, без Shell-доступу ззовні) — це принцип **Least Privilege**: процес отримує лише ті права, які йому справді потрібні. Саме так `www-data` запускає nginx, а `postgres` — базу даних.

Перевіримо синтаксис:

```bash
# З каталогу ansible/:
ansible-playbook playbook.yml --syntax-check
```

**Очікуваний результат:** `playbook has no syntax errors`.

---

### Крок 8: Додаємо встановлення пакетів

Розширимо playbook — додамо встановлення Python та Flask.

Додайте після задачі з користувачем:

```yaml
    - name: Встановити необхідні пакети
      apt:
        name:
          - git
          - python3
          - python3-pip
          - python3-venv
        state: present
        update_cache: yes
```

**Очікуваний результат:** Задача додана до playbook. Синтаксис — без помилок.

---

### Крок 9: Додаємо створення каталогів

Додайте наступні задачі:

```yaml
    - name: Створити директорію додатку
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'

    - name: Створити директорію для конфігурації
      file:
        path: "{{ app_home }}/.config"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0700'
```

Змінні `{{ app_dir }}`, `{{ app_user }}`, `{{ app_home }}` визначені у секції `vars` на початку playbook (Крок 7).

Директорії створюються з правильними власником та правами — `755` для робочої директорії, `700` для конфігурації.

---

### Крок 10: Додаємо розгортання коду з GitHub

Тепер головний крок — розгортаємо код додатку на сервер. Замість копіювання файлів з ноутбука, ми клонуємо репозиторій з GitHub — так, як це відбувається в реальних проєктах.

Додайте:

```yaml
    - name: Клонувати репозиторій training-project
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_dir }}"
        version: main
      become_user: "{{ app_user }}"

    - name: Створити файл .env
      copy:
        content: "PORT=5000\n"
        dest: "{{ app_dir }}/.env"
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0600'
```

**Що тут відбувається:**

- **`git` модуль** — клонує репозиторій на сервер. При повторному запуску Ansible перевірить, чи є нові коміти, і оновить код лише за потреби (ідемпотентність!).
- **`version: main`** — гілка, з якої брати код.
- **`become_user`** — клонуємо від імені `training`, щоб файли належали правильному користувачу.
- **`content: "PORT=5000\n"`** — файл `.env` зберігає налаштування додатку у форматі `КЛЮЧ=значення`. `PORT=5000` каже додатку, на якому порту слухати (згадайте порт `18789` з Теми 3 — тут та сама ідея). Це варіант модуля `copy`, який записує текст напряму у файл на сервері, без вихідного файлу на ноутбуці. Ми робимо це, бо `.env` не зберігається в Git. Подумайте, чому (це буде одне з контрольних питань).

Зверніть увагу на права `0600` для `.env` — тільки власник може читати. У реальних проєктах у `.env` зберігають паролі, API-ключі та інші секрети, тому доступ має бути обмежений.

> 💡 **Важливо:** замініть `<ваш-username>` у змінній `app_repo` (Крок 7) на ваш реальний GitHub username.

---

### Крок 11: Додаємо встановлення Python-залежностей

Додайте фінальну задачу:

```yaml
    - name: Встановити Python-залежності додатку
      pip:
        requirements: "{{ app_dir }}/requirements.txt"
        virtualenv: "{{ app_dir }}/venv"
        virtualenv_python: python3
      become_user: "{{ app_user }}"
```

Ansible сам створить virtualenv і встановить Flask у нього. Задача виконується від імені `training` (`become_user`), а не root — пакети встановлюються в ізольоване середовище користувача.

---

### Крок 12: Запуск playbook — перший раз

Збережіть `playbook.yml` і запустіть:

```bash
# З каталогу ansible/:
ansible-playbook playbook.yml
```

**Очікуваний результат (перший запуск):**

```text
PLAY [Налаштування сервера для training-project] ******************************
...
TASK [Створити користувача для додатку] ****************************************
changed: [devvm]

TASK [Встановити необхідні пакети] *********************************************
changed: [devvm]
...
PLAY RECAP *********************************************************************
devvm : ok=8    changed=7    unreachable=0    failed=0
```

Усі `changed` — це нормально для першого запуску. Ми щойно створили все з нуля.

> 💡 **Чому `ok=8`, якщо задач у playbook — 7?** Ansible автоматично додає задачу «Gathering Facts» на початку кожного play — збирає інформацію про сервер (ОС, IP-адреси, доступну пам'ять). Ця задача завжди `ok`, ніколи `changed`.

---

### Крок 13: Перевірка ідемпотентності — запуск вдруге

Запустіть той самий playbook ще раз:

```bash
ansible-playbook playbook.yml
```

**Очікуваний результат (другий запуск):**

```text
PLAY RECAP *********************************************************************
devvm : ok=8    changed=0    unreachable=0    failed=0
```

**`changed=0`** — жодна задача нічого не змінила. Сервер вже у бажаному стані. Ansible перевірив кожну задачу і вирішив, що робити нічого не потрібно.

**Це і є ідемпотентність у дії.** Спробуйте запустити bash-скрипт з теорії двічі — і порівняйте результат.

---

### Крок 14: Перевірка на сервері

Підключіться до VM і перевірте, що все створено правильно.

> **Чому `vagrant`, а не `devvm-openclaw`?** Для перевірки результатів playbook нам потрібен `sudo` (наприклад, `sudo su - training`). Користувач `openclaw` (з `devvm-openclaw`) не має цих прав. Тут ми використовуємо `vagrant` — так само, як inventory для Ansible.

```bash
# Перевіряємо, що користувач training існує
ssh vagrant@192.168.56.10 "id training"
# Очікуваний результат: uid=... gid=... groups=...

# Перевіряємо структуру каталогів
ssh vagrant@192.168.56.10 "ls -la /opt/training-app/"
# Очікуваний результат:
# drwxr-xr-x ... .git/         ← репозиторій клонований з GitHub
# -rw-r--r-- ... app.py
# -rw-r--r-- ... requirements.txt
# -rw------- ... .env           ← створений Ansible, не з Git
# drwxr-xr-x ... venv/

# Перевіряємо, що Flask встановлено у virtualenv
ssh vagrant@192.168.56.10 "sudo su - training -c '/opt/training-app/venv/bin/pip list | grep Flask'"
# Очікуваний результат: Flask x.x.x
```

---

### Крок 15: Тестовий запуск додатку

Переконаємось, що додаток працює. Запустимо його на VM і перевіримо через `curl`:

```bash
# Підключаємось до VM
ssh vagrant@192.168.56.10

# На VM: запускаємо додаток у фоні від імені training
sudo su - training -c 'cd /opt/training-app && nohup venv/bin/python app.py > /dev/null 2>&1 &'

# Чекаємо секунду, поки додаток стартує
sleep 1

# Перевіряємо головну сторінку
curl http://localhost:5000
# Очікуваний результат: <h1>My Training App</h1><p>Status: running</p>

# Перевіряємо health-ендпоінт
curl http://localhost:5000/health
# Очікуваний результат: {"status":"ok"}

# Зупиняємо додаток
sudo pkill -f 'python app.py'

# Виходимо з VM
exit
```

**Очікуваний результат:**

- `/` — HTML з текстом «My Training App» та «Status: running»
- `/health` — JSON: `{"status": "ok"}`

> 💡 Додаток зараз працює лише поки ми його тримаємо вручну. Якщо закрити термінал або перезавантажити VM — він зупиниться. Як зробити, щоб додаток працював постійно як системний сервіс — тема наступного заняття (**Тема 7: systemd**).

#### Бонус: перевірка через браузер (SSH-тунель з Теми 4)

Якщо хочете побачити додаток у браузері, а не лише через `curl` — скористайтесь SSH-тунелем, який ми вивчали в Темі 4.

Поки додаток ще працює на VM (якщо зупинили — запустіть знову командою з кроку вище), відкрийте **новий термінал на хості** та створіть тунель:

```bash
ssh -L 5000:localhost:5000 devvm-openclaw
```

Тепер відкрийте у браузері на вашому ноутбуці:

- `http://localhost:5000` — сторінка «My Training App»
- `http://localhost:5000/health` — JSON `{"status": "ok"}`

Це той самий прийом, що і в Темі 4 з портом 18789: додаток слухає на `127.0.0.1` усередині VM, а тунель пробиває безпечний канал з вашого ноутбука.

Після перевірки — закрийте тунель (`Ctrl+C`) і зупиніть додаток на VM:

```bash
ssh vagrant@192.168.56.10 "sudo pkill -f 'python app.py'"
```

---

## ✅ Результат виконання роботи

Після виконання всіх кроків у вас має бути:

- [ ] `training-project` містить `app.py`, `requirements.txt`, `.gitignore` та каталог `ansible/`
- [ ] Код додатку закомічено і запушено на GitHub
- [ ] Ansible playbook за одну команду налаштовує сервер: користувач, пакети, каталоги, клонування з GitHub, `.env`
- [ ] `changed=0` при повторному запуску — ідемпотентність підтверджена
- [ ] Flask-додаток працює на VM під користувачем `training`

Фінальна перевірка — виконайте на хості:

```bash
# З каталогу ansible/:
ansible-playbook playbook.yml

# Має бути: changed=0 (сервер вже у бажаному стані)
```

Перевірка на сервері:

```bash
# Користувач існує
ssh vagrant@192.168.56.10 "id training"

# Каталоги на місці
ssh vagrant@192.168.56.10 "ls -la /opt/training-app/"

# Flask встановлено
ssh vagrant@192.168.56.10 "sudo su - training -c '/opt/training-app/venv/bin/python -c \"import flask; print(flask.__version__)\"'"
```

---

## ❓ Контрольні питання

> Дайте відповіді письмово або усно перед захистом роботи.

1. Ви запустили `ansible-playbook playbook.yml` і отримали `changed=7`. Запустили ще раз — тепер `changed=0`. Що сталося і чому результат відрізняється?
2. У playbook ми вказали `groups: []` для користувача `training`. Чому ми не додали його до групи `sudo`? Який принцип безпеки ми застосовуємо?
3. Чому Ansible встановлюється на хості (вашому комп'ютері), а не на VM? Як він взаємодіє з сервером?
4. Уявіть, що VM зламалась. Опишіть покроково, як ви відновите середовище за допомогою `vagrant` та вашого playbook.
5. Код додатку (`app.py`, `requirements.txt`) потрапляє на сервер через `git` модуль з GitHub. Але файл `.env` створюється окремо через `copy` з `content:`. Чому `.env` не зберігається в Git разом з рештою коду?
6. У playbook є дві задачі з `become_user: "{{ app_user }}"` — клонування репозиторію та встановлення pip-залежностей. Чому ці задачі виконуються від імені `training`, а не від root?

---

## 📚 Додаткові матеріали

> *(для самостійного вивчення)*

- [Ansible Documentation — Modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html) — повний перелік вбудованих модулів Ansible
- [Ansible Documentation — Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/index.html) — офіційний посібник з написання playbook
