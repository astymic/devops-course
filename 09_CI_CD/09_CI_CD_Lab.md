# Work in progress

# Тема 9: Автоматизація конвеєра — CI/CD — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `09_CI_CD_Theory.md`.

---

## 🎯 Мета роботи

Налаштувати перший CI/CD pipeline для `training-project`: автоматично перевіряти Ansible playbook, запускати тест Flask-додатку і підготувати безпечний deploy через GitHub Actions та Ansible. Після виконання роботи студент матиме workflow-файл, який запускається при `push` та Pull Request і не дозволяє деплоїти код, якщо перевірки не пройшли.

**Контекст:** У Темах 6-8 ми вже навчилися розгортати застосунок вручну однією командою `ansible-playbook`: Ansible готує сервер, systemd запускає Flask-сервіс, Docker Compose піднімає PostgreSQL. У цій роботі ми автоматизуємо наступний рівень: не самі налаштування сервера, а момент, коли ці налаштування мають запускатися.

> **Важлива мережна реальність:** GitHub-hosted runner працює в хмарі GitHub і не бачить вашу локальну Vagrant VM з адресою `192.168.56.10`. Тому CI-частина лабораторної працює для всіх одразу, а реальний CD-деплой на локальну VM потребує self-hosted runner на вашому комп'ютері або публічного VPS. Це не помилка GitHub Actions — це нормальне обмеження приватної локальної мережі.

---

## 🛠 Покрокова інструкція

### Крок 1: Перевірка передумов

Переконаємось, що `training-project` після попередніх тем знаходиться у робочому стані.

На **хості**:

```bash
cd ~/devops-course/training-project/

# Перевіряємо, що VM запущена
vagrant status

# Перевіряємо, що сервіс з Теми 7 працює
ssh vagrant@192.168.56.10 "sudo systemctl is-active training-app"

# Перевіряємо, що health-endpoint відповідає
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"
```

**Очікуваний результат:** VM має статус `running`, сервіс повертає `active`, а `/health` відповідає `{"status":"ok"}`.

Якщо щось не працює — спочатку запустіть playbook з Теми 8:

```bash
cd ~/devops-course/training-project/ansible/
ansible-playbook playbook.yml -e "db_password=ВашПароль123"
```

---

### Крок 2: Додаємо тест для Flask-додатку

CI має перевіряти не лише Ansible playbook, а й мінімальну поведінку застосунку. У нас уже є `/health`, тому напишемо простий автоматичний тест.

На **хості**, у корені `training-project`:

```bash
# Створюємо каталог для тестів
mkdir -p tests

# Створюємо файл тесту
touch tests/test_app.py
```

Відкрийте `tests/test_app.py` і додайте:

```python
from app import app


def test_health_endpoint():
    client = app.test_client()

    response = client.get("/health")

    assert response.status_code == 200
    assert response.get_json() == {"status": "ok"}
```

Створіть файл залежностей для розробки:

```bash
touch requirements-dev.txt
```

Відкрийте `requirements-dev.txt` і додайте:

```text
-r requirements.txt
pytest
```

**Чому окремий `requirements-dev.txt`?** `requirements.txt` описує залежності застосунку для запуску, а `requirements-dev.txt` — інструменти для перевірки під час розробки та CI. `pytest` потрібен pipeline, але не обов'язково потрібен самому production-сервісу.

Переконайтесь, що локальні службові каталоги не потраплять у Git:

```bash
# Додайте ці рядки до .gitignore, якщо їх ще немає
echo -e ".venv/\n.pytest_cache/\n__pycache__/" >> .gitignore
```

---

### Крок 3: Локальна перевірка перед GitHub Actions

Перед тим як просити GitHub запускати перевірки, переконаємось, що вони проходять на нашому комп'ютері. Pipeline не має бути першим місцем, де ми дізнаємось про очевидну помилку.

На **хості**, у корені `training-project`:

```bash
# Створюємо локальне середовище, якщо його ще немає
python3 -m venv .venv
source .venv/bin/activate

# Встановлюємо залежності для тестів
pip install -r requirements-dev.txt

# Запускаємо тест
pytest -q
```

**Очікуваний результат:** один тест пройшов успішно.

Тепер перевіримо Ansible playbook:

```bash
# Встановлюємо інструменти для перевірки Ansible
pip install ansible ansible-lint

# Перевіряємо синтаксис playbook
cd ansible/
ansible-playbook playbook.yml --syntax-check

# Запускаємо linter у мінімальному профілі для першого CI
ansible-lint --profile min playbook.yml
```

**Очікуваний результат:** синтаксис без помилок, `ansible-lint` не знаходить критичних проблем.

> **Чому `--profile min`?** У реальних проєктах `ansible-lint` часто налаштовують суворіше. Але наш playbook навчальний і вже містить кілька свідомих спрощень. Мінімальний профіль перевіряє найважливіше для першого CI: чи файл читається, чи YAML коректний, чи playbook можна розібрати без помилок.

---

### Крок 4: Створення GitHub Actions workflow

Workflow-файли живуть у каталозі `.github/workflows/` у корені репозиторію. Саме Git-репозиторій є single source of truth для pipeline.

На **хості**, у корені `training-project`:

```bash
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
        run: cd ansible && ansible-playbook playbook.yml --syntax-check

      - name: Run ansible-lint
        run: ansible-lint --profile min ansible/playbook.yml

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
        run: pip install -r requirements-dev.txt

      - name: Run tests
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
          cat > ansible/ci_inventory.ini <<EOF
          [devservers]
          devvm ansible_host=${{ secrets.SERVER_IP }} ansible_user=${{ secrets.SERVER_USER }} ansible_ssh_private_key_file=$HOME/.ssh/deploy_key
          EOF

      - name: Run Ansible deploy
        run: ansible-playbook -i ansible/ci_inventory.ini ansible/playbook.yml -e "db_password=${{ secrets.POSTGRES_PASSWORD }}"
```

**Що тут важливо:**

- `lint` і `test` запускаються на GitHub-hosted runner `ubuntu-latest`.
- `deploy` має `needs: [lint, test]`, тому не стартує, якщо перевірки впали.
- `deploy` запускається тільки для `push` у `main` або ручного запуску workflow з гілки `main`.
- `deploy` додатково вимкнений змінною `ENABLE_DEPLOY`, щоб випадково не запускати CD без готової інфраструктури.
- `deploy` використовує `self-hosted`, бо локальна Vagrant VM недоступна з хмари GitHub.

---

### Крок 5: Коміт і запуск CI

Збережемо тести та workflow у Git і відправимо на GitHub.

На **хості**, у корені `training-project`:

```bash
git add tests/test_app.py requirements-dev.txt .github/workflows/ci-cd.yml .gitignore
git commit -m "Add CI/CD workflow for training project"
git push
```

Відкрийте ваш репозиторій на GitHub:

```text
Actions → CI/CD Pipeline
```

**Очікуваний результат:** jobs `Lint Ansible` і `Test Flask app` запустилися автоматично. Job `Deploy with Ansible` має бути пропущений, якщо `ENABLE_DEPLOY` не увімкнено.

---

### Крок 6: Перевірка захисного механізму CI

Навмисно створимо помилку в тесті, щоб побачити, що pipeline справді зупиняє неправильні зміни.

На **хості** змініть у `tests/test_app.py` очікуваний статус:

```python
assert response.get_json() == {"status": "broken"}
```

Зафіксуйте і відправте зміну:

```bash
git add tests/test_app.py
git commit -m "Break health endpoint test intentionally"
git push
```

Відкрийте `Actions` на GitHub.

**Очікуваний результат:** job `Test Flask app` має впасти. Це добре: CI знайшов проблему до деплою.

Тепер поверніть правильне значення:

```python
assert response.get_json() == {"status": "ok"}
```

Зафіксуйте виправлення:

```bash
git add tests/test_app.py
git commit -m "Fix health endpoint test"
git push
```

**Очікуваний результат:** pipeline знову зелений.

---

### Крок 7: Підготовка секретів для CD

Навіть якщо ви не вмикаєте реальний deploy у цій лабораторній, потрібно зрозуміти, які секрети потрібні pipeline.

Створимо окремий deploy-ключ для CI/CD. Не використовуйте особистий ключ без потреби: окремий ключ легше відкликати.

На **хості**:

```bash
# Створюємо окрему пару ключів для CI/CD
ssh-keygen -t ed25519 -f ~/.ssh/training_ci_cd -C "training-ci-cd"

# Додаємо публічний ключ на VM для користувача vagrant
ssh-copy-id -i ~/.ssh/training_ci_cd.pub vagrant@192.168.56.10

# Перевіряємо доступ
ssh -i ~/.ssh/training_ci_cd vagrant@192.168.56.10 "echo 'CI/CD SSH works'"
```

**Очікуваний результат:** команда повертає `CI/CD SSH works` без запиту пароля.

У GitHub відкрийте:

```text
Settings → Secrets and variables → Actions
```

Додайте **Repository secrets**:

| Назва секрету | Значення                                                        |
| ------------------------- | ----------------------------------------------------------------------- |
| `SSH_PRIVATE_KEY`       | вміст файлу `~/.ssh/training_ci_cd`                         |
| `SERVER_IP`             | `192.168.56.10` для Vagrant VM або публічний IP VPS    |
| `SERVER_USER`           | `vagrant` для Vagrant VM                                           |
| `POSTGRES_PASSWORD`     | пароль бази, який ви передавали у Темі 8 |

Щоб переглянути приватний ключ для копіювання в GitHub Secret:

```bash
cat ~/.ssh/training_ci_cd
```

> **Увага:** приватний ключ не комітиться в Git і не вставляється у workflow-файл. Він зберігається тільки в GitHub Secrets.

---

### Крок 8: Чому deploy поки не запускається з GitHub-hosted runner

Перевіримо логіку мережі. Адреса `192.168.56.10` існує тільки у вашій локальній мережі між ноутбуком і Vagrant VM. Хмарний runner GitHub знаходиться в іншій мережі, тому для нього ця адреса недосяжна.

```text
Ваш ноутбук ───────────────→ Vagrant VM 192.168.56.10
      ↑                         доступ є
      │
GitHub-hosted runner ──────X    доступу немає
```

Тому в нашому workflow:

```yaml
runs-on: self-hosted
```

для job `deploy`. Це означає: deploy має виконувати runner, який встановлений у мережі, де VM доступна. Для лабораторії це може бути ваш ноутбук. Для реального production — GitHub-hosted runner може працювати напряму, якщо сервер має публічний IP і firewall дозволяє SSH.

---

### Крок 9: Опційно — запуск CD через self-hosted runner

Цей крок потрібен, якщо викладач вимагає повністю автоматичний deploy на локальну VM. Якщо мета заняття — CI та розуміння CD-архітектури, цей крок можна виконувати демонстраційно.

У GitHub відкрийте:

```text
Settings → Actions → Runners → New self-hosted runner
```

Оберіть вашу ОС і виконайте команди, які GitHub покаже на сторінці. Після налаштування runner має з'явитися у списку зі статусом `Idle`.

На **хості** переконайтесь, що runner може запускати Ansible:

```bash
ansible --version
python3 --version
```

Якщо Ansible не встановлено глобально — workflow встановить його через `pip install ansible`, але сам runner має мати Python.

Тепер увімкніть deploy-змінну:

```text
Settings → Secrets and variables → Actions → Variables → New repository variable
```

Додайте:

| Назва змінної | Значення |
| ------------------------- | ---------------- |
| `ENABLE_DEPLOY`         | `true`         |

Зробіть невелику зміну в `main`, наприклад у `README.md`, або запустіть workflow вручну через `workflow_dispatch`, обравши гілку `main`.

**Очікуваний результат:** після успішних `lint` і `test` запускається `Deploy with Ansible`, Ansible підключається до VM і застосовує playbook.

Після deploy перевірте VM:

```bash
ssh vagrant@192.168.56.10 "sudo systemctl status training-app --no-pager"
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"
ssh vagrant@192.168.56.10 "sudo bash -c 'cd /opt/training-app && docker compose ps'"
```

---

### Крок 10: Перевірка логіки Pull Request

Створимо окрему гілку і подивимось, що CI запускається до злиття у `main`.

На **хості**:

```bash
git checkout -b feature/ci-readme-check
echo "CI/CD pipeline is configured." >> README.md
git add README.md
git commit -m "Document CI/CD pipeline"
git push -u origin feature/ci-readme-check
```

На GitHub відкрийте Pull Request з `feature/ci-readme-check` у `main`.

**Очікуваний результат:**

- `lint` і `test` запускаються для Pull Request.
- `deploy` не запускається, бо це не `push` у `main`.
- Після успішних перевірок PR можна змерджити.

Після завершення перевірки поверніться на `main`:

```bash
git checkout main
git pull
```

---

## ✅ Результат виконання роботи

Після виконання лабораторної роботи у вас має бути:

- [ ] Створено `tests/test_app.py` з перевіркою `/health`.
- [ ] Створено `requirements-dev.txt` для тестових залежностей.
- [ ] Створено `.github/workflows/ci-cd.yml`.
- [ ] GitHub Actions запускає `lint` і `test` при `push` у `main`.
- [ ] GitHub Actions запускає `lint` і `test` при Pull Request.
- [ ] `deploy` не запускається, якщо `lint` або `test` впали.
- [ ] Секрети для CD зберігаються у GitHub Secrets, а не в Git.
- [ ] Ви розумієте, чому локальна Vagrant VM потребує self-hosted runner для реального автодеплою.

Фінальна перевірка:

```bash
# Локальні тести
pytest -q

# Локальна перевірка Ansible
cd ansible/
ansible-playbook playbook.yml --syntax-check
ansible-lint --profile min playbook.yml
```

У GitHub:

```text
Actions → CI/CD Pipeline → останній запуск має бути успішним
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
