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
vagrant@host1:~$ sudo ufw allow 22
vagrant@host1:~$ sudo ufw allow 443
```
3. Установите hashicorp vault ([инструкция по ссылке](https://learn.hashicorp.com/tutorials/vault/getting-started-install?in=vault/getting-started#install-vault)).

```bash
vagrant@host1:~$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
vagrant@host1:~$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
vagrant@host1:~$ sudo apt-get update && sudo apt-get install vault
```
4. Cоздайте центр сертификации по инструкции ([ссылка](https://learn.hashicorp.com/tutorials/vault/pki-engine?in=vault/secrets-management)) и выпустите сертификат для использования его в настройке веб-сервера nginx (срок жизни сертификата - месяц).

```bash
vagrant@host1:~$ sudo apt-get install jq
vagrant@host1:~$ vault server -dev -dev-root-token-id root
vagrant@host1:~$ export VAULT_ADDR=http://127.0.0.1:8200
vagrant@host1:~$ export VAULT_TOKEN=root
vagrant@host1:~$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
vagrant@host1:~$ vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/
vagrant@host1:~$ vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/
vagrant@host1:~$ vault write -field=certificate pki/root/generate/internal \
>      common_name="example.com" \
>      ttl=87600h > CA_cert.crt
vagrant@host1:~$ vault write pki/config/urls \
>      issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
>      crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
Success! Data written to: pki/config/urls
vagrant@host1:~$ vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
vagrant@host1:~$ vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/
vagrant@host1:~$ vault write -format=json pki_int/intermediate/generate/internal \
common_>      common_name="example.com Intermediate Authority" \
>      | jq -r '.data.csr' > pki_intermediate.csr
vagrant@host1:~$ vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr \
>      format=pem_bundle ttl="43800h" \
>      | jq -r '.data.certificate' > intermediate.cert.pem
vagrant@host1:~$ vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
vagrant@host1:~$ vault write pki_int/roles/example-dot-com \
>      allowed_domains="example.com" \
>      allow_subdomains=true \
>      max_ttl="720h"
Success! Data written to: pki_int/roles/example-dot-com
vagrant@host1:~$ vault write -format=json pki_int/issue/example-dot-com common_name="test.example.com" ttl="720h" > test.example.com.crt
vagrant@host1:~$ cat test.example.com.crt | jq -r .data.certificate > test.example.pem
vagrant@host1:~$ cat test.example.com.crt | jq -r .data.issuing_ca >> test.example.pem
vagrant@host1:~$ sudo cp test.example.pem /etc/nginx/ssl/
vagrant@host1:~$ cat test.example.com.crt | jq -r .data.private_key > test.example.key
vagrant@host1:~$ sudo cp test.example.key /etc/nginx/ssl/
```
5. Установите корневой сертификат созданного центра сертификации в доверенные в хостовой системе.

![image](https://user-images.githubusercontent.com/59846765/150016766-b8fd3972-e136-4bb1-a7d8-7152997fc3a2.png)


6. Установите nginx.

```bash
vagrant@host1:~$ sudo apt install nginx
```
7. По инструкции ([ссылка](https://nginx.org/en/docs/http/configuring_https_servers.html)) настройте nginx на https, используя ранее подготовленный сертификат:
  - можно использовать стандартную стартовую страницу nginx для демонстрации работы сервера;
  - можно использовать и другой html файл, сделанный вами;

Конфиг сервера nginx:
```bash
server {
        listen     443 ssl;
        server_name         test.example.com;
        ssl_certificate     /etc/nginx/ssl/test.example.pem;
        ssl_certificate_key /etc/nginx/ssl/test.example.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
}
```
Проверяем конфиг и рестарт:
```bash
vagrant@host1:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
vagrant@host1:~$ sudo systemctl restart nginx
```
8. Откройте в браузере на хосте https адрес страницы, которую обслуживает сервер nginx.

В файле **hosts** 127.0.0.1	test.example.com

![image](https://user-images.githubusercontent.com/59846765/150017444-23ec76e0-7fe4-4386-a382-081d076414ce.png)

9. Создайте скрипт, который будет генерировать новый сертификат в vault:
  - генерируем новый сертификат так, чтобы не переписывать конфиг nginx;
  - перезапускаем nginx для применения нового сертификата.
10. Поместите скрипт в crontab, чтобы сертификат обновлялся какого-то числа каждого месяца в удобное для вас время.

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
