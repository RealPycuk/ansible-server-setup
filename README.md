# Ansible: настройка сервера с нуля

[![CI](https://github.com/RealPycuk/ansible-server-setup/actions/workflows/ci.yml/badge.svg)](https://github.com/RealPycuk/ansible-server-setup/actions/workflows/ci.yml)

Плейбук приводит свежий Ubuntu-сервер к рабочему состоянию: базовые пакеты и swap, защита SSH, fail2ban, файрвол, Docker и стек мониторинга. Он повторяет ручную настройку двух моих серверов, на которых работает Telegram-бот, только теперь это код, который можно прогнать заново и проверить в CI.

## Роли

| Роль | Что делает |
|---|---|
| `base` | пакеты, часовой пояс, автоматические обновления безопасности, swap-файл, `vm.swappiness` |
| `ssh_hardening` | `MaxAuthTries 3`, `LoginGraceTime 30`; вход по паролю отключается, **только если в журнале уже есть успешный вход по ключу**, — чтобы не потерять доступ к серверу |
| `fail2ban` | jail для sshd с растущим баном (1 час → до недели), второй сервер в whitelist |
| `firewall` | ufw: SSH разрешается до включения файрвола, публичные порты, порты только для второго сервера, по умолчанию deny |
| `docker` | Docker из официального репозитория, ротация логов контейнеров; существующий `daemon.json` дополняется, а не перезаписывается |
| `monitoring` | стек из [monitoring-stack](https://github.com/RealPycuk/monitoring-stack) по закреплённому тегу, секреты из ansible-vault, цели blackbox из переменных, ожидание готовности Prometheus |

Все конфиги sshd проверяются `sshd -t` до применения, перезапуски — через handlers, только если что-то изменилось.

## Запуск

```bash
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml              # адреса серверов
cp group_vars/all/vault.example.yml group_vars/all/vault.yml    # пароль Grafana, токен и chat_id для алертов
ansible-vault encrypt group_vars/all/vault.yml

ansible-playbook site.yml --check --diff --ask-vault-pass       # посмотреть, что изменится
ansible-playbook site.yml --ask-vault-pass                      # применить
ansible-playbook site.yml --tags firewall --ask-vault-pass      # только одну роль
```

## CI

GitHub Actions на каждый push:

1. `ansible-lint` и `--syntax-check`;
2. **реальный прогон на чистой VM** — плейбук применяется к самому раннеру GitHub Actions (`inventory/ci.yml`, `ansible_connection: local`);
3. **проверка идемпотентности** — второй прогон должен завершиться с `changed=0`;
4. проверка результата: ufw активен и пускает второй сервер только на свой порт, fail2ban следит за sshd, `sshd -T` показывает запрет пароля, мониторинг поднят нужной версии и отвечает, у файлов с секретами правильные права.
