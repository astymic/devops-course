# Тема 8: Контейнеризація — Docker — Лабораторна робота (FIXED)

> **Виправлена версія.** Адаптовано під реальне середовище: WSL Ubuntu для Ansible, `ANSIBLE_CONFIG`, `ssh devvm-vagrant`, правильні шляхи до коду у репозиторії.

---

## 🎯 Мета роботи

Додати до нашого `training-project` базу даних PostgreSQL, розгорнуту ізольовано в Docker-контейнері. Оновити Ansible playbook, щоб він автоматично встановлював Docker Engine та запускав контейнер бази даних паралельно з існуючим systemd-сервісом Flask.

**Контекст:** У Темі 7 ми перетворили Flask-додаток на systemd-сервіс. Але для реального додатку потрібна база даних. Замість ручного встановлення PostgreSQL на хост-систему, ми використаємо офіційний Docker-образ. Це гарантує 100% ідентичне середовище бази даних на будь-якій машині.

**Передумова:** виконана Лабораторна робота Теми 7. На VM працює systemd-сервіс `training-app` з Flask-додатком.

---

## 🛠 Покрокова інструкція

### Крок 1: Перевірка передумов

Переконаємось, що VM працює і Flask-сервіс активний.

На **хості (Git Bash)**, з директорії де лежить `Vagrantfile`:

```bash
cd ~/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm

# Перевіряємо статус VM
vagrant status
# Очікуваний результат: devops-sandbox running
```

Якщо VM не працює — виконайте `vagrant up`.

Перевірте Flask-сервіс:

```bash
ssh devvm-vagrant "sudo systemctl is-active training-app"
# Очікуваний результат: active
```

> Якщо `inactive` — запустіть playbook з Теми 7:
> ```bash
> wsl -d Ubuntu -- bash -c "cd /mnt/c/Users/chapa/Desktop/'!KHPI'/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm/ansible && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml"
> ```

---

### Крок 2: Створення `docker-compose.yml`

Замість багаторядкових команд Docker, опишемо інфраструктуру у файлі `docker-compose.yml`.

На **хості**, у каталозі `training-project/` (там, де лежить `app.py`):

```bash
cd ~/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm/training-project
```

Створіть файл `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB: training_db
      POSTGRES_USER: training_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
EOF
```

Перевірте:

```bash
cat docker-compose.yml
```

> 💡 Пароль не захардкоджений — Docker Compose бере його зі змінної оточення `POSTGRES_PASSWORD`, яка підтягнеться з файлу `.env`.

---

### Крок 3: Оновлення Git-репозиторію

Щоб Ansible міг розгорнути `docker-compose.yml` на сервері, він має бути в GitHub.

На **хості**, з каталогу `10_Implementation/01_vm/`:

```bash
cd ~/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm
git add training-project/docker-compose.yml
git commit -m "Add docker-compose.yml for PostgreSQL database"
git push
```

---

### Крок 4: Оновлення Ansible playbook — Встановлення Docker

Відкрийте `ansible/playbook.yml` у редакторі.

**Після** існуючої задачі `Встановити необхідні пакети` додайте дві нові задачі:

```yaml
    - name: Встановити Docker
      apt:
        name:
          - docker.io
          - docker-compose-v2
        state: present
        update_cache: yes

    - name: Додати користувача training до групи docker
      user:
        name: "{{ app_user }}"
        groups: docker
        append: yes
```

> 💡 `docker.io` — Docker Engine, `docker-compose-v2` — плагін `docker compose`. Додавання `training` до групи `docker` — принцип Least Privilege.

> 📍 **Кроки 4–6 — підготовка на хості.** Ми лише редагуємо `playbook.yml`. Застосуємо всі зміни однією командою у **Кроці 7**.

---

### Крок 5: Розширення .env файлу секретами

У `playbook.yml` знайдіть задачу `Створити файл .env` і замініть її на:

```yaml
    - name: Створити файл .env
      copy:
        content: |
          PORT=5000
          POSTGRES_PASSWORD={{ db_password }}
        dest: "{{ app_dir }}/.env"
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0600'
      notify:
        - Restart training-app
```

> ⚠️ `{{ db_password }}` — змінна Ansible, яку передаємо через `-e` при запуску. Пароль не потрапляє в Git.

---

### Крок 6: Запуск контейнера через Ansible

У `playbook.yml` додайте **ще одну задачу** перед секцією `handlers:`. Також додайте задачу для створення symlink на `docker-compose.yml`:

```yaml
    - name: Створити символічне посилання на docker-compose.yml
      file:
        src: "{{ app_code_dir }}/docker-compose.yml"
        dest: "{{ app_dir }}/docker-compose.yml"
        state: link
        owner: "{{ app_user }}"
        group: "{{ app_user }}"

    - name: Запустити PostgreSQL контейнер
      command: docker compose up -d
      args:
        chdir: "{{ app_dir }}"
```

> 💡 `docker-compose.yml` знаходиться у клонованому репозиторії у підпапці `training-project/`. Symlink робить його доступним у `/opt/training-app/`, де також лежить `.env` з паролем. Команда `docker compose up -d` запускає контейнер у фоні.

Перевірте синтаксис (запускайте з **PowerShell**):

```powershell
wsl -d Ubuntu -- bash -c "cd /mnt/c/Users/chapa/Desktop/'!KHPI'/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm/ansible && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml --syntax-check"
```

**Очікуваний результат:** `playbook: playbook.yml` (без помилок).

---

### Крок 7: Розгортання оновленої інфраструктури

Запустіть оновлений playbook з передачею пароля (з **PowerShell**):

```powershell
wsl -d Ubuntu -- bash -c "cd /mnt/c/Users/chapa/Desktop/'!KHPI'/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm/ansible && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml -e 'db_password=Training2024'"
```

**Очікуваний результат:** Старі задачі — `ok` (ідемпотентність). Нові задачі (`Встановити Docker`, оновлення `.env`, `Запустити PostgreSQL`) — `changed`.

---

### Крок 8: Перевірка результатів на сервері

Підключіться до VM:

```bash
ssh devvm-vagrant
```

Перевірте Docker:

```bash
sudo systemctl is-enabled docker
sudo systemctl status docker --no-pager
# Очікуваний результат: enabled, active (running)
```

Перевірте контейнер:

```bash
sudo bash -c 'cd /opt/training-app && docker compose ps'
```

**Очікуваний результат:**

```text
NAME                     IMAGE         ...   STATUS    PORTS
training-app-db-1        postgres:16   ...   Up ...    0.0.0.0:5432->5432/tcp
```

Перевірте порт та Flask:

```bash
ss -tuln | grep 5432
# Очікуваний результат: tcp LISTEN ... 0.0.0.0:5432

curl http://localhost:5000/health
# Очікуваний результат: {"status":"ok"}
```

---

### Крок 9: Перевірка persistent даних (Volumes)

Доведемо, що дані зберігаються навіть після знищення контейнера.

На **VM** (`ssh devvm-vagrant`):

**Крок 9.1 — Записати дані в базу:**

```bash
sudo docker exec -it \
  $(sudo docker compose -f /opt/training-app/docker-compose.yml ps -q db) \
  psql -U training_user -d training_db
```

У консолі `psql`:

```sql
CREATE TABLE test (id SERIAL, note TEXT);
INSERT INTO test (note) VALUES ('Data survives container restart');
SELECT * FROM test;
\q
```

**Крок 9.2 — Знищити контейнер:**

```bash
sudo bash -c 'cd /opt/training-app && docker compose down'
```

**Крок 9.3 — Відновити та перевірити:**

```bash
sudo bash -c 'cd /opt/training-app && docker compose up -d'
sleep 3

sudo docker exec -it \
  $(sudo docker compose -f /opt/training-app/docker-compose.yml ps -q db) \
  psql -U training_user -d training_db -c "SELECT * FROM test;"
```

**Очікуваний результат:** дані на місці! Контейнер новий, але volume залишився.

Подивіться де Docker зберігає дані:

```bash
sudo docker volume inspect training-app_postgres_data
```

Вийдіть з VM: `exit`.

---

### Крок 10: Збереження коду у Git

На **хості**, з каталогу `10_Implementation/01_vm/`:

```bash
cd ~/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm
git add ansible/playbook.yml
git commit -m "Update playbook: Docker install, Postgres container, .env secrets"
git push
```

---

## ✅ Результат виконання роботи

Після виконання всіх кроків:

- [ ] У репозиторії є `docker-compose.yml` з налаштуваннями PostgreSQL
- [ ] Ansible playbook встановлює Docker і запускає `docker compose up -d`
- [ ] `docker compose ps` показує активний контейнер `postgres:16`
- [ ] Flask-додаток (systemd) продовжує працювати паралельно з Docker
- [ ] Дані бази зберігаються у Docker volume навіть після `docker compose down`

---

## ❓ Контрольні питання

1. Ми встановили системні пакунки `docker.io` та `docker-compose-v2` в Ansible playbook. Чому ми маємо встановлювати `docker-compose-v2` окремо від основного Engine?
2. У `docker-compose.yml` у `POSTGRES_PASSWORD` використано значення `${POSTGRES_PASSWORD}`. Як саме Docker дізнається пароль, якщо він не прописаний в самому YAML-файлі?
3. У Кроці 9 ви виконали `docker compose down`. Що при цьому відбувається з базою і чому дані не зникають назавжди?
4. У нас тепер два різні інструменти для автоматизації процесів: systemd (для Flask) та Docker Compose (для Postgres). Чому ми не запустили наш Flask додаток у Docker-контейнері і залишили його керуватися через systemd?
5. Якщо ми повністю видалимо VM командою `vagrant destroy -f`, піднімемо знову і запустимо Ansible playbook — чи буде база все ще мати старі дані? Чому?
