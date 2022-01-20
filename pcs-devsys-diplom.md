# Курсовая работа по итогам модуля "DevOps и системное администрирование"

Курсовая работа необходима для проверки практических навыков, полученных в ходе прохождения курса "DevOps и системное администрирование".

Мы создадим и настроим виртуальное рабочее место. Позже вы сможете использовать эту систему для выполнения домашних заданий по курсу

## Задание

1. Создайте виртуальную машину Linux. 

Создал через Vagrant

Конфиг:

```bash
Vagrant.configure("2") do |config|
		config.vm.box = "bento/ubuntu-20.04"
		config.vm.hostname = 'host1'
		config.vm.network :forwarded_port, host: 4555, guest: 80
		config.vm.network :forwarded_port, host: 443, guest: 443
	#end
end
```
2. Установите ufw и разрешите к этой машине сессии на порты 22 и 443, при этом трафик на интерфейсе localhost (lo) должен ходить свободно на все порты.

ufw установлен в Ubuntu по умолчанию
```bash
 sudo ufw allow 22
 sudo ufw allow 443
 sudo ufw enable
```
3. Установите hashicorp vault ([инструкция по ссылке](https://learn.hashicorp.com/tutorials/vault/getting-started-install?in=vault/getting-started#install-vault)).

```bash
 curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
 sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
 sudo apt-get update && sudo apt-get install vault
 sudo apt-get install jq
```

Запускаем сервер в отельной сессии
```bash
vault server -dev -dev-root-token-id root
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.5
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.9.2
             Version Sha: f4c6d873e2767c0d6853b5d9ffc77b0d297bfbdf

==> Vault server started! Log data will stream in below:
```
Включаем аудит для troubleshooting
```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root
vault audit enable file file_path=/var/log/vault_audit.log
Success! Enabled the file audit device at: file/
vault audit enable -path="file_raw" file  log_raw=true file_path=/var/log/vault_audit_raw.log
Success! Enabled the file audit device at: file_raw/
```
4. Cоздайте центр сертификации по инструкции ([ссылка](https://learn.hashicorp.com/tutorials/vault/pki-engine?in=vault/secrets-management)) и выпустите сертификат для использования его в настройке веб-сервера nginx (срок жизни сертификата - месяц).

Создаем корневой и промежуточный сертификаты
```bash
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
vault write -field=certificate pki/root/generate/internal \
common_name="example.com" \
ttl=87600h > CA_cert.crt
vault write pki/config/urls \
issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int
vault write -format=json pki_int/intermediate/generate/internal \
     common_name="example.com Intermediate Authority" \
     | jq -r '.data.csr' > pki_intermediate.csr

vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr \
     format=pem_bundle ttl="43800h" \
     | jq -r '.data.certificate' > intermediate.cert.pem
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```
Создаём роль
```bash
vault write pki_int/roles/example-dot-com \
     allowed_domains="example.com" \
     allow_subdomains=true \
     max_ttl="720h"
```
Создаём файл с политикой, c именем **pki_int.hcl**
```
path "pki_int/issue/*" {
      capabilities = ["create", "update"]
    }

    path "pki_int/certs" {
      capabilities = ["list"]
    }

    path "pki_int/revoke" {
      capabilities = ["create", "update"]
    }

    path "pki_int/tidy" {
      capabilities = ["create", "update"]
    }

    path "pki/cert/ca" {
      capabilities = ["read"]
    }

    path "auth/token/renew" {
      capabilities = ["update"]
    }

    path "auth/token/renew-self" {
      capabilities = ["update"]
    }
```
Создаём политику
```bash
vault policy write pki_int pki_int.hcl
```
Создаём пользователя под которым будем создвать сертификаты. Лучше иметь отдельного пользователя в целях безопасности.
```bash
export VAULT_USER="user_cer"
export VAULT_PASSWORD="secret"
vault auth enable userpass
vault write auth/userpass/users/${VAULT_USER} \
    password=${VAULT_PASSWORD} \
    token_policies="pki_int"
```
Входим под сервисным пользователем
```bash
export VAULT_USER="user_cer"
export VAULT_PASSWORD="secret"

vault login -format=json -method=userpass \
    username=${VAULT_USER} \
    password=${VAULT_PASSWORD} | jq -r .auth.client_token > user.token

#Сохраняем токен в переменную, чтобы проходить аутентификацию
export VAULT_TOKEN=`cat user.token`
```
Генерируем сертификат для сайта **test.example.com** и сохраняем его для конфига nginx
```bash
vault write -format=json pki_int/issue/example-dot-com common_name="test.example.com" ttl="720h" > test.example.com.crt
cat test.example.com.crt | jq -r .data.certificate > /usr/local/bin/test.example.pem
cat test.example.com.crt | jq -r .data.issuing_ca >> /usr/local/bin/test.example.pem
cat test.example.com.crt | jq -r .data.private_key > /usr/local/bin/test.example.key
```
5. Установите корневой сертификат созданного центра сертификации в доверенные в хостовой системе.

![image](https://user-images.githubusercontent.com/59846765/150016766-b8fd3972-e136-4bb1-a7d8-7152997fc3a2.png)


6. Установите nginx.

```bash
 sudo apt install nginx
```
7. По инструкции ([ссылка](https://nginx.org/en/docs/http/configuring_https_servers.html)) настройте nginx на https, используя ранее подготовленный сертификат:
  - можно использовать стандартную стартовую страницу nginx для демонстрации работы сервера;
  - можно использовать и другой html файл, сделанный вами;

Правим конфиг сервера nginx, добавялем раздел:
```bash
server {
        listen     443 ssl;
        server_name         test.example.com;
        ssl_certificate     /usr/local/bin/test.example.pem;
        ssl_certificate_key /usr/local/bin/test.example.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
}
```
Проверяем конфиг и рестарт:
```bash
sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
sudo systemctl restart nginx
```
8. Откройте в браузере на хосте https адрес страницы, которую обслуживает сервер nginx.

В файле **hosts** 127.0.0.1	test.example.com

![image](https://user-images.githubusercontent.com/59846765/150017444-23ec76e0-7fe4-4386-a382-081d076414ce.png)

9. Создайте скрипт, который будет генерировать новый сертификат в vault:
  - генерируем новый сертификат так, чтобы не переписывать конфиг nginx;
  - перезапускаем nginx для применения нового сертификата.
```bash
#!/bin/sh
set -o xtrace
export VAULT_ADDR=http://localhost:8200
export VAULT_USER="user_cer"
export VAULT_PASSWORD="secret"

vault login -format=json -method=userpass \
    username=${VAULT_USER} \
    password=${VAULT_PASSWORD} | jq -r .auth.client_token > user.token

export VAULT_TOKEN=`cat user.token`

vault write -format=json pki_int/issue/example-dot-com common_name="test.example.com" ttl="720h" > test.example.com.crt
cat test.example.com.crt | jq -r .data.certificate > /usr/local/bin/test.example.pem
cat test.example.com.crt | jq -r .data.issuing_ca >> /usr/local/bin/test.example.pem
cat test.example.com.crt | jq -r .data.private_key > /usr/local/bin/test.example.key
systemctl restart nginx
```
10. Поместите скрипт в crontab, чтобы сертификат обновлялся какого-то числа каждого месяца в удобное для вас время.

Чтобы убедиться, что всё работает, меняю срок в сертификате на другое значение. Сделал 5 минут для проверки работы крона. 
```bash
*/5 * * * * /usr/local/bin/generate_cer.sh
```
Крон отрабатывает, новые сертификаты генерируются. Чтобы увидеть новый сертификат на хосте, нужно очистить кэш и перезапустить браузер. Так же можно увидеть, что серийный номер сертификата другой
```
Jan 20 20:45:01 vagrant CRON[3635]: (root) CMD (/usr/local/bin/generate_cer.sh)
Jan 20 20:45:01 vagrant CRON[3634]: (CRON) info (No MTA installed, discarding output)
Jan 20 20:50:01 vagrant CRON[3667]: (root) CMD (/usr/local/bin/generate_cer.sh)
Jan 20 20:50:02 vagrant CRON[3666]: (CRON) info (No MTA installed, discarding output)
```
## Результат

Результатом курсовой работы должны быть снимки экрана или текст:

- Процесс установки и настройки ufw
- Процесс установки и выпуска сертификата с помощью hashicorp vault
- Процесс установки и настройки сервера nginx
- Страница сервера nginx в браузере хоста не содержит предупреждений 
- Скрипт генерации нового сертификата работает (сертификат сервера ngnix должен быть "зеленым")
- Crontab работает (выберите число и время так, чтобы показать что crontab запускается и делает что надо)

## Как сдавать курсовую работу

Курсовую работу выполните в файле readme.md в github репозитории. В личном кабинете отправьте на проверку ссылку на .md-файл в вашем репозитории.

Также вы можете выполнить задание в [Google Docs](https://docs.google.com/document/u/0/?tgif=d) и отправить в личном кабинете на проверку ссылку на ваш документ.
Если необходимо прикрепить дополнительные ссылки, просто добавьте их в свой Google Docs.

Перед тем как выслать ссылку, убедитесь, что ее содержимое не является приватным (открыто на комментирование всем, у кого есть ссылка), иначе преподаватель не сможет проверить работу. 
Ссылка на инструкцию [Как предоставить доступ к файлам и папкам на Google Диске](https://support.google.com/docs/answer/2494822?hl=ru&co=GENIE.Platform%3DDesktop).
