# ansible-admins

Управление учётными записями администраторов на флоте VPS-нод и базовое
hardening SSH-конфигурации. Один источник правды для команды, идемпотентные
прогоны, чистый офбординг.

## Что делает

Два playbook'а:

- **add_admins.yml** — создаёт/удаляет админов на всех нодах согласно списку
  `inventory/group_vars/all/admins.yml`. Раскладывает SSH-ключи, выдаёт sudo
  через `/etc/sudoers.d/10-<user>`, фиксит мелочи (`/etc/hosts`, terminfo).

- **harden_ssh.yml** — закрывает root по SSH, отключает парольный логин,
  нейтрализует cloud-init оверрайды. Применяется ПОСЛЕ того, как админы
  раскатаны и проверены.

## Структура репозитория

\```
ansible-admins/
├── ansible.cfg                          # дефолтные настройки ansible
├── inventory/
│   ├── hosts.yml                        # список нод
│   └── group_vars/
│       ├── all/
│       │   └── admins.yml               # источник правды о команде
│       └── vpn_nodes/
│           └── main.yml                 # SSH-настройки для группы
├── files/
│   └── admins/
│       ├── admin_oleg.pub               # публичные ключи админов
│       └── admin_petya.pub              # ВАЖНО: только .pub, никогда приватные
├── playbooks/
│   ├── add_admins.yml
│   └── harden_ssh.yml
├── .gitignore
└── README.md
Требования

Ansible 2.16+ на машине, с которой запускаются playbook'и
Коллекция ansible.posix (модуль authorized_key)
Python 3 на целевых нодах (по дефолту есть на Debian/Ubuntu)
SSH-доступ под root на ноды (по ключу или паролю — для bootstrap)
На контроллере: sshpass если используется --ask-pass

Установка
```bash
git clone <repo-url> ~/.ansible/ansible-admins
cd ~/.ansible/ansible-admins
зависимости ansible
ansible-galaxy collection install ansible.posix
для парольного bootstrap (опционально)
sudo pacman -S sshpass    # CachyOS / Arch
sudo apt install sshpass    # Debian / Ubuntu
```
Концепция работы
Модель доступа

Ansible ходит на ноды под root (по ключу или паролю при bootstrap'е)
Playbook создаёт админских юзеров для членов команды
Люди ходят под своими юзерами (admin_<name>) с личных ключей
Все админы получают sudo NOPASSWD через отдельный sudoers dropin
Парольный вход залочен (password_lock: true) — только по SSH-ключу

Файл admins.yml — единый источник правды
```yaml
admins:

name: admin_oleg
state: present       # present = создать, absent = удалить
role: superadmin
comment: "Evgeniy / founder"
name: admin_petya
state: present
role: superadmin
comment: "Petya / backend"
name: admin_ivan
state: absent        # уволился — playbook удалит со всех нод
comment: "Left team 2026-03-01"
```

Жизненный цикл админа

Человек у себя генерит ключ: ssh-keygen -t ed25519 -C "user@machine"
Шлёт публичную часть (~/.ssh/id_ed25519.pub) ответственному
Ответственный кладёт в files/admins/admin_<name>.pub
Добавляет запись в inventory/group_vars/all/admins.yml
Коммит → MR → merge → прогон playbook'а
Человек ходит на ноды как admin_<name>

При увольнении: меняем state: present → state: absent, прогоняем playbook
с тегом offboard — юзер удаляется со всех нод вместе с домашкой и sudoers.
Использование
Добавить нового члена команды
```bash
cd ~/.ansible/ansible-admins
положить присланный публичный ключ
cp ~/Downloads/petya.pub files/admins/admin_petya.pub
добавить запись в admins.yml
$EDITOR inventory/group_vars/all/admins.yml
проверка синтаксиса
ansible-playbook playbooks/add_admins.yml --syntax-check
dry-run на нескольких нодах
ansible-playbook playbooks/add_admins.yml --check --diff --limit 'vpn_nodes[0:3]'
боевой прогон
ansible-playbook playbooks/add_admins.yml
коммит изменений
git add files/admins/admin_petya.pub inventory/group_vars/all/admins.yml
git commit -m "Onboard admin_petya"
git push
```
Убрать члена команды
```bash
$EDITOR inventory/group_vars/all/admins.yml
заменить state: present → state: absent
ansible-playbook playbooks/add_admins.yml --tags offboard
git commit -am "Offboard admin_ivan"
git push
```
.pub файл можно удалить из репозитория или оставить — state: absent его
больше не использует.
Раскатить на новую ноду
Свежая VPS с парольным root-доступом:
```bash
добавить в inventory/hosts.yml
$EDITOR inventory/hosts.yml
первый прогон с паролем (Ansible разложит ключи)
ansible-playbook playbooks/add_admins.yml --limit node-new-01 --ask-pass
проверить что admin_oleg ходит
ssh -i ~/.ssh/prod_ed25519 admin_oleg@<IP> "sudo -n whoami"
закрыть root и пароли (см. ниже)
ansible-playbook playbooks/harden_ssh.yml --limit node-new-01
```
Если у провайдера root-доступ только по ключу из коробки — --ask-pass не
нужен, всё проходит сразу.
Hardening SSH
Применяется ПОСЛЕ того, как add_admins.yml отработал успешно и проверена
работа admin_oleg под ключом с sudo.
```bash
обязательно: открыть safety-сессию в отдельном окне
ssh -i ~/.ssh/prod_ed25519 admin_oleg@<IP>
не закрывать до конца проверки!
в основном окне:
ansible-playbook playbooks/harden_ssh.yml --check --diff --limit node-test
ansible-playbook playbooks/harden_ssh.yml --limit node-test
```
Проверки после прогона из третьего окна:
```bash
admin_oleg по ключу — должен зайти
ssh -i ~/.ssh/prod_ed25519 admin_oleg@<IP> "sudo -n whoami"
root — должен отказать
ssh root@<IP>
admin_oleg по паролю — должен отказать
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin_oleg@<IP>
```
Прогон на флот
Поэтапно, через --limit:
```bash
dry-run всегда первым делом
ansible-playbook playbooks/add_admins.yml --check --diff --limit 'vpn_nodes[0:5]'
первая пятёрка
ansible-playbook playbooks/add_admins.yml --limit 'vpn_nodes[0:5]'
проверка что ходит
for ip in $(ansible vpn_nodes -m debug -a 'msg={{ ansible_host }}' 
--limit 'vpn_nodes[0:5]' | grep msg | awk -F'"' '{print $4}'); do
echo "=== $ip ==="
ssh -i ~/.ssh/prod_ed25519 admin_oleg@$ip "sudo -n whoami"
done
на остальные
ansible-playbook playbooks/add_admins.yml
harden — той же логикой
ansible-playbook playbooks/harden_ssh.yml --limit 'vpn_nodes[0:5]'
ansible-playbook playbooks/harden_ssh.yml
```
Что делает каждый playbook
add_admins.yml
ТаскаЧто делаетValidate all pubkeys exist on controllerПроверяет, что для каждого state: present админа есть .pub файлUpdate apt cacheapt-get update если кеш старше часаEnsure hostname resolves locallyДобавляет в /etc/hosts запись 127.0.1.1 <fqdn> <hostname> (убирает sudo warning'и)Ensure sudo is installedСтавит пакет sudo (на минимальных VPS его нет)Set safe default TERMКладёт скрипт-фолбэк в /etc/profile.d/ для терминалов вроде kitty/alacrittyCreate admin usersuseradd с залоченным паролем, shell /bin/bash, домашка создаётсяHarden home permissionschmod 0700 на /home/<user>Install authorized_keyКладёт публичный ключ из files/admins/<user>.pub, exclusive: trueGrant sudo via dropinСоздаёт /etc/sudoers.d/10-<user> с NOPASSWD ALL, валидируется через visudo -cfRemove sudoers dropinДля state: absent — удаляет sudoers файлRemove departed admin usersДля state: absent — userdel -r -f (с домашкой)Verify sudo worksПроверяет что sudo -u <user> -n true возвращает 0
harden_ssh.yml
ТаскаЧто делаетVerify admin_oleg can sudoPre-check: если admin_oleg не работает — playbook падает, root не закрываетсяVerify authorized_keys existsPre-check: проверяет что ключи реально лежат у admin_olegBackup sshd_configКопия в /etc/ssh/sshd_config.bak.YYYY-MM-DD (один раз в день)Set PermitRootLogin noЗакрывает root SSHSet PasswordAuthentication noЗакрывает парольный логинSet KbdInteractiveAuthentication noЗакрывает PAM challenge-responseFind sshd_config.d override filesИщет файлы в /etc/ssh/sshd_config.d/Neutralize cloud-init overridesКомментирует PasswordAuthentication yes в файлах провайдера (Hetzner, DO кладут такие)Handler: restart sshdПрименяет конфиг через systemctl restart ssh
Откат
Если harden_ssh.yml сломал доступ
В safety-сессии (которая должна быть открыта во время прогона):
```bash
sudo ls /etc/ssh/sshd_config.bak.*
sudo cp /etc/ssh/sshd_config.bak.YYYY-MM-DD /etc/ssh/sshd_config
sudo sshd -t   # проверить синтаксис
sudo systemctl restart ssh
```
Если safety-сессия не открыта и доступа нет
Вход через web-консоль провайдера (KVM / recovery mode):
```bash
mount -o remount,rw /
cp /etc/ssh/sshd_config.bak.YYYY-MM-DD /etc/ssh/sshd_config
systemctl restart ssh
```
Если admin случайно удалён через absent
Ставишь обратно state: present, прогоняешь playbook — юзер пересоздастся.
Но домашка не восстановится (была удалена userdel -r). Дотфайлы и
прочее придётся восстанавливать самому.
Безопасность
Что не должно попадать в репозиторий

Приватные SSH-ключи (id_*, *.pem) — только публичные .pub
Файлы *.retry — артефакты прогонов
.vault_pass, .vault_password — пароли от Ansible Vault
.DS_Store, .idea/, .vscode/ — IDE-мусор

Всё это покрыто .gitignore. Перед первым коммитом проверь:
```bash
git status
git diff --cached
```
Хранение публичных ключей в git
Это нормально. Публичные ключи по определению не секретны — их назначение
проверять подпись, не подписывать. Любой может видеть публичный ключ — без
приватного он бесполезен.
Ротация ключей
Если ключ скомпрометирован (украли ноут с приватником):

Сгенерировать новую пару у пострадавшего
Заменить files/admins/admin_<name>.pub на новый
Прогнать add_admins.yml — exclusive: true затрёт старый ключ новым
Старая пара теперь бесполезна

Security checklist перед прокаткой на прод

 Тестовая нода прошла полный цикл: создание → проверка → удаление
 Под admin_oleg sudo работает: sudo -n whoami → root
 Идемпотентность: повторный прогон playbook'а возвращает changed=0
 У провайдера известна процедура recovery / KVM-доступа
 Safety-сессия открывается в новом окне ПЕРЕД harden_ssh
 Прогон по принципу 1 нода → 5 нод → весь флот
 --check --diff перед каждой группой нод
 Бэкапы sshd_config создаются автоматически в первой таске harden
 Изменения в admins.yml идут через git с осмысленными коммитами

Тэги
```bash
ansible-playbook playbooks/add_admins.yml --tags bootstrap     # apt+sudo+hosts
ansible-playbook playbooks/add_admins.yml --tags user          # создание юзеров
ansible-playbook playbooks/add_admins.yml --tags ssh_key       # только ключи
ansible-playbook playbooks/add_admins.yml --tags sudo          # только sudoers
ansible-playbook playbooks/add_admins.yml --tags offboard      # только удаление
ansible-playbook playbooks/add_admins.yml --tags verify        # только проверка
```
Для harden_ssh.yml тэги не разделены — играется целиком.
Inventory
Хосты определены в inventory/hosts.yml:
```yaml
all:
children:
vpn_nodes:
children:
opengater_nodes:
hosts:
node-de-01:
ansible_host: 95.142.42.195
node-nl-01:
ansible_host: 140.99.223.126
unfence_nodes:
hosts: {}
infra:
hosts:
panel:
ansible_host: panel.duckpunch.cc
```
Соединение настроено в inventory/group_vars/vpn_nodes/main.yml:
```yaml
ansible_user: root
ansible_ssh_private_key_file: ~/.ssh/prod_ed25519
ansible_python_interpreter: /usr/bin/python3
```
Известные warning'и
Host using discovered Python interpreter — Ansible сам определил Python
на ноде. Безопасно игнорировать или явно указать
ansible_python_interpreter: /usr/bin/python3 (уже в group_vars).
INJECT_FACTS_AS_VARS deprecated — старый способ обращения к фактам
(ansible_pkg_mgr vs ansible_facts['pkg_mgr']). На совместимости не
сказывается, исправится при апгрейде до Ansible 2.24.
Importing 'to_native' from ansible.module_utils._text — внутри коллекции
ansible.posix. Лечится: ansible-galaxy collection install ansible.posix --upgrade.
Troubleshooting
Permission denied (publickey) при подключении к ноде
```bash
ssh -v -i ~/.ssh/prod_ed25519 -o IdentitiesOnly=yes root@<IP> 2>&1 | 
grep -E "Offering|Server accepts|denied"
```
Проверить:

Права на ключ: chmod 600 ~/.ssh/prod_ed25519
Права authorized_keys на ноде: 0600
Права ~/.ssh на ноде: 0700
Что в ~/.ssh/config нет конфликтующих настроек

No package matching 'sudo' is available
Кеш apt пустой. add_admins.yml сам делает apt update в начале — но если
прогоняется по тэгу мимо bootstrap, можно вручную:
```bash
ansible vpn_nodes -m apt -a "update_cache=yes" --limit <node> --become
```
sudo: unable to resolve host <name>
Не критично, sudo всё равно работает. Решается таской
Ensure hostname resolves locally. Если warning остался — проверить
содержимое /etc/hosts на ноде.
Failed when, что pubkey не существует
Файл в files/admins/admin_<name>.pub отсутствует или путь некорректный.
Проверить:
```bash
ls -la files/admins/
ssh-keygen -lf files/admins/admin_<name>.pub
```
Lineinfile зацепился за комментарий
Регулярка слишком широкая. Должна быть ^[#\s]*KEY\s+(yes|no), а не
^#?\s*KEY — последняя ловит любые упоминания в текстах комментариев.
Дальнейшее развитие
Что стоит добавить позже:

Role-based sudo — разные шаблоны sudoers для operator/devops/superadmin
по полю role в admins.yml
fail2ban в bootstrap-таски для защиты от brute force
Очистка /root/.ssh/authorized_keys — после harden оставлять там
только провайдерский ключ, личные убирать
Централизованное логирование sudo — log_input/log_output в sudoers,
отгрузка в Loki через rsyslog
Semaphore / Tower для запуска не с локальной машины, с историей
прогонов и аудитом изменений
Vault для секретов — токены Telegram-уведомлений, API-ключи если
понадобятся в playbook'ах
Уведомление в Telegram на завершение прогона из CI/Semaphore

Авторы и история
Repository maintained by <team>. Issues, PRs welcome.
License: MIT (либо на твоё усмотрение)
```

## Куда положить

```bash
cd ~/.ansible/ansible-admins
$EDITOR README.md
# вставить содержимое выше

git add README.md
git commit -m "Add comprehensive README"
git push
```

Если потом захочешь разнести по отдельным файлам — для команды это часто удобнее, потому что проще найти нужное:
docs/
├── ONBOARDING.md       # как добавить нового члена команды
├── OFFBOARDING.md      # как уволить
├── BOOTSTRAP.md        # как раскатать на новую ноду
├── HARDENING.md        # подробно про harden_ssh + откат
├── TROUBLESHOOTING.md  # частые ошибки
└── SECURITY.md         # модель угроз, политики
