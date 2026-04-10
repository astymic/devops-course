# Тема 8: Контейнеризація — Docker — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `08_Docker_Theory.md`.

---

## 🎯 Мета роботи

Додати до нашого `training-project` базу даних PostgreSQL, розгорнуту ізольовано в Docker-контейнері. Оновити Ansible playbook, щоб він автоматично встановлював Docker Engine та запускав контейнер бази даних паралельно з існуючим systemd-сервісом Flask.

**Контекст:** У Темі 7 ми перетворили Flask-додаток на systemd-сервіс. Але для реального додатку потрібна база даних. Замість ручного встановлення PostgreSQL на хост-систему (що робить її "унікальною сніжинкою", або Pet), ми використаємо офіційний Docker-образ. Це гарантує, що середовище бази даних буде 100% ідентичним на будь-якій машині.

---

## 🛠 Покрокова інструкція

### Крок 1: Перевірка передумов

Переконаємось, що система готова до роботи, а VM працює (після виконання Теми 7).

На **хості** (вашому ноутбуці):

```bash
cd ~/devops-course/training-project/

# Перевіряємо статус VM
vagrant status
```

Якщо VM не працює — виконайте `vagrant up`.

Підключіться до VM і перевірте, що Flask-сервіс активний:

```bash
ssh vagrant@192.168.56.10 "sudo systemctl is-active training-app"
```

**Очікуваний результат:** `active`. Якщо ні — перевірте playbook з Теми 7.

---

### Крок 2: Створення `docker-compose.yml`

Замість багаторядкових команд для Docker ми опишемо нашу інфраструктуру у файлі `docker-compose.yml`.

На **хості**, у корені директорії `training-project/` (там, де лежить `app.py`), створіть файл `docker-compose.yml`:

```yaml
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
```

Зверніть увагу: пароль не захардкоджений у файлі. Docker Compose буде шукати його у змінній оточення `POSTGRES_PASSWORD`, яка підтягнеться з файлу `.env` у цій розгорнутій директорії.

---

### Крок 3: Оновлення Git-репозиторію

Щоб Ansible міг розгорнути цей файл на сервері, він має бути в GitHub-репозиторії нашого додатку (Single Source of Truth).

На **хості**, у корені `training-project/`:

```bash
git add docker-compose.yml
git commit -m "Add docker-compose.yml for PostgreSQL database"
git push
```

---

### Крок 4: Оновлення Ansible playbook — Встановлення Docker

Тепер потрібно навчити наш Ansible встановлювати Docker. Відкрийте файл `ansible/playbook.yml` на **хості**.

Знайдіть існуючу задачу `Встановити необхідні пакети` і розширте її новими системними пакунками (або додайте нову окрему задачу після існуючого блоку `apt`):

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

> 💡 **Чому ці пакети?** `docker.io` — це Docker Engine, а `docker-compose-v2` — це плагін `docker compose` (саме через пробіл!), який дозволяє запускати сервіси з нашого `.yml` файлу. Додавання `training` до групи `docker` відображає принцип Least Privilege: якщо в майбутньому знадобиться запускати контейнери без `sudo`, ізольований сервісний акаунт буде готовий.

> 📍 **Кроки 4–6 — підготовка на хості.** Ми редагуємо `playbook.yml`, описуємо нові задачі і оновлюємо секрети. Жодних змін на VM ще не відбувається. Застосуємо всі зміни однією командою у **Кроці 7**.

---

### Крок 5: Розширення .env файлу секретами

У `ansible/playbook.yml` знайдіть задачу `Створити файл .env`. Вона виглядає так:

```yaml
    - name: Створити файл .env
      copy:
        content: "PORT=5000\n"
        # ...
```

Змініть параметр `content`, додавши **змінну** для пароля бази даних:

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

> ⚠️ **Чому змінна, а не пряме значення?** Якби ми вписали пароль у `playbook.yml` — він потрапив би в Git і став публічним. `{{ db_password }}` — це змінна Ansible, яку ми передаємо окремо при кожному запуску команди. Секрети ніколи не живуть у Git.

Тепер playbook запускається з явною передачею пароля:

```bash
# З каталогу ansible/ на хості:
ansible-playbook playbook.yml -e "db_password=ВашПароль123"
```

> 💡 **Погляд у майбутнє (CI/CD):** Вводити пароль вручну в терміналі щоразу — це навчальний (проміжний) етап. У наступній Темі 9 ми налаштуємо автоматичні системи (GitHub Actions): пароль буде зберігатися в спеціальному надійному сховищі GitHub Secrets, і сервер сам підставлятиме його в цю ж саму команду (`-e "db_password=${{ secrets.POSTGRES_PASSWORD }}"`), не розкриваючи його.

> 💡 **Ідемпотентність (як Ansible уникає зайвих дій):** Коли ви запустите playbook вдруге з тим самим паролем, Ansible зазирне у файл `.env` на сервері. Якщо там уже є цей пароль і нічого не мінялося — він нічого не перезаписуватиме, а лише покаже статус `ok`. Якщо ж ви передасте інший пароль при запуску — він підмінить файл і покаже `changed`.
---

### Крок 6: Запуск контейнера через Ansible

Наприкінці списку `tasks:` (перед `handlers:`) додайте завдання для запуску PostgreSQL через `docker compose`:

```yaml
    - name: Запустити PostgreSQL контейнер
      command: docker compose up -d
      args:
        chdir: "{{ app_dir }}"
```

> 💡 **Як це працює?** Параметр `chdir` вказує Ansible спочатку перейти в директорію `/opt/training-app/` (де знаходиться клонований з GitHub `docker-compose.yml`), а вже потім виконати `docker compose up -d`. Прапорець `-d` запускає контейнер у фоновому режимі (detached mode).

> ⚠️ **Ця задача завжди показує `changed`** — модуль `command:` не «вміє» перевіряти стан Docker без додаткових плагінів. Це відомий компроміс для нашого курсу. На практиці для повної ідемпотентності використовують колекцію `community.docker`, але для навчальних цілей поточного рішення достатньо.

Перевірте синтаксис playbook:

```bash
# З каталогу ansible/ на хості:
ansible-playbook playbook.yml --syntax-check
```

---

### Крок 7: Розгортання оновленої інфраструктури

Тепер ми можемо запустити оновлений playbook і перевірити, як автоматично встановлюється Docker і піднімається наша база:

```bash
# З каталогу ansible/ на хості:
ansible-playbook playbook.yml -e "db_password=ВашПароль123"
```

**Очікуваний результат:** Ansible виконає всі попередні задачі швидко (`ok`), оскільки стан не змінився (ідемпотентність!). Але на нових задачах (`Встановити Docker`, оновлення `.env`, `Запустити PostgreSQL`) статус буде `changed`.

---

### Крок 8: Перевірка результатів на сервері

Скрипт відпрацював. Час зайти на VM і подивитися, як все виглядає зсередини.

На **хості**:

```bash
ssh vagrant@192.168.56.10
```

Переконаємося, що Docker Engine зареєстровано у systemd і запускається автоматично:

```bash
sudo systemctl is-enabled docker
sudo systemctl status docker --no-pager
```

**Очікуваний результат:** `enabled` і `active (running)`. Саме завдяки цьому після перезавантаження VM systemd піднімає Docker daemon, а Docker — контейнери з `restart: unless-stopped`. Ланцюжок: `systemd → Docker daemon → PostgreSQL-контейнер`.

Тепер перевіряємо чи запущений Docker контейнер:

```bash
# Пам'ятайте, що виконуємо це у робочій директорії додатка з sudo (бо Docker Engine вимагає прав)
sudo bash -c 'cd /opt/training-app && docker compose ps'
```

**Очікуваний результат:**

```text
NAME                     IMAGE         COMMAND                  SERVICE   CREATED         STATUS         PORTS
training-app-db-1        postgres:16   "docker-entrypoint.s…"   db        ...             Up ...         0.0.0.0:5432->5432/tcp
```

Як бачимо, сервіс `db` активний, а порт `5432` прокинуто назовні.

Перевіримо, чи дійсно база слухає на порту 5432:

```bash
# Перевіряємо відкриті порти на VM
ss -tuln | grep 5432
```

**Очікуваний результат:**
```text
tcp   LISTEN 0      4096         0.0.0.0:5432       0.0.0.0:*
tcp   LISTEN 0      4096            [::]:5432          [::]:*
```

Також перевіримо, чи продовжує працювати наш Flask-додаток:

```bash
curl http://localhost:5000/health
```

**Очікуваний результат:**
```json
{"status":"ok"}
```

---

### Крок 9: Перевірка persistent даних (Volumes)

У теорії ми вивчили, що контейнери — тимчасові (ephemeral). Але завдяки `volumes` у `docker-compose.yml`, наші дані зберігаються на хості. Зараз ми **власноруч** це доведемо.

Підключіться до VM, якщо ще не там:

```bash
ssh vagrant@192.168.56.10
```

**Крок 9.1 — Запишемо дані в базу.** Підключимося до PostgreSQL всередині контейнера і створимо тестовий запис:

```bash
# Підключаємось до psql всередині контейнера
sudo docker exec -it \
  $(sudo docker compose -f /opt/training-app/docker-compose.yml ps -q db) \
  psql -U training_user -d training_db
```

У консолі `psql` (запрошення `training_db=#`) виконайте:

```sql
CREATE TABLE test (id SERIAL, note TEXT);
INSERT INTO test (note) VALUES ('Ці дані мають вижити після перезапуску!');
SELECT * FROM test;
\q
```

**Очікуваний результат:**

```text
 id |               note
----+-----------------------------------
  1 | Ці дані мають вижити після перезапуску!
```

**Крок 9.2 — Знищимо контейнер.** Тепер зупинимо і видалимо контейнер разом з його файловою системою:

```bash
sudo bash -c 'cd /opt/training-app && docker compose down'
```

Контейнер видалено. Процес PostgreSQL більше не існує.

**Крок 9.3 — Відновимо та перевіримо.** Піднімемо новий контейнер з того самого образу:

```bash
sudo bash -c 'cd /opt/training-app && docker compose up -d'
sleep 3

# Підключаємось до нового контейнера і перевіряємо дані
sudo docker exec -it \
  $(sudo docker compose -f /opt/training-app/docker-compose.yml ps -q db) \
  psql -U training_user -d training_db -c "SELECT * FROM test;"
```

**Очікуваний результат:** той самий рядок — `Ці дані мають вижити після перезапуску!`. Контейнер новий, але **volume** залишився на хості і був підключений знову.

Щоб побачити, де саме Docker фізично зберігає дані:

```bash
sudo docker volume inspect training-app_postgres_data
```

У полі `Mountpoint` буде вказано фізичний шлях (зазвичай `/var/lib/docker/volumes/training-app_postgres_data/_data`). Саме там живуть файли PostgreSQL — поза контейнером, на хост-системі (VM).

> 💡 **Прибирання за собою:** Залишіть контейнер увімкненим. Вийдіть із сесії VM командою `exit`.

---

### Крок 10: Збереження коду у ваш репозиторій

Ми оновили `playbook.yml`. Його обов'язково треба зафіксувати в системі контролю версій.

На **хості**, з каталогу `training-project/`:

```bash
git add ansible/playbook.yml
git commit -m "Update Ansible playbook to install Docker and start Postgres container"
git push
```

---

## ✅ Результат виконання роботи

Після виконання всіх кроків:

- [ ] У репозиторії додатку є файл `docker-compose.yml` з налаштуваннями PostgreSQL.
- [ ] Ansible playbook оновлений і може в одну дію: встановлювати Docker, оновлювати `.env` та запускати `docker compose up -d`.
- [ ] `docker compose ps` на сервері показує активний контейнер `postgres:16`.
- [ ] Flask-додаток (з Теми 7) продовжує функціонувати як systemd-сервіс, не конфліктуючи з базою.
- [ ] Ви знаєте як знайти фізичне розміщення постійних даних бази на хост-машині.

---

## ❓ Контрольні питання

> Дайте відповіді письмово або усно перед захистом роботи.

1. Ми встановили системні пакунки `docker.io` та `docker-compose-v2` в Ansible playbook. Чому ми маємо встановлювати `docker-compose-v2` окремо від основного Engine?
2. У `docker-compose.yml` у `POSTGRES_PASSWORD` використано значення `${POSTGRES_PASSWORD}`. Як саме Docker дізнається пароль, якщо він не прописаний в самому YAML-файлі?
3. У Кроці 9 ви виконали `docker compose down`. Що при цьому відбувається з базою і чому дані не зникають назавжди?
4. У нас тепер два різні інструменти для автоматизації процесів: systemd (для Flask) та Docker Compose (для Postgres). Чому ми не запустили наш Flask додаток у Docker-контейнері і залишили його керуватися через systemd?
5. Якщо ми повністю видалимо VM командою `vagrant destroy -f` (на ноутбуці), піднімемо знову і запустимо Ansible playbook — чи буде база все ще мати старі дані (створені на попередній VM)? Чому?
