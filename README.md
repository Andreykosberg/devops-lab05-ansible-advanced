# Лаб. П05 — Ansible advanced: переменные, vault, handlers, j2-шаблоны + сканирование NMAP

Практика по продвинутому Ansible: плейбук, который на целевом хосте устанавливает
сканер `nmap`, рендерит список целей сканирования из переменных через Jinja2-шаблон
(с одним значением, зашифрованным `ansible-vault`), запускает `nmap` в цикле по
целям, собирает и выводит результаты (`debug`), и оповещает об изменениях через
`handlers`.

## Стенд

Подготовка стенда — **та же инфраструктура, что и в П04** («Подготовка аналогична
предыдущей лабе» из методички), переиспользована без изменений: две VM Ubuntu
Server 24.04.3 LTS в Oracle VirtualBox, NAT Network `lab4net` (`10.0.2.0/24`):

| ВМ | Роль | IP в `lab4net` | SSH с хоста |
|---|---|---|---|
| `devops-lab01` | Control node (Ansible) | `10.0.2.15` | `127.0.0.1:2223` |
| `devops-lab04-vm2` | Managed node (цель сканирования) | `10.0.2.16` | `127.0.0.1:2224` |

На `devops-lab01` уже установлен Ansible (из П04), SSH-ключ `andrey` → `runner@10.0.2.16`
и passwordless sudo для `runner` уже настроены и переиспользуются как есть.

## Содержимое

| Файл/папка | Назначение |
|---|---|
| `ansible/inventory.yml` | инвентарь в YAML-формате (группа `scanners` → `devops-lab04-vm2`) |
| `ansible/vars/scan_vars.yml` | переменные: список целей, порт, зашифрованный `ansible-vault` получатель отчёта |
| `ansible/templates/targets.txt.j2` | Jinja2-шаблон файла целей, рендерится модулем `template` |
| `ansible/playbook.yml` | плейбук: install nmap → template → debug → scan (loop) → debug → handlers |
| `screenshots/` | подтверждение работы — реальный вывод терминала (control/managed node) |
| `README.md` | этот отчёт |

## Задание и как оно выполнено

> Сделать ansible-плейбук, который на целевых хостах:
> • устанавливает сканер NMAP
> • копирует файл со списком целей для сканирования
> • запускает NMAP и сканирует цели
> • собирает результаты сканирования (stdout)
> • выводит результаты сканирования
>
> Подготовка аналогична предыдущей лабе.

Помимо минимального требования задания, плейбук дополнительно и осмысленно
демонстрирует весь набор тем лекции П05 (vars/scope, vault, debug, loops, register,
handlers, error handling, j2-шаблоны):

| # | Тема лекции | Как реализовано в `playbook.yml` |
|---|---|---|
| 1 | Install nmap | `ansible.builtin.apt`, задача выполняется условно (`when: ansible_facts['os_family'] == 'Debian'`), `register: nmap_install`, триггерит handler |
| 2 | Копирование файла целей | **не `copy`, а `ansible.builtin.template`** — рендерит `targets.txt.j2` из переменной `scan_targets` (цикл `{% for %}` в Jinja2), триггерит второй handler |
| 3 | Переменные (`vars`) | `vars_files: vars/scan_vars.yml` — `scan_targets` (список), `scan_port` |
| 4 | Vault | `scan_report_recipient` зашифрован через `ansible-vault encrypt_string` и хранится прямо в vars-файле как `!vault \|` блок; расшифровывается только в памяти при запуске с `--vault-password-file`/`--ask-vault-pass` |
| 5 | debug (var/msg/verbosity) | `debug: var: scan_targets`; `debug: msg: "...{{ scan_report_recipient }}"`; `debug: msg: "..." verbosity: 1` (видно только с `-v`) |
| 6 | Conditionals (`when`) | задача-предупреждение «список целей пуст» с `when: scan_targets \| length == 0` (в реальном прогоне корректно пропускается, т.к. список непустой) |
| 7 | Loop + register + loop_control | `loop: "{{ scan_targets }}"` с переименованным `loop_var: scan_item` → `nmap ... -oG -`, `register: scan_results` |
| 8 | Error handling | `ignore_errors: true` + `changed_when: false` + `failed_when: scan_results.rc is defined and scan_results.rc != 0` — переопределяем критерий успеха/изменения на основе кода возврата `nmap` |
| 9 | Вывод результатов | второй `loop` по `scan_results.results`, `debug: msg:` с `item.scan_item`/`item.rc`/`item.stdout` для каждой цели |
| 10 | Handlers | `notify: nmap installed notification` и `notify: targets file updated notification` — оба объявлены в `handlers:`, запускаются один раз в конце плея, только если соответствующая задача вернула `changed: true` |

### Выбор целей сканирования

Пример в методичке сканирует реальные сторонние сайты (`www.psu.ru`,
`moodle-epos.permkrai.ru`, `fast.com`). Чтобы не сканировать чужую инфраструктуру
без явной авторизации, цели сканирования заменены на собственные узлы лабораторного
стенда: `10.0.2.16` (сам managed node) и `10.0.2.15` (control node, на котором с П02
установлен nginx) — обе цели полностью принадлежат связке ВМ этой лабораторной
работы, порт 80 на обеих машинах реально открыт (nginx), что дало честный,
воспроизводимый и при этом безопасный результат сканирования.

## Инвентарь, переменные и плейбук

`ansible/inventory.yml`:
```yaml
all:
  children:
    scanners:
      hosts:
        devops-lab04-vm2:
          ansible_host: 10.0.2.16
          ansible_user: runner
```

`ansible/vars/scan_vars.yml`:
```yaml
---
scan_targets:
  - 10.0.2.16
  - 10.0.2.15
scan_port: 80
scan_report_recipient: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          62396632396230306434316439393361303561636236366264323539663232623966383635323732
          6362306165303236666438333936373931333165646631630a363566623563633134363835313534
          38393262326332313864343630633434653864613662396635396263373235356563306261653031
          6565383937613066320a613463383361613766343834323762323332313065613331306536363934
          37353765623335313032633633313737303530653438616261643565656331636632
```

`ansible/templates/targets.txt.j2`:
```jinja2
# Список целей для сканирования nmap
# Сгенерировано Ansible ({{ ansible_date_time.iso8601 }}) для хоста {{ inventory_hostname }}
{% for target in scan_targets %}
{{ target }}
{% endfor %}
```

`ansible/playbook.yml` — см. файл целиком в репозитории; структура показана в таблице
выше и на скриншоте 3.

## Развёртывание (команды, реально выполненные на control node `devops-lab01`)

```bash
# --- рабочая папка (переиспользует Ansible/SSH-ключ из П04) ---
mkdir -p ~/lab5-ansible/{vars,templates} && cd ~/lab5-ansible
# inventory.yml, vars/scan_vars.yml (без vault-части), templates/targets.txt.j2, playbook.yml

# --- пароль Vault хранится в отдельном файле вне репозитория (демо-пароль для лабы) ---
echo 'lab5-vault-demo-2026' > .vault_pass.txt && chmod 600 .vault_pass.txt

# --- шифруем значение переменной и вставляем готовый блок !vault в vars/scan_vars.yml ---
ansible-vault encrypt_string --vault-password-file .vault_pass.txt \
  'scan-report@devops-lab05.local' --name 'scan_report_recipient'

# --- дополнительная демонстрация encrypt/view на целом файле ---
echo 'секретные заметки по сканированию lab5' > vault_demo.txt
ansible-vault encrypt vault_demo.txt --vault-password-file .vault_pass.txt
cat vault_demo.txt                                               # $ANSIBLE_VAULT;1.1;AES256 ...
ansible-vault view vault_demo.txt --vault-password-file .vault_pass.txt   # расшифрованный текст

# --- проверка связи ---
ansible scanners -i inventory.yml -m ping
ansible scanners -i inventory.yml -m shell -a 'which nmap || echo NOT_INSTALLED'   # NOT_INSTALLED

# --- сухой прогон ---
ansible-playbook playbook.yml -i inventory.yml --check --diff --vault-password-file .vault_pass.txt

# --- реальный запуск (-v чтобы увидеть debug с verbosity: 1) ---
ansible-playbook playbook.yml -i inventory.yml --vault-password-file .vault_pass.txt -v

# --- проверка на managed node ---
ssh runner@10.0.2.16 'cat /home/runner/scan_targets.txt'
```

В «боевом» (не автоматизированном) запуске вместо `--vault-password-file` обычно
используют интерактивный `--ask-vault-pass`; в этой лабораторной `.vault_pass.txt`
понадобился для неинтерактивного запуска через SSH-сессию и **намеренно не включён
в репозиторий** (добавлен бы в `.gitignore` в реальном проекте) — это демо-пароль
только для данного учебного стенда, а не рабочий секрет.

## Нюансы и наблюдения

- **Jinja2 vs copy**: вместо статичного `copy` файла целей (как в первом варианте
  методички) использован `ansible.builtin.template`, рендерящий список целей из
  переменной `scan_targets` — значения можно менять в одном месте (`vars/scan_vars.yml`),
  не трогая сам плейбук или шаблон.
- **Vault**: при сухом прогоне (`--check --diff`) задача `Show vault-protected
  recipient` корректно вывела **расшифрованное** значение `scan_report_recipient` —
  Vault расшифровывает переменную в памяти при каждом запуске плейбука с правильным
  паролем, вне зависимости от режима `--check`.
- **shell-модуль и `--check`**: при сухом прогоне (`--check --diff`) задача сканирования
  (`ansible.builtin.shell`) была честно пропущена (`skipping`) для обеих целей — модуль
  `shell` не поддерживает check-mode и Ansible не станет выполнять произвольную команду
  в режиме имитации. Та же особенность уже встречалась в П04 для модуля `service` —
  это общее и ожидаемое ограничение dry-run для задач с реальными побочными эффектами.
- **failed_when/register внутри loop**: `failed_when: scan_results.rc is defined and
  scan_results.rc != 0` ссылается на `scan_results` **до** того, как Ansible соберёт
  результаты всех итераций в `scan_results.results[]` — на момент вычисления
  `failed_when`/`changed_when` для конкретной итерации `register`-переменная содержит
  ещё необработанный (per-item) результат текущего шага, поэтому обращение к `.rc`
  работает корректно для каждой цели по отдельности.
- **Handlers запустились один раз**: обе задачи (`Install nmap`, `Render... template`)
  вернули `changed: true`, поэтому оба handler'а из секции `handlers:` были поставлены
  в очередь и выполнены **один раз каждый**, в конце плея, в порядке их объявления —
  ровно так, как описано в лекции.
- **Реальные, а не фиктивные результаты сканирования**: `nmap` в реальном прогоне
  подтвердил, что порт 80/tcp открыт на обоих внутренних узлах (`10.0.2.16` — nginx
  из П04, `10.0.2.15` — nginx из П02), что совпадает с фактическим состоянием стенда.
- **Почему не внешние цели**: пример методички сканирует реальные интернет-ресурсы
  университета; в этой работе выбраны исключительно узлы, принадлежащие самой
  лабораторной инфраструктуре (`10.0.2.15`/`10.0.2.16`), чтобы не выполнять сетевое
  сканирование стороннего хоста без явного согласия его владельца.

## Скриншоты

1. `screenshots/01-inventory-vars-template.png` — `inventory.yml` (YAML), `vars/scan_vars.yml`
   (список целей + зашифрованная `vault`-переменная), `templates/targets.txt.j2`.
2. `screenshots/02-ansible-vault-demo.png` — `ansible-vault encrypt_string` для переменной
   и полный цикл `encrypt`/`cat`/`view` для отдельного файла `vault_demo.txt`.
3. `screenshots/03-playbook-full.png` — полное содержимое `playbook.yml`: vars_files,
   when, template, debug (var/msg), loop+register+failed_when, handlers.
4. `screenshots/04-ping-and-precheck.png` — `ansible -m ping` через новый YAML-инвентарь;
   подтверждение, что `nmap` ещё не установлен (честный "с нуля" прогон).
5. `screenshots/05-dry-run-check-diff.png` — `--check --diff`: diff рендера шаблона,
   расшифрованное значение vault-переменной, пропуск shell-задачи сканирования.
6. `screenshots/06-real-run-full.png` — реальный запуск: установка nmap, рендер файла,
   debug var/msg, результаты сканирования по циклу, debug verbosity, оба handler'а.
7. `screenshots/07-verify-results.png` — проверка результата на самом managed node
   (`cat scan_targets.txt`) и подтверждение реально открытого порта 80 на обеих целях.
