# Тема 7: Агент має працювати 24/7 — systemd — Лабораторна робота

> **Файл для студентів.** Практична частина до теорії `07_systemd_Theory.md`.

---

## 🎯 Мета роботи

Перетворити Flask-додаток, розгорнутий у Темі 6, на справжній системний сервіс: написати unit-файл, зареєструвати його в systemd, переконатися в автозапуску після перезавантаження та спостерігати за автоматичним відновленням після збою. Наприкінці — розширити Ansible playbook так, щоб увесь цикл від порожньої VM до працюючого сервісу виконувався однією командою.

**Передумова:** виконана Лабораторна робота Теми 6. На VM існує користувач `training`, каталог `/opt/training-app/` з кодом і virtualenv, файл `/opt/training-app/.env`.

> **Підключення до VM:** У цій роботі всі дії на VM вимагають прав адміністратора (`sudo`): реєстрація сервісів, перегляд логів, перезавантаження. Тому ми підключаємось напряму як `vagrant@192.168.56.10` — цей користувач має `sudo`, на відміну від `openclaw`. Саме так і Ansible у Темі 6 взаємодіє з сервером.

---

## 🛠 Покрокова інструкція

### Крок 0: Налаштування прямого SSH-доступу до адміністративного користувача

У Темі 4 ми налаштовували SSH-ключі для підключення під власним акаунтом (`devvm-openclaw`). Зараз повторимо цей прийом для адміністративного користувача `vagrant` — того самого, якого Ansible використовує для виконання playbook.

Цей крок — одноразове налаштування. Після нього всі підключення у цій роботі виконуються однією командою `ssh vagrant@192.168.56.10`.

На **хості**, з директорії `training-project/`:

```bash
# Дізнаємося, де Vagrant зберігає свій приватний ключ для VM
VAGRANT_KEY=$(vagrant ssh-config | grep IdentityFile | awk '{print $2}')
echo "Ключ знаходиться: $VAGRANT_KEY"
```

Тепер додаємо наш публічний ключ до авторизованих ключів `vagrant`-користувача:

```bash
# Копіюємо ваш публічний ключ на VM (Тема 4!)
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=$VAGRANT_KEY" vagrant@192.168.56.10
```

**Очікуваний результат:**

```text
Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'vagrant@192.168.56.10'"
```

Перевіримо, що все працює:

```bash
ssh vagrant@192.168.56.10 "echo 'SSH-доступ налаштовано!'"
```

**Очікуваний результат:** `SSH-доступ налаштовано!` — без запиту пароля.

> 💡 **Що ми щойно зробили?** Ми повторили прийом з Теми 4, але для іншого акаунта. Різниця: `openclaw` — персональний акаунт інженера для щоденної роботи (без `sudo`). `vagrant` — адміністративний акаунт (аналог `ubuntu` на AWS або `admin` на Azure), який має `sudo` і потрібен для системних операцій.

---

### Крок 1: Перевірка стартового стану

Переконаємось, що все з Теми 6 на місці.

Підключіться до VM:

```bash
ssh vagrant@192.168.56.10
```

Перевірте наявність усіх артефактів Теми 6:

```bash
# Користувач training існує
id training

# Каталог і файли додатку на місці
ls -la /opt/training-app/

# Flask встановлено у virtualenv
/opt/training-app/venv/bin/pip show flask
```

**Очікуваний результат:** користувач `training` є, у `/opt/training-app/` присутні `app.py`, `.env`, `venv/`. Якщо щось відсутнє — запустіть `ansible-playbook playbook.yml` з каталогу `ansible/` на хості (Тема 6).

---

### Крок 2: Ручний запуск — відтворення проблеми

Перш ніж вирішувати проблему — переконаємось, що вона існує. Запустимо додаток вручну і перевіримо, що відбувається після перезавантаження сервера.

На VM запустіть Flask у фоні:

```bash
sudo -u training bash -c 'cd /opt/training-app && nohup venv/bin/python app.py > /tmp/flask.log 2>&1 &'
sleep 1
```

Перевірте, що він відповідає:

```bash
curl http://localhost:5000/health
```

**Очікуваний результат:** `{"status": "ok"}`

Тепер перезавантажимо VM — імітуємо типову виробничу ситуацію: оновлення ядра, аварійний рестарт, планове технічне обслуговування:

```bash
sudo reboot
```

Зачекайте 20–30 секунд, потім підключіться знову і перевірте:

```bash
# На хості:
ssh vagrant@192.168.56.10

# На VM:
curl http://localhost:5000/health
```

**Очікуваний результат:** `curl: (7) Failed to connect to localhost port 5000`. Після перезавантаження ніхто не запустив наш процес знову — він зник разом зі старим сеансом.

Ми власноруч відтворили проблему, яку вирішуємо сьогодні.

---

### Крок 3: Створення unit-файлу

Вийдіть з VM, якщо ще на ній. Усі наступні кроки до Кроку 8 виконуємо на **хості**, у каталозі `training-project/`.

Створимо unit-файл у каталозі `ansible/` — поряд з playbook. Він буде зберігатися в Git і розгортатися Ansible.

```bash
# На хості, у каталозі training-project/
touch ansible/training-app.service
```

Відкрийте `ansible/training-app.service` у редакторі та вставте:

```ini
[Unit]
Description=Training Flask Application
After=network.target

[Service]
Type=simple
User=training
WorkingDirectory=/opt/training-app
EnvironmentFile=/opt/training-app/.env
ExecStart=/opt/training-app/venv/bin/python app.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Перевірте, що файл створено:

```bash
cat ansible/training-app.service
```

**Очікуваний результат:** вміст файлу відповідає написаному вище — три секції `[Unit]`, `[Service]`, `[Install]`.

---

### Крок 4: Ручне розгортання unit-файлу на VM

Поки не автоматизували через Ansible — скопіюємо файл вручну, щоб зрозуміти, що відбувається «під капотом».

На **хості**:

```bash
# Копіюємо unit-файл на VM за допомогою SCP (Тема 4)
scp ansible/training-app.service vagrant@192.168.56.10:/tmp/training-app.service
```

На **VM** (`vagrant ssh`):

```bash
# Переміщуємо unit-файл у системний каталог systemd
sudo mv /tmp/training-app.service /etc/systemd/system/training-app.service

# Перевіряємо, що файл на місці
ls -la /etc/systemd/system/training-app.service
```

**Очікуваний результат:**

```text
-rw-r--r-- 1 root root 275 ... /etc/systemd/system/training-app.service
```

`root root` — власник і група. Саме так і має бути: ми підключились як `vagrant` (який має `sudo`), виконали `sudo mv` — команда запустилась від імені root. Коли root переміщує файл — файл стає власністю root.

> 💡 **Чому саме `/etc/systemd/system/`?** Це каталог для unit-файлів, які встановлені системним адміністратором. Системні unit-файли самого Ubuntu знаходяться в `/lib/systemd/system/` — їх не чіпаємо.

---

### Крок 5: Реєстрація сервісу в systemd

Після будь-яких змін unit-файлів потрібно повідомити systemd, щоб він перечитав конфігурацію.

На **VM**:

```bash
# Повідомляємо systemd про новий unit-файл
sudo systemctl daemon-reload
```

Перевіримо, що systemd «бачить» новий сервіс:

```bash
sudo systemctl status training-app
```

**Очікуваний результат:**

```text
○ training-app.service - Training Flask Application
     Loaded: loaded (/etc/systemd/system/training-app.service; disabled; ...)
     Active: inactive (dead)
```

`Loaded` — systemd знайшов і прочитав unit-файл. `disabled` — автозапуск ще не активований. `inactive (dead)` — сервіс ще не запущений. Це очікувана картина після `daemon-reload`.

---

### Крок 6: Запуск сервісу та активація автозапуску

Активуємо автозапуск і запустимо сервіс:

```bash
# Реєструємо автозапуск при завантаженні системи
sudo systemctl enable training-app

# Запускаємо сервіс прямо зараз
sudo systemctl start training-app
```

Перевіримо статус:

```bash
sudo systemctl status training-app
```

**Очікуваний результат:**

```text
● training-app.service - Training Flask Application
     Loaded: loaded (/etc/systemd/system/training-app.service; enabled; ...)
     Active: active (running) since ...
   Main PID: XXXX (python)
     Memory: ~25M
        CPU: ...
     CGroup: /system.slice/training-app.service
             └─XXXX /opt/training-app/venv/bin/python app.py
```

`enabled` — автозапуск активований. `active (running)` — сервіс працює.

Перевіримо, що додаток дійсно відповідає:

```bash
curl http://localhost:5000/health
```

**Очікуваний результат:** `{"status":"ok"}`

---

### Крок 7: Перевірка автозапуску після перезавантаження

Це ключова перевірка — переконаємось, що сервіс підніметься без нашого втручання.

На **VM**:

```bash
# Перезавантажуємо VM
sudo reboot
```

Зачекайте 20–30 секунд, потім підключіться знову:

```bash
# На хості:
ssh vagrant@192.168.56.10
```

Відразу перевіряємо статус і відповідь додатку:

```bash
sudo systemctl status training-app
curl http://localhost:5000/health
```

**Очікуваний результат:** `active (running)` і `{"status":"ok"}` — без будь-якого ручного втручання після перезавантаження.

---

### Крок 8: Перегляд логів через journalctl

systemd зберігає всі виводи додатку централізовано. Переглянемо їх.

На **VM**:

```bash
# Усі логи сервісу з самого початку
sudo journalctl -u training-app

# Останні 20 рядків
sudo journalctl -u training-app -n 20

# Логи в реальному часі (зупинити — Ctrl+C)
sudo journalctl -u training-app -f
```

Поки `journalctl -f` запущений — відкрийте **другий термінал** і зробіть кілька запитів до сервісу:

```bash
# У другому терміналі на хості:
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/"
```

**Очікуваний результат:** у першому терміналі з `journalctl -f` з'являться нові рядки — Flask логує кожен запит.

Зупиніть `journalctl -f` через `Ctrl+C`.

---

### Крок 9: Симуляція збою та автоматичне відновлення

Перевіримо директиву `Restart=on-failure` в дії. Знайдемо PID процесу і «вб'ємо» його примусово.

На **VM**, у першому терміналі запустіть моніторинг логів:

```bash
sudo journalctl -u training-app -f
```

У **другому терміналі** на VM:

```bash
# Примусово зупиняємо процес (сигнал SIGKILL — некоректне завершення)
sudo kill -9 $(systemctl show -p MainPID training-app --value)
```

Спостерігайте за першим терміналом.

**Очікуваний результат у journalctl:**

```text
systemd[1]: training-app.service: Main process exited, code=killed, status=9/KILL
systemd[1]: training-app.service: Failed with result 'signal'.
systemd[1]: training-app.service: Scheduled restart job, restart counter is at 1.
systemd[1]: Started Training Flask Application.
python[НОВИЙ_PID]: * Serving Flask app 'app'
python[НОВИЙ_PID]: * Running on http://127.0.0.1:5000
```

systemd виявив збій, зачекав 5 секунд (`RestartSec=5`) і перезапустив сервіс з новим PID.

Перевірте, що сервіс знову відповідає:

```bash
curl http://localhost:5000/health
```

> 💡 Зверніть на різницю між `kill -9` (SIGKILL, примусове знищення) та `systemctl stop` (коректна зупинка). `Restart=on-failure` спрацьовує лише при **ненормальному** завершенні процесу. `systemctl stop` — нормальне завершення, systemd не перезапускає сервіс. Спробуйте самостійно: `sudo systemctl stop training-app` — і переконайтесь, що він **не** піднявся сам.

---

### Крок 10: Інтеграція з Ansible playbook

Зараз unit-файл ми скопіювали вручну через SCP. Це суперечить принципу IaC з Теми 6: якщо дія не автоматизована — вона не відтворювана. Розширимо playbook.

На **хості**, у каталозі `training-project/`, відкрийте `ansible/playbook.yml` і додайте три задачі після встановлення Python-залежностей (після блоку `pip`):

```yaml
    - name: Скопіювати unit-файл сервісу
      copy:
        src: training-app.service
        dest: /etc/systemd/system/training-app.service
        owner: root
        group: root
        mode: '0644'
      notify:
        - Reload systemd
        - Restart training-app

    - name: Активувати автозапуск сервісу
      systemd:
        name: training-app
        enabled: yes

    - name: Запустити сервіс
      systemd:
        name: training-app
        state: started
```

Додайте секцію `handlers` після секції `tasks` (на тому самому рівні відступу, що і `tasks:`):

```yaml
  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Restart training-app
      systemd:
        name: training-app
        state: restarted
```

> 💡 **Що таке handler?** Це задача, яка виконується лише якщо її «повідомив» (`notify`) інший task. У нашому випадку: якщо unit-файл змінився — systemd перечитає конфігурацію (`Reload systemd`) і перезапустить сервіс із новими налаштуваннями (`Restart training-app`). Якщо файл не змінювався — жоден з handlers не запускається. Це ідемпотентність на рівні orchestration.

Перевіримо синтаксис:

```bash
# На хості, з каталогу ansible/
ansible-playbook playbook.yml --syntax-check
```

**Очікуваний результат:** `playbook.yml has no syntax errors`

---

### Крок 11: Перевірка повного playbook — від нуля

Перевіримо, що весь pipeline тепер відтворюваний однією командою. Знищимо VM і відновимо її повністю.

На **хості**, з каталогу `training-project/`:

```bash
# Знищуємо VM
vagrant destroy -f

# Створюємо нову VM з нуля
vagrant up
```

Зачекайте, поки VM підніметься (1–2 хвилини). Тепер запустіть playbook:

```bash
# З каталогу ansible/
ansible-playbook playbook.yml
```

**Очікуваний результат:** playbook пройшов усі задачі без `failed`. Наприкінці:

```text
PLAY RECAP *********************************************************************
devvm : ok=11   changed=10   unreachable=0    failed=0
```

Перевіримо, що сервіс працює на новій VM:

```bash
ssh vagrant@192.168.56.10 "sudo systemctl status training-app"
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"
```

**Очікуваний результат:** `active (running)` і `{"status":"ok"}` — на повністю новій VM, без жодного ручного кроку.

---

### Крок 12: Збереження змін у Git

Unit-файл і оновлений playbook мають жити в Git — це єдине джерело правди для нашого `training-project`.

На **хості**, з кореня `training-project/`:

```bash
git add ansible/training-app.service ansible/playbook.yml
git commit -m "Add systemd service unit and ansible tasks for training-app"
git push
```

**Очікуваний результат:** зміни у GitHub. Тепер будь-хто з командою може відтворити повне середовище командою `vagrant up && ansible-playbook playbook.yml`.

---

## ✅ Результат виконання роботи

Після виконання всіх кроків:

- [ ] Unit-файл `training-app.service` створений і знаходиться в `/etc/systemd/system/`
- [ ] `systemctl status training-app` показує `active (running)` і `enabled`
- [ ] Після `sudo reboot` сервіс піднявся автоматично без ручного втручання
- [ ] Після `kill -9 <PID>` systemd перезапустив сервіс протягом 5 секунд
- [ ] Ansible playbook містить задачі для копіювання unit-файлу та запуску сервісу
- [ ] `vagrant destroy -f && vagrant up && ansible-playbook playbook.yml` відтворює повне середовище
- [ ] Зміни закомічено і запушено до GitHub

Фінальна перевірка на хості:

```bash
# Перевіряємо статус сервісу
ssh vagrant@192.168.56.10 "sudo systemctl status training-app --no-pager"

# Перевіряємо health-endpoint
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"

# Перевіряємо автозапуск (enabled)
ssh vagrant@192.168.56.10 "sudo systemctl is-enabled training-app"
```

---

## ❓ Контрольні питання

> Дайте відповіді письмово або усно перед захистом роботи.

1. У Кроці 2 ми запустили Flask вручну і перезавантажили VM — додаток не піднявся. Поясніть, чому це сталося з точки зору Linux-процесів.
2. Після `sudo systemctl enable training-app` ви перевірили статус і побачили `Active: inactive (dead)`. Це помилка чи очікувана поведінка? Яка команда потрібна далі?
3. У Кроці 9 ми виконали `kill -9` і сервіс перезапустився. Потім виконали `systemctl stop` — і сервіс **не** перезапустився. Чому поведінка відрізняється? Яка директива unit-файлу це контролює?
4. У Кроці 10 ми додали `handler` з `daemon-reload`. Навіщо потрібен handler, якщо можна просто додати задачу `daemon-reload` після копіювання файлу?
5. Ansible playbook тепер розгортає і код додатку, і unit-файл, і запускає сервіс. Як це пов'язано з концепцією «Single Source of Truth» з Теми 5?
6. Після `vagrant destroy -f && vagrant up && ansible-playbook playbook.yml` сервіс запрацював без жодного ручного кроку. Як це пов'язано з аналогією «Pets vs. Cattle» з Теми 6?

---

## 📚 Додаткові матеріали

- [systemd.service — документація](https://www.freedesktop.org/software/systemd/man/systemd.service.html) — повний список директив секції `[Service]`
- [journalctl — документація](https://www.freedesktop.org/software/systemd/man/journalctl.html) — всі прапори для фільтрації логів
