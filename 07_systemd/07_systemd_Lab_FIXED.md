# Тема 7: Агент має працювати 24/7 — systemd — Лабораторна робота (FIXED)

> **Виправлена версія.** Адаптовано під реальне середовище: WSL-хост, `ANSIBLE_CONFIG`, правильні шляхи до `app.py` (symlink), `vagrant ssh` замість прямого SSH.

---

## 🎯 Мета роботи

Перетворити Flask-додаток, розгорнутий у Темі 6, на справжній системний сервіс: написати unit-файл, зареєструвати його в systemd, переконатися в автозапуску після перезавантаження та спостерігати за автоматичним відновленням після збою. Наприкінці — розширити Ansible playbook так, щоб увесь цикл від порожньої VM до працюючого сервісу виконувався однією командою.

**Передумова:** виконана Лабораторна робота Теми 6. На VM існує користувач `training`, каталог `/opt/training-app/` з кодом (`app.py` — символічне посилання), virtualenv, файл `.env`.

---

## 🛠 Покрокова інструкція

### Крок 0: Налаштування прямого SSH-доступу до vagrant

У Темі 4 ми налаштували SSH для `openclaw`. Зараз зробимо те саме для `vagrant` — адміністративного користувача з `sudo`, якого Ansible використовує для playbook.

На **хості (WSL)**, з директорії де лежить `Vagrantfile`:

```bash
cd /mnt/c/Users/chapa/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm

# Дізнаємось шлях до приватного ключа Vagrant
VAGRANT_KEY=$(vagrant ssh-config | grep IdentityFile | awk '{print $2}')
echo "Vagrant key: $VAGRANT_KEY"
```

Тепер копіюємо наш публічний ключ на VM для користувача vagrant:

```bash
# Копіюємо публічний ключ через Vagrant-ключ
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=$VAGRANT_KEY" vagrant@192.168.56.10
```

> **Якщо `ssh-copy-id` вимагає пароль:** стандартний пароль vagrant — це `vagrant`. Введіть його, якщо буде запит.

Перевіряємо:

```bash
ssh vagrant@192.168.56.10 "echo 'SSH-доступ налаштовано!'"
# Очікуваний результат: SSH-доступ налаштовано! (без пароля)
```

> 💡 Після цього кроку **всі наступні команди** `ssh vagrant@192.168.56.10` працюватимуть без пароля.

---

### Крок 1: Перевірка стартового стану

Переконаємось, що все з Теми 6 на місці.

```bash
ssh vagrant@192.168.56.10
```

На VM:

```bash
# Користувач training існує
id training

# Каталог і файли додатку на місці
ls -la /opt/training-app/

# Flask встановлено у virtualenv
/opt/training-app/venv/bin/pip show flask
```

**Очікуваний результат:** користувач `training` є, у `/opt/training-app/` є `app.py` (символічне посилання), `.env`, `venv/`, `repo/`.

> Якщо чогось не вистачає — запустіть playbook із Теми 6:
> ```bash
> cd /mnt/c/.../ansible
> ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml
> ```

---

### Крок 2: Ручний запуск — відтворення проблеми

Перш ніж вирішувати проблему — переконаємось, що вона існує.

На **VM** (`ssh vagrant@192.168.56.10`):

```bash
# Запускаємо Flask вручну від імені training
sudo -u training bash -c 'cd /opt/training-app && nohup venv/bin/python app.py > /tmp/flask.log 2>&1 &'
sleep 2

# Перевірка
curl http://localhost:5000/health
# Очікуваний результат: {"status": "ok"}
```

Тепер перезавантажуємо VM:

```bash
sudo reboot
```

Зачекайте 30 секунд, підключіться знову:

```bash
ssh vagrant@192.168.56.10
curl http://localhost:5000/health
# Очікуваний результат: Connection refused — Flask не піднявся після reboot
```

Ми власноруч відтворили проблему: процес, запущений вручну, не виживає перезавантаження.

---

### Крок 3: Створення unit-файлу

Вийдіть з VM (`exit`). Наступні кроки до Кроку 8 виконуємо на **хості (WSL)**.

На хості, у каталозі `ansible/`:

```bash
cd /mnt/c/Users/chapa/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm/ansible
```

Створіть файл `training-app.service`:

```bash
cat > training-app.service << 'EOF'
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
EOF
```

Перевірте вміст:

```bash
cat training-app.service
```

**Очікуваний результат:** три секції `[Unit]`, `[Service]`, `[Install]`.

> 💡 `WorkingDirectory=/opt/training-app` і `ExecStart=... python app.py` працює, оскільки `app.py` у цій директорії є символічним посиланням (створене playbook у Темі 6) на реальний файл у клонованому репозиторії.

---

### Крок 4: Ручне розгортання unit-файлу на VM

Поки не автоматизували через Ansible — скопіюємо файл вручну, щоб зрозуміти процес.

На **хості** (з каталогу `ansible/`):

```bash
# Копіюємо unit-файл на VM
scp training-app.service vagrant@192.168.56.10:/tmp/training-app.service
```

> **Якщо `scp` не працює** (Permission denied), використайте `vagrant scp` або зверніться до Кроку 0.

Підключаємось до VM:

```bash
ssh vagrant@192.168.56.10
```

На **VM**:

```bash
# Переміщуємо unit-файл у системний каталог
sudo mv /tmp/training-app.service /etc/systemd/system/training-app.service

# Перевірка
ls -la /etc/systemd/system/training-app.service
# -rw-r--r-- 1 root root ... /etc/systemd/system/training-app.service
```

---

### Крок 5: Реєстрація сервісу в systemd

На **VM**:

```bash
# Повідомляємо systemd про новий unit
sudo systemctl daemon-reload

# Перевіряємо
sudo systemctl status training-app
```

**Очікуваний результат:**

```text
○ training-app.service - Training Flask Application
     Loaded: loaded (/etc/systemd/system/training-app.service; disabled; ...)
     Active: inactive (dead)
```

`Loaded` — systemd знайшов файл. `disabled` — автозапуск ще не активований. `inactive (dead)` — не запущений.

---

### Крок 6: Запуск сервісу та активація автозапуску

На **VM**:

```bash
# Активуємо автозапуск
sudo systemctl enable training-app

# Запускаємо
sudo systemctl start training-app

# Перевіряємо статус
sudo systemctl status training-app
```

**Очікуваний результат:**

```text
● training-app.service - Training Flask Application
     Loaded: loaded (/etc/systemd/system/training-app.service; enabled; ...)
     Active: active (running) since ...
   Main PID: XXXX (python)
```

Перевірка:

```bash
curl http://localhost:5000/health
# {"status":"ok"}
```

---

### Крок 7: Перевірка автозапуску після перезавантаження

На **VM**:

```bash
sudo reboot
```

Зачекайте 30 секунд, підключіться:

```bash
ssh vagrant@192.168.56.10
sudo systemctl status training-app
curl http://localhost:5000/health
```

**Очікуваний результат:** `active (running)` і `{"status":"ok"}` — без жодного ручного втручання!

---

### Крок 8: Перегляд логів через journalctl

На **VM**:

```bash
# Останні 20 рядків логів
sudo journalctl -u training-app -n 20

# Логи в реальному часі (Ctrl+C для зупинки)
sudo journalctl -u training-app -f
```

Поки `journalctl -f` працює — в **другому терміналі**:

```bash
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/"
```

У першому терміналі з'являться нові рядки — Flask логує запити. Зупиніть `Ctrl+C`.

---

### Крок 9: Симуляція збою та автоматичне відновлення

На **VM** у першому терміналі:

```bash
sudo journalctl -u training-app -f
```

У **другому терміналі** на VM:

```bash
# "Вбиваємо" процес примусово
sudo kill -9 $(systemctl show -p MainPID training-app --value)
```

**Очікуваний результат у journalctl:**

```text
systemd[1]: training-app.service: Main process exited, code=killed, status=9/KILL
systemd[1]: training-app.service: Failed with result 'signal'.
systemd[1]: training-app.service: Scheduled restart job, restart counter is at 1.
systemd[1]: Started Training Flask Application.
python[НОВИЙ_PID]: * Running on http://127.0.0.1:5000
```

systemd виявив збій → зачекав 5 секунд (`RestartSec=5`) → перезапустив сервіс.

Перевірте:

```bash
curl http://localhost:5000/health
# {"status":"ok"}
```

> 💡 **Різниця:** `kill -9` = ненормальне завершення → systemd перезапускає (`Restart=on-failure`). `systemctl stop` = нормальна зупинка → systemd **не** перезапускає. Спробуйте: `sudo systemctl stop training-app` — і переконайтесь, що він не піднявся сам. Потім запустіть назад: `sudo systemctl start training-app`.

---

### Крок 10: Інтеграція з Ansible playbook

Зараз unit-файл ми скопіювали вручну. Це суперечить IaC з Теми 6. Автоматизуємо.

На **хості (WSL)**, відкрийте файл `ansible/playbook.yml` і додайте **після** задачі "Створити символічне посилання на app.py" наступні три задачі:

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

Також додайте секцію `handlers` **після** секції `tasks` (на тому самому рівні відступу):

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

Перевірте синтаксис:

```bash
cd /mnt/c/Users/chapa/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm/ansible
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml --syntax-check
```

**Очікуваний результат:** `playbook: playbook.yml` (без помилок).

---

### Крок 11: Запуск оновленого playbook

> ⚠️ **НЕ виконуйте `vagrant destroy`!** Це зламає SSH-ключ в inventory та усі налаштування. Просто запустіть playbook на існуючій VM.

На **хості (WSL)**, з каталогу `ansible/`:

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml
```

**Очікуваний результат (перший запуск з новими задачами):**

```text
PLAY RECAP ****************************************************************
devvm : ok=11   changed=3   unreachable=0    failed=0
```

Три нові задачі показали `changed` (копіювання unit-файлу, enable, start). Решта — `ok` (вже існували з Теми 6).

Запустіть playbook **вдруге** для перевірки ідемпотентності:

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini playbook.yml
```

**Очікуваний результат:** `changed=0` — сервіс вже налаштований і працює.

Перевірка:

```bash
ssh vagrant@192.168.56.10 "sudo systemctl status training-app --no-pager"
ssh vagrant@192.168.56.10 "curl -s http://localhost:5000/health"
ssh vagrant@192.168.56.10 "sudo systemctl is-enabled training-app"
```

---

### Крок 12: Збереження змін у Git

На **хості (WSL)**, з кореня `training-project/`:

```bash
cd /mnt/c/Users/chapa/Desktop/\!KHPI/DevOps/AI_ASSISTANT/devops-ai-assistant/10_Implementation/01_vm

git add ansible/training-app.service ansible/playbook.yml
git commit -m "Add systemd service unit and ansible tasks for training-app"
git push
```

**Очікуваний результат:** зміни у GitHub.

---

## ✅ Результат виконання роботи

Після виконання всіх кроків:

- [ ] Unit-файл `training-app.service` створений і знаходиться в `/etc/systemd/system/`
- [ ] `systemctl status training-app` показує `active (running)` і `enabled`
- [ ] Після `sudo reboot` сервіс піднявся автоматично без ручного втручання
- [ ] Після `kill -9 <PID>` systemd перезапустив сервіс протягом 5 секунд
- [ ] Ansible playbook містить задачі для копіювання unit-файлу та запуску сервісу
- [ ] Playbook ідемпотентний: повторний запуск показує `changed=0`
- [ ] Зміни закомічено і запушено до GitHub

---

## ❓ Контрольні питання

1. У Кроці 2 ми запустили Flask вручну і перезавантажили VM — додаток не піднявся. Поясніть, чому це сталося з точки зору Linux-процесів.
2. Після `sudo systemctl enable training-app` ви перевірили статус і побачили `Active: inactive (dead)`. Це помилка чи очікувана поведінка? Яка команда потрібна далі?
3. У Кроці 9 ми виконали `kill -9` і сервіс перезапустився. Потім виконали `systemctl stop` — і сервіс **не** перезапустився. Чому поведінка відрізняється? Яка директива unit-файлу це контролює?
4. У Кроці 10 ми додали `handler` з `daemon-reload`. Навіщо потрібен handler, якщо можна просто додати задачу `daemon-reload` після копіювання файлу?
5. Ansible playbook тепер розгортає і код додатку, і unit-файл, і запускає сервіс. Як це пов'язано з концепцією «Single Source of Truth» з Теми 5?
6. Після `vagrant destroy -f && vagrant up && ansible-playbook playbook.yml` сервіс запрацював без жодного ручного кроку. Як це пов'язано з аналогією «Pets vs. Cattle» з Теми 6?
