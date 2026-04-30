# Тема 6: Ліквідація ручної праці — Infrastructure as Code — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `06_IaC_Theory.md`.

---

## 🎯 Мета роботи

Написати свій перший Ansible playbook, який автоматизує налаштування VM для `training-project`: створення користувача, встановлення залежностей, підготовка каталогів, клонування коду з GitHub та розгортання мінімального веб-додатку. Після виконання роботи студент отримає файл `playbook.yml`, який за одну команду приводить порожній сервер до готового стану — замість десятків ручних кроків з Тем 1–5.

**Контекст:** У Темах 1–5 ми налаштовували сервер вручну: створювали користувача, встановлювали пакети, налаштовували SSH, працювали з Git. Тепер ми опишемо цей самий результат у коді — і зможемо відтворити його за одну команду.

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

**Для Ubuntu/Debian (або WSL на Windows):**

```bash
sudo apt update
sudo apt install -y ansible

# Перевіряємо версію
ansible --version
```

**Для macOS:**

```bash
brew install ansible
```

**Для Windows:**

Ansible не запускається нативно на Windows. Використовуйте **WSL (Windows Subsystem for Linux)**:

1. Відкрийте **PowerShell** і виконайте: `wsl --install -d Ubuntu`
2. Після встановлення відкрийте **Ubuntu** зі Стартового меню
3. У терміналі Ubuntu виконайте команди для Ubuntu/Debian вище

> **Важливо для Windows/WSL:** Всі команди `ansible-playbook` у цій лабораторній треба запускати з **терміналу Ubuntu (WSL)**, а не з PowerShell чи Git Bash. Щоб запустити playbook через WSL з Windows, використовуйте:
>
> ```powershell
> wsl -d Ubuntu -- bash -c "cd /mnt/c/шлях/до/ansible && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml"
> ```

**Очікуваний результат:** Вивід версії Ansible, наприклад `ansible [core 2.16.x]`.

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

| Параметр                | Навіщо                                                                                                                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `inventory`             | Де шукати список серверів                                                                                                                                                                                     |
| `host_key_checking`     | Не питати «чи довіряєте серверу?» при підключенні (для лабораторної VM це безпечно; у Production завжди перевіряйте host keys)                                                                               |
| `retry_files_enabled`   | Не створювати файли-«спроби» після невдалих запусків                                                                                                                                                          |
| `become`                | За замовчуванням виконувати через `sudo`                                                                                                                                                                      |
| `become_ask_pass`       | Не питати пароль sudo                                                                                                                                                                                         |

> **Важливо для Windows/WSL:** Ansible ігнорує `ansible.cfg` у директоріях з правами 777 (такими є диски Windows у WSL). Завжди запускайте playbook явно вказуючи конфіг:
> ```bash
> ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml
> ```

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
>
> **Для Windows/WSL:** якщо Vagrant встановлений на Windows, шлях у `vagrant ssh-config` буде у форматі Windows (`C:\Users\...`). Перетворіть його на шлях WSL: `C:\Users\student\...` → `/mnt/c/Users/student/...`

**Що означає кожне поле:**

| Поле                           | Значення                                                              |
| ------------------------------ | --------------------------------------------------------------------- |
| `[devservers]`                 | Група серверів                                                        |
| `devvm`                        | Псевдонім хоста                                                       |
| `ansible_host`                 | IP-адреса VM (`192.168.56.10` з Теми 1)                               |
| `ansible_user`                 | Користувач для SSH (`vagrant` — має `sudo`)                           |
| `ansible_ssh_private_key_file` | Шлях до SSH-ключа Vagrant (з виводу `vagrant ssh-config`)            |

> **Чому `vagrant`, а не `openclaw`?** Playbook створює користувача та налаштовує каталоги — для цього потрібні права `sudo`. Користувач `vagrant` має ці права, а `openclaw` — ні.

Перевіримо зв'язок:

```bash
# З каталогу ansible/ на хості (Linux/macOS):
ansible devvm -m ping -i inventory.ini

# Для Windows/WSL — обов'язково з ANSIBLE_CONFIG:
ANSIBLE_CONFIG=./ansible.cfg ansible devvm -m ping -i inventory.ini
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

Тепер зафіксуємо додаток у Git і відправимо на GitHub — саме звідти playbook буде отримувати код:

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

> **Важливо:** замініть `<ваш-username>` на ваш реальний GitHub username!

**Розберемо, що ми написали:**

| Елемент          | Призначення                                                            |
| ---------------- | ---------------------------------------------------------------------- |
| `hosts`          | Для яких серверів (група `devservers` з inventory)                     |
| `become: yes`    | Виконувати через `sudo`                                                |
| `vars`           | Змінні — ім'я користувача, шляхи, URL репозиторію                     |
| `groups: []`     | Порожній список груп — Principle of Least Privilege                    |
| `state: present` | Декларативний опис: «користувач має існувати»                          |

> 💡 **Чому не `openclaw`?** `openclaw` — це персональний акаунт інженера для щоденної роботи. Ізольований сервісний акаунт `training` (без `sudo`, без Shell-доступу ззовні) — це принцип **Least Privilege**: процес отримує лише ті права, які йому справді потрібні.

Перевіримо синтаксис:

```bash
# Linux/macOS — з каталогу ansible/:
ansible-playbook playbook.yml --syntax-check

# Windows/WSL:
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml --syntax-check
```

**Очікуваний результат:** `playbook has no syntax errors`

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

---

### Крок 10: Додаємо розгортання коду з GitHub

Додайте:

```yaml
    - name: Клонувати репозиторій training-project
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_dir }}/training-project"
        version: main
        force: yes
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

- **`git` модуль** — клонує репозиторій на сервер. При повторному запуску Ansible перевірить, чи є нові коміти (ідемпотентність!).
- **`dest: "{{ app_dir }}/training-project"`** — Git клонує репозиторій у підпапку `training-project`. Тому у наступному кроці треба вказати правильний шлях до `requirements.txt`.
- **`become_user`** — клонуємо від імені `training`, щоб файли належали правильному користувачу.
- **`content: "PORT=5000\n"`** — файл `.env` зберігає налаштування. Він створюється Ansible напряму на сервері, бо `.env` не зберігається в Git (секрети!).

> ⚠️ Права `0600` для `.env` — тільки власник може читати. У `.env` зберігають паролі та API-ключі.

---

### Крок 11: Додаємо встановлення Python-залежностей

Додайте фінальну задачу:

```yaml
    - name: Встановити Python-залежності додатку
      pip:
        requirements: "{{ app_dir }}/training-project/requirements.txt"
        virtualenv: "{{ app_dir }}/venv"
        virtualenv_python: python3
      become_user: "{{ app_user }}"
```

Ansible сам створить virtualenv і встановить Flask у нього. Задача виконується від імені `training` (`become_user`), а не root.

---

### Крок 12: Запуск playbook — перший раз

Збережіть `playbook.yml` і запустіть:

```bash
# Linux/macOS — з каталогу ansible/:
ansible-playbook playbook.yml

# Windows/WSL — обов'язково з ANSIBLE_CONFIG та -i:
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml
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

> 💡 **Чому `ok=8`, якщо задач у playbook — 7?** Ansible автоматично додає задачу «Gathering Facts» на початку — збирає інформацію про сервер.

---

### Крок 13: Перевірка ідемпотентності — запуск вдруге

Запустіть той самий playbook ще раз:

```bash
# Linux/macOS:
ansible-playbook playbook.yml

# Windows/WSL:
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml
```

**Очікуваний результат (другий запуск):**

```text
PLAY RECAP *********************************************************************
devvm : ok=8    changed=0    unreachable=0    failed=0
```

**`changed=0`** — жодна задача нічого не змінила. Це і є **ідемпотентність**: сервер вже у бажаному стані.

---

### Крок 14: Перевірка на сервері

Підключіться до VM і перевірте, що все створено правильно.

> **Чому `vagrant`, а не `devvm-openclaw`?** Для перевірки результатів playbook нам потрібен `sudo`. Користувач `openclaw` не має цих прав.

```bash
# Перевіряємо, що користувач training існує
ssh vagrant@192.168.56.10 "id training"

# Перевіряємо структуру каталогів
ssh vagrant@192.168.56.10 "ls -la /opt/training-app/"

# Перевіряємо, що Flask встановлено у virtualenv
ssh vagrant@192.168.56.10 "sudo su - training -c '/opt/training-app/venv/bin/pip show flask'"
```

> **Якщо `ssh vagrant@192.168.56.10` не працює** — використовуйте `vagrant ssh` з директорії, де знаходиться Vagrantfile, і виконуйте команди вже на VM.

---

### Крок 15: Тестовий запуск додатку

Переконаємось, що додаток працює. Підключіться до VM:

```bash
# Увійдіть на VM одним зі способів:
ssh vagrant@192.168.56.10
# або:
vagrant ssh
```

На VM виконайте:

```bash
# Запускаємо додаток від імені training
sudo su - training -c 'cd /opt/training-app/training-project && nohup /opt/training-app/venv/bin/python app.py > /dev/null 2>&1 &'

# Чекаємо секунду
sleep 2

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

> 💡 Додаток зараз працює лише поки ми його тримаємо вручну. Як зробити, щоб він запускався автоматично — тема наступного заняття (**Тема 7: systemd**).

#### Бонус: перевірка через браузер (SSH-тунель з Теми 4)

Якщо хочете побачити додаток у браузері — скористайтесь SSH-тунелем. Відкрийте **новий термінал на хості** та створіть тунель:

```bash
ssh -L 5000:localhost:5000 devvm-openclaw
```

Тепер відкрийте у браузері:

- `http://localhost:5000` — сторінка «My Training App»
- `http://localhost:5000/health` — JSON `{"status": "ok"}`

Після перевірки — закрийте тунель (`Ctrl+C`) та зупиніть додаток.

---

## ✅ Результат виконання роботи

Після виконання всіх кроків у вас має бути:

- [ ] `training-project` містить `app.py`, `requirements.txt`, `.gitignore` та каталог `ansible/`
- [ ] Код додатку закомічено і запушено на GitHub
- [ ] Ansible playbook за одну команду налаштовує сервер: користувач, пакети, каталоги, клонування з GitHub, `.env`
- [ ] `changed=0` при повторному запуску — ідемпотентність підтверджена
- [ ] Flask-додаток працює на VM під користувачем `training`

Фінальна перевірка:

```bash
# Linux/macOS:
ansible-playbook playbook.yml

# Windows/WSL:
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml

# Має бути: changed=0
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
