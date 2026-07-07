# 3x-ui control

Ansible-роль для автоматического развёртывания, или быстрой миграции, VLESS VPN-сервера.

---

<p align="center">
  <img src="https://img.shields.io/badge/3X--UI-1E90FF?style=for-the-badge">
  <img src="https://img.shields.io/badge/HAProxy-6A0DAD?style=for-the-badge&logo=haproxy">
  <img src="https://img.shields.io/badge/Cloudflare-DNS--01/WARP/R2%20Bucket-F38020?style=for-the-badge&logo=cloudflare">


Устанавливает [3X-UI](https://github.com/mhsanaei/3x-ui) за HAProxy L4 балансировщиком,<br>
выпускает wildcard TLS-сертификаты через Certbot + Cloudflare DNS-01,<br>
настраивает автоматический бэкап базы данных в Cloudflare R2-бакет, устанавливает Cloudflare WARP.<br>
настраивает файервол.

---

### Перед запуском роли, уже должно быть:
- Домен управляется через Cloudflare
- Настроен рут-доступ по SSH к вашей VPS
- Создан Cloudflare R2-бакет + S3-ключи
- Сгенерирован API-токен для управляемого домена с правами `Zone:DNS:Edit`

<br>

## Архитектура

HAProxy маршрутизирует трафик по TLS SNI, с фронта `ft_main:443`, на бэкенды:

```
vless_fake_domain     Xray/VLESS        :55555
xui_panel_domain      3X-UI panel       :2053
ssh_domain            sshd              :22
sub_domain            Xray/Subscription :2096
```

<br>

После выполнения роли сервер будет открыт наружу только на порту **443**.<br>
Для подключения по SSH на клиенте рекомендуется настроить алиас:

```
Host vps
  HostName connect.mydomain.com
  Port 443
  User root
  ProxyCommand openssl s_client -connect %h:%p -servername connect.mydomain.com -quiet 2>/dev/null
```

<br>

## Установка

```bash
ansible-galaxy role install ortizmoon.xui_control
```

### Зависимости — коллекции:

```bash
ansible-galaxy collection install community.general amazon.aws community.crypto ansible.posix
```

<br>

## Переменные

### Обязательные (задаются в `group_vars`)
<br>

**`main.yml`:**

| Переменная | Описание | Пример |
|---|---|---|
| `main_domain` | Корневой домен на Cloudflare | `example.com` |
| `xui_panel_domain` | Поддомен для панели 3X-UI | `panel.example.com` |
| `ssh_domain` | Поддомен для SSH | `connect.example.com` |
| `vless_fake_domain` | Маскирующий SNI для VLESS | `vk.com` |
| `cf_email` | Email аккаунта Cloudflare | `you@example.com` |
| `r2_bucket` | Имя R2-бакета | `my-bucket` |
| `r2_endpoint` | URL эндпоинта R2 | `https://xxxx.r2.cloudflarestorage.com` |

<br>
<br>

**`creds.yml`** :

| Переменная | Описание |
|---|---|
| `cf_api_token` | Cloudflare API-токен |
| `r2_access_key` | R2 Access Key |
| `r2_secret_key` | R2 Secret Key |

<br>

### Флаги задач (по умолчанию `false`)

| Переменная | Описание |
|---|---|
| `xui_control_deploy_cert` | Выпустить wildcard TLS-сертификат |
| `xui_control_install_3xui` | Установить HAProxy + 3X-UI |
| `xui_control_restore_db` | Восстановить БД из R2 (только при миграции) |
| `xui_control_add_backup_service` | Задеплоить systemd-службу бэкапа с таймером |
| `xui_control_set_firewall` | Настроить UFW |
| `xui_control_install_warp` | Изменить DNS-записи + установить и запустить Cloudflare WARP |


<br>

## Пример плейбука

```yaml
---
- name: Deploy to new VPS
  hosts: xui_server
  become: true
  gather_facts: true
  roles:
    - role: xui_control
      vars:
        xui_control_deploy_cert: true
        xui_control_restore_db: true
        xui_control_install_3xui: true
        xui_control_add_backup_service: true
        xui_control_set_firewall: true
        xui_control_install_warp: true

```

Пример `group_vars/main.yml`:

```yaml
main_domain: example.com
xui_panel_domain: panel.example.com
ssh_domain: connect.example.com
vless_fake_domain: vk.com
cf_email: you@example.com
r2_bucket: my-bucket
r2_endpoint: https://xxxx.r2.cloudflarestorage.com

firewall_rules:
  - port: 22
    proto: tcp
    comment: ssh
  - port: 443
    proto: tcp
    comment: haproxy
```

<br>

## Бэкап и восстановление vless конфигов

Бэкап запускается ежедневно в **04:00** через systemd-таймер.<br>
Файлы хранятся в R2 с именем `x-ui-YYYY-MM-DD_HHMMSS.db`.

**Запустить бэкап вручную:**

```bash
systemctl start backup-db.service
```

<br>

**Восстановить бэкап вручную:**

```bash
/usr/local/bin/x-ui-restore-r2.sh
```



<br>

## Обновление сертификата

Certbot обновляет сертификат автоматически.<br>
После каждого обновления хук `/etc/letsencrypt/renewal-hooks/deploy/renewal-hook.sh` пересобирает `bundle.pem` и перезагружает HAProxy.

<br>

## Лицензия

MIT
