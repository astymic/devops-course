# Тема 9: Автоматизація конвеєра — CI/CD — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `09_CI_CD_Theory.md`.

---

## 🗺 Де ми зараз? Підсумок пройденого шляху

Перш ніж почати, зупинимось на хвилину. Ця лабораторна — не ізольоване завдання. Вона є прямим логічним продовженням усього, що ми будували з Теми 6. Подивимось, що вже зроблено і де ми знаходимось.

### Що ми маємо після Тем 6–8

| Етап / Тема          | Що ми автоматизували                                                                                                              | Результат                                                      |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **1. Vagrant**       | Створення самої віртуальної машини «з нуля»                                                                                       | `vagrant up` → створюється і стартує чиста VM Ubuntu           |
| **6. IaC (Ansible)** | Написали playbook, який налаштовує сервер: створює користувача `training`, встановлює Python, клонує код з GitHub, створює `.env` | `ansible-playbook playbook.yml` → порожній сервер стає готовим |
| **7. systemd**       | Написали unit-файл `training-app.service`, додали його розгортання в playbook                                                     | Flask працює 24/7, перезапускається після збою та reboot       |
| **8. Docker**        | Додали `docker-compose.yml` з PostgreSQL, розширили playbook                                                                      | Ansible тепер також встановлює Docker і піднімає базу даних    |

**Ключова думка:** увесь ланцюжок інфраструктури вже описано кодом. `Vagrantfile` піднімає сервер, а `playbook.yml` налаштовує на ньому систему, сервіс і базу даних.

### Поточна архітектура: три середовища і зв'язки між ними

```
💻 Хост (ноутбук)     →   ☁️ GitHub (main)   →   🖥️ VM (192.168.56.10)
   Ваш код, Ansible       Single Source of Truth    Flask (systemd) + PostgreSQL (Docker)
```

**Але є одна проблема:** хто і коли запускає `ansible-playbook`?

Поки що — ви самі, вручну. Це означає:
- Зміна коду в GitHub не потрапляє на сервер автоматично.
- Якщо ви запушили код із синтаксичною помилкою — ніхто не попередить вас до деплою.
- Немає гарантії, що останній код у `main` дійсно працездатний.

### Логічний наступний крок

Саме тут і з'являється **CI/CD**: автоматична система, яка:

1. **При кожному `git push`** перевіряє код (тести, lint) — це **CI (Continuous Integration)**.
2. **Якщо перевірки пройшли** — запускає `ansible-playbook` сама — це **CD (Continuous Delivery)**.

---

## 🎯 Мета роботи

Налаштувати перший CI/CD pipeline для `training-project`: автоматично перевіряти Ansible playbook, запускати тест Flask-додатку і підготувати безпечний deploy через GitHub Actions та Ansible. Після виконання роботи студент матиме workflow-файл, який запускається при `push` та Pull Request і не дозволяє деплоїти код, якщо перевірки не пройшли.

---

## 📁 Важливо: структура репозиторію

У цьому курсі весь ваш код знаходиться в одному репозиторії `devops-course`:

```
devops-course/               ← корінь Git-репозиторію
├── .github/
│   └── workflows/           ← тут живуть workflow-файли GitHub Actions
├── training-project/        ← код Flask-додатку (app.py, requirements.txt)
│   ├── app.py
│   ├── requirements.txt
│   ├── ansible/
│   │   ├── ansible.cfg
│   │   ├── inventory.ini
│   │   ├── playbook.yml
│   │   └── training-app.service
│   ├── docker-compose.yml   ← (якщо додавали у Темі 8)
│   └── tests/               ← тут будемо створювати тести
└── ...
```

> **Ключовий момент:** GitHub Actions шукає workflow-файли тільки в `.github/workflows/` у **корені репозиторію**, тобто в `devops-course/.github/workflows/`. Всі шляхи у workflow треба вказувати відносно цього кореня.

---

## 🛠 Покрокова інструкція

### Крок 1: Перевірка передумов

Переконаємось, що `training-project` після попередніх тем знаходиться у робочому стані.

На **хості**, з директорії де знаходиться `Vagrantfile`:

```bash
# Перевіряємо, що VM запущена
vagrant status
# Очікуваний результат: devops-sandbox running (або default running)

# Якщо не запущена — піднімаємо
vagrant up
```

Перевіряємо роботу Flask-сервісу та бази даних:

```bash
# Підключаємось до VM
vagrant ssh

# На VM:
sudo systemctl is-active training-app
# Очікуваний результат: active

curl -s http://localhost:5000/health
# Очікуваний результат: {"status":"ok"}

# Виходимо з VM
exit
```

Якщо щось не працює — спочатку запустіть playbook з Теми 8:

```bash
# Linux/macOS — з каталогу ansible/:
cd training-project/ansible/
ansible-playbook playbook.yml -e "db_password=ВашПароль123"

# Windows/WSL — з кореня репозиторію devops-course/:
wsl -d Ubuntu -- bash -c "cd /mnt/c/шлях/до/devops-course/training-project/ansible && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml -e 'db_password=ВашПароль123'"
```

---

### Крок 2: Додаємо тест для Flask-додатку

CI має перевіряти не лише Ansible playbook, а й мінімальну поведінку застосунку. У нас уже є `/health`, тому напишемо простий автоматичний тест.

На **хості**, у директорії `training-project/`:

```bash
# Створюємо каталог для тестів
mkdir -p tests

# Створюємо файл тесту
touch tests/test_app.py
```

Відкрийте `tests/test_app.py` і додайте:

```python
# Імпортуємо Flask-застосунок із головного файлу проєкту
from app import app


def test_health_endpoint():
    # Створюємо тестового клієнта Flask без запуску реального сервера
    client = app.test_client()

    # Імітуємо HTTP-запит до endpoint /health
    response = client.get("/health")

    # Перевіряємо, що endpoint відповів успішно
    assert response.status_code == 200

    # Перевіряємо, що застосунок повернув очікуваний JSON
    assert response.get_json() == {"status": "ok"}
```

`assert` означає перевірку очікуваного результату: якщо хоча б одна перевірка не виконується — `pytest` позначає тест як failed, і CI зупиняється.

Створіть файл залежностей для розробки у директорії `training-project/`:

```bash
touch requirements-dev.txt
```

Відкрийте `requirements-dev.txt` і додайте:

```text
-r requirements.txt
pytest
```

**Чому окремий `requirements-dev.txt`?** `requirements.txt` описує залежності застосунку для запуску. `pytest` потрібен лише pipeline та при розробці — але не потрібен на production-сервері.

Переконайтесь, що службові каталоги не потраплять у Git. Відкрийте `.gitignore` у корені `training-project/` і перевірте, чи є там ці рядки:

```text
.venv/
.pytest_cache/
__pycache__/
.env
venv/
```

---

### Крок 3: Локальна перевірка перед GitHub Actions

Перед тим як просити GitHub запускати перевірки, переконаємось, що вони проходять локально. Pipeline не має бути першим місцем, де ми дізнаємось про очевидну помилку.

На **хості**, у директорії `training-project/`:

```bash
# Створюємо локальне середовище
python3 -m venv .venv
source .venv/bin/activate   # Linux/macOS
# або для Windows: .venv\Scripts\activate

# Встановлюємо залежності для тестів
pip install -r requirements-dev.txt

# Запускаємо тест
pytest -q
```

**Очікуваний результат:** `1 passed`

Тепер перевіримо Ansible playbook. Це перевірка файлу `playbook.yml` як коду — без реального підключення до VM:

```bash
# Встановлюємо інструменти для перевірки Ansible
pip install ansible ansible-lint

# Перевіряємо синтаксис playbook (з каталогу training-project/ansible/)
cd ansible/
ansible-playbook playbook.yml --syntax-check

# Запускаємо linter у мінімальному профілі
ansible-lint --profile min playbook.yml
```

**Очікуваний результат:** синтаксис без помилок.

> **Для Windows/WSL:** ці команди треба виконувати у терміналі **Ubuntu (WSL)**, а не в PowerShell чи Git Bash.

> **Чому `--profile min`?** Навчальний playbook містить свідомі спрощення. Мінімальний профіль перевіряє найважливіше: YAML коректний, файл читається, playbook можна розібрати без помилок.

---

### Крок 4: Створення GitHub Actions workflow

GitHub Actions читає workflow тільки з каталогу `.github/workflows/` у **корені репозиторію** (`devops-course/`).

На **хості**, перейдіть у корінь репозиторію `devops-course/`:

```bash
cd ..   # або: cd ~/devops-course/
mkdir -p .github/workflows
touch .github/workflows/ci-cd.yml
```

Відкрийте `.github/workflows/ci-cd.yml` і додайте:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  lint:
    name: Lint Ansible
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Ansible tools
        run: pip install ansible ansible-lint

      - name: Check Ansible syntax
        working-directory: training-project/ansible
        run: ansible-playbook playbook.yml --syntax-check

      - name: Run ansible-lint
        run: ansible-lint --profile min training-project/ansible/playbook.yml

  test:
    name: Test Flask app
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Python dependencies
        run: pip install -r training-project/requirements-dev.txt

      - name: Run tests
        working-directory: training-project
        run: pytest -q

  deploy:
    name: Deploy with Ansible
    needs: [lint, test]
    if: github.ref == 'refs/heads/main' && vars.ENABLE_DEPLOY == 'true' && (github.event_name == 'push' || github.event_name == 'workflow_dispatch')
    runs-on: self-hosted

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Ansible
        run: pip install ansible

      - name: Configure SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H "${{ secrets.SERVER_IP }}" >> ~/.ssh/known_hosts

      - name: Create CI inventory
        run: |
          cat > training-project/ansible/ci_inventory.ini <<EOF
          [devservers]
          devvm ansible_host=${{ secrets.SERVER_IP }} ansible_user=${{ secrets.SERVER_USER }} ansible_ssh_private_key_file=$HOME/.ssh/deploy_key
          EOF

      - name: Run Ansible deploy
        run: |
          ANSIBLE_CONFIG=training-project/ansible/ansible.cfg \
          ansible-playbook \
            -i training-project/ansible/ci_inventory.ini \
            training-project/ansible/playbook.yml \
            -e "db_password=${{ secrets.POSTGRES_PASSWORD }}"
```

**Що тут важливо:**

| Параметр | Пояснення |
|---|---|
| `lint` і `test` | Запускаються на GitHub-hosted runner `ubuntu-latest` при кожному push/PR |
| `needs: [lint, test]` | `deploy` не стартує, якщо перевірки впали |
| `if: github.ref == 'refs/heads/main'` | `deploy` тільки для гілки `main`, не для feature-гілок |
| `vars.ENABLE_DEPLOY == 'true'` | Захисна змінна — не запустить CD без вашого дозволу |
| `runs-on: self-hosted` | `deploy` запускається на вашому ноутбуці, бо VM недоступна з хмари GitHub |
| `ANSIBLE_CONFIG=...` | Обов'язково для WSL-середовища |

---

### Крок 5: Коміт і запуск CI

Збережемо тести та workflow у Git і відправимо на GitHub.

На **хості**, у корені `devops-course/`:

```bash
git add training-project/tests/test_app.py \
        training-project/requirements-dev.txt \
        training-project/.gitignore \
        .github/workflows/ci-cd.yml
git commit -m "Add CI/CD workflow, Flask tests and dev requirements"
git push
```

Після `git push` відкрийте свій репозиторій на GitHub, перейдіть на вкладку **Actions** і відкрийте workflow **CI/CD Pipeline**.

**Очікуваний результат:**
- Jobs `Lint Ansible` і `Test Flask app` запустилися автоматично і зелені ✅
- Job `Deploy with Ansible` — пропущений (skipped), бо `ENABLE_DEPLOY` ще не увімкнено — це правильна поведінка

---

### Крок 6: Перевірка захисного механізму CI

Навмисно створимо помилку, щоб побачити, що pipeline справді зупиняє неправильні зміни.

На **хості** відкрийте `training-project/app.py` і змініть відповідь endpoint `/health`:

```python
return jsonify({"status": "broken"})
```

Зафіксуйте і відправте зміну:

```bash
git add training-project/app.py
git commit -m "Break health endpoint intentionally"
git push
```

**Очікуваний результат:** у вкладці **Actions** job `Test Flask app` впаде ❌ — тест очікує `{"status": "ok"}`, але застосунок повертає `{"status": "broken"}`. `Deploy` теж не запуститься, бо залежить від `test`.

Це і є головна цінність CI: помилка знайдена **до** того, як потрапила на сервер.

Поверніть правильну відповідь у `app.py`:

```python
return jsonify({"status": "ok"})
```

Зафіксуйте виправлення:

```bash
git add training-project/app.py
git commit -m "Fix health endpoint"
git push
```

**Очікуваний результат:** pipeline знову зелений ✅

---

### Крок 7: Підготовка секретів для CD

Навіть якщо ви не вмикаєте реальний deploy у цій лабораторній, потрібно зрозуміти, які секрети потрібні pipeline.

Створимо окремий deploy-ключ для CI/CD. Окремий ключ легше відкликати — ніколи не використовуйте особистий ключ для автоматизації.

На **хості**:

```bash
# Створюємо окрему пару ключів для CI/CD
ssh-keygen -t ed25519 -f ~/.ssh/training_ci_cd -C "training-ci-cd"

# Додаємо публічний ключ на VM (для vagrant-користувача)
# Варіант 1: якщо ssh vagrant@192.168.56.10 працює
ssh-copy-id -i ~/.ssh/training_ci_cd.pub vagrant@192.168.56.10

# Варіант 2: через vagrant ssh
vagrant ssh -c "echo '$(cat ~/.ssh/training_ci_cd.pub)' >> ~/.ssh/authorized_keys"

# Перевіряємо доступ
ssh -i ~/.ssh/training_ci_cd vagrant@192.168.56.10 "echo 'CI/CD SSH works'"
```

**Очікуваний результат:** `CI/CD SSH works` без запиту пароля.

На сторінці вашого репозиторію на GitHub:

```
Settings → Secrets and variables → Actions → Repository secrets
```

Додайте такі секрети:

| Назва секрету       | Значення                                                  |
| ------------------- | --------------------------------------------------------- |
| `SSH_PRIVATE_KEY`   | вміст файлу `~/.ssh/training_ci_cd` (приватний ключ!)    |
| `SERVER_IP`         | `192.168.56.10` (для Vagrant VM)                          |
| `SERVER_USER`       | `vagrant`                                                 |
| `POSTGRES_PASSWORD` | пароль бази, який ви передавали у Темі 8                  |

Переглянути вміст приватного ключа:

```bash
cat ~/.ssh/training_ci_cd
```

> ⚠️ Приватний ключ не комітиться в Git і не вставляється у workflow-файл. Він зберігається тільки в GitHub Secrets.

---

### Крок 8: Чому deploy поки не запускається з GitHub-hosted runner

Адреса `192.168.56.10` існує тільки у вашій локальній мережі між ноутбуком і Vagrant VM. Хмарний runner GitHub знаходиться в іншій мережі — він цю адресу не бачить.

```
Ваш ноутбук ──────────────→ Vagrant VM 192.168.56.10  (є доступ)
GitHub-hosted runner ──────X Vagrant VM 192.168.56.10  (немає доступу)
```

У реальних проєктах сервер має публічний IP або DNS-ім'я, і GitHub-hosted runner може підключитися напряму. Для нашої локальної VM потрібен **self-hosted runner** — він встановлюється на ноутбук і є «мостом»: бачить і GitHub, і VM одночасно.

---

### Крок 9: Опційно — запуск CD через self-hosted runner

Цей крок потрібен, якщо викладач вимагає повністю автоматичний deploy на локальну VM.

> **💡 Архітектурна довідка:** У великих компаніях внутрішні сервери зазвичай недоступні з інтернету. Self-hosted Runner встановлюють у закритій мережі компанії. Він слухає задачі від GitHub і виконує їх зсередини мережі — точно так само, як ваш ноутбук може дістатись до VM `192.168.56.10`.

**Процес deploy через Self-hosted Runner:**

```
git push → GitHub Actions (lint + test) → Self-hosted Runner (ноутбук) → ansible-playbook → VM
```

У GitHub відкрийте:

```
Settings → Actions → Runners → New self-hosted runner
```

Оберіть вашу ОС і виконайте команди, які GitHub покаже на сторінці. Після налаштування runner з'явиться зі статусом `Idle`.

Перевірте, що runner може запускати Ansible:

```bash
ansible --version
python3 --version
```

Тепер увімкніть deploy-змінну. На GitHub:

```
Settings → Secrets and variables → Actions → Variables → New repository variable
```

Додайте:

| Назва змінної   | Значення |
| --------------- | -------- |
| `ENABLE_DEPLOY` | `true`   |

Зробіть невелику зміну або запустіть workflow вручну через `workflow_dispatch`, обравши гілку `main`.

**Очікуваний результат:** після успішних `lint` і `test` запускається `Deploy with Ansible`, Ansible підключається до VM і застосовує playbook.

Після deploy перевірте VM:

```bash
vagrant ssh -c "sudo systemctl status training-app --no-pager"
vagrant ssh -c "curl -s http://localhost:5000/health"
vagrant ssh -c "sudo bash -c 'cd /opt/training-app && docker compose ps'"
```

---

### Крок 10: Перевірка логіки Pull Request

У командній розробці ніхто не пушить код одразу в `main`. Замість цього — власна гілка + Pull Request.

Головна цінність CI тут: перш ніж хтось перевірятиме код очима, автомат перевірить чи він взагалі працює.

На **хості**:

```bash
# Переходимо в корінь репозиторію
cd ~/devops-course/   # або ваш шлях

# Створюємо нову гілку і перемикаємось на неї
git checkout -b feature/ci-readme-check

# Робимо невеличку зміну
echo "CI/CD pipeline is configured." >> README.md

# Фіксуємо і відправляємо на GitHub
git add README.md
git commit -m "Document CI/CD pipeline"
git push -u origin feature/ci-readme-check
```

**У GitHub:**

1. З'явиться кнопка **Compare & pull request** — натисніть її.
2. Переконайтесь: зливаємо `feature/ci-readme-check` → `main`.
3. Натисніть **Create pull request**.

**Очікуваний результат на сторінці PR:**
- Запустяться `Lint Ansible` і `Test Flask app` ✅
- `Deploy with Ansible` — **не з'явиться** (це правильно — деплоємо тільки після злиття в `main`)

Після того, як перевірки позеленіли — натисніть **Merge pull request**.

Після злиття оновіть локальний репозиторій:

```bash
git checkout main
git pull
```

---

## ✅ Результат виконання роботи

Після виконання лабораторної роботи у вас має бути:

- [ ] Створено `training-project/tests/test_app.py` з перевіркою `/health`
- [ ] Створено `training-project/requirements-dev.txt` для тестових залежностей
- [ ] Створено `.github/workflows/ci-cd.yml` у **корені** репозиторію `devops-course/`
- [ ] GitHub Actions запускає `lint` і `test` при `push` у `main`
- [ ] GitHub Actions запускає `lint` і `test` при Pull Request
- [ ] `deploy` не запускається, якщо `lint` або `test` впали
- [ ] Секрети для CD зберігаються у GitHub Secrets, а не в Git
- [ ] Ви розумієте, чому GitHub не може напряму виконати автодеплой на локальну Vagrant VM

Фінальна перевірка локально (з `training-project/`):

```bash
# Активуємо середовище
source .venv/bin/activate

# Тести
pytest -q

# Перевірка Ansible (з каталогу ansible/)
cd ansible/
ansible-playbook playbook.yml --syntax-check
ansible-lint --profile min playbook.yml
```

На GitHub:

```
Actions → CI/CD Pipeline → останній запуск має бути успішним ✅
```

---

## ❓ Контрольні питання

> Дайте відповіді письмово або усно перед захистом роботи.

1. Чому job `deploy` має залежати від `lint` і `test` через `needs`, а не запускатися паралельно з ними?
2. Чому `deploy` у workflow обмежений умовою `github.ref == 'refs/heads/main'`?
3. Чому GitHub-hosted runner не може напряму підключитися до Vagrant VM `192.168.56.10`?
4. Яка різниця між секретами `SSH_PRIVATE_KEY`, `POSTGRES_PASSWORD` у GitHub Secrets і файлом `.env` на сервері?
5. Чому для CI/CD краще створити окремий SSH-ключ `training_ci_cd`, а не використовувати особистий ключ розробника?
6. У Pull Request перевірки пройшли успішно, але deploy не запустився. Це помилка чи очікувана поведінка? Поясніть.
7. `ansible-lint` пройшов успішно. Чи означає це, що Flask-додаток точно працює правильно? Чому?

---

## 📚 Додаткові матеріали

- [GitHub Actions Documentation](https://docs.github.com/actions) — офіційна документація GitHub Actions
- [GitHub Actions — Self-hosted runners](https://docs.github.com/actions/hosting-your-own-runners) — коли runner має працювати у вашій мережі
- [ansible-lint Documentation](https://ansible.readthedocs.io/projects/lint/) — правила перевірки Ansible playbooks
