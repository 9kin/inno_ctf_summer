# Первый запуск

Запускаем машину. Видим корпоративную шиндовс 7. Лично я всегда чекаю `utilman.exe` на предмет подмены. Ожидаемо, вместо экранной клавиатуры открывается консоль от администратора. Или это физический бэкдор, оставленный хакером на всякий случай, или у взломщика был физический доступ к машине.

Заходим под нашим пользователем `investigator` (пароль `12345`)

Сразу открывается и закрывается `powershell`. Ну такое.

Запускаем `powershell`, прописываем `net users`:

```
\> net users

Учетные записи пользователей для \\ROGERS

-------------------------------------------------------------------------------
Admin                    backup                   bob
investigator             joker                    Администратор
Гость
Команда выполнена успешно.
```

Видим подозрительного юзера `joker`, запомним его.

# Ищем логи

Запускаем `просмотр событий`, выбираем `журналы приложений и служб` > `Windows`.

Наверняка на машине установлено какое-то ПО для сбора логов. 

Спустя пару минут поиска находим `Sysmon`, открываем.

Видим тонну логов.

Попробуем найти момент создания юзера `joker`:

Видим последовательность команд от юзера `система`, начавшуюся `31.01.2017` в `11:02:21`:

```
\> whoami
\> user joker joker /add
\> net  localgroup Администраторы joker /add
```

Очевидно, что злоумышленник получил права администратора и создал себе юзера `joker`, тоже имеющего права админа.

Листаем логи, ищем момент первого проникновения.

# Момент первого проникновения

Прямо перед первыми подозрительными командами был запуск `WinMail.exe` от юзера `backup`. Можем предположить что он ~~повелся на кликбейт~~ стал жертвой социальной инженерии.

Находим предположительно первую подозрительную команду, видимо хацкер вошел в систему под юзером `backup` и собирает информацию о системе:
```
\> wmic  qfe
Caption                                     CSName  Description  FixComments  HotFixID   InstallDate  InstalledBy                 InstalledOn  Name  ServicePackInEffect  Status
http://support.microsoft.com/?kbid=2534111  ROGERS  Hotfix                    KB2534111                 12/5/2016
http://support.microsoft.com/?kbid=2999226  ROGERS  Update                    KB2999226               ROGERS\investigator                8/17/2020
http://support.microsoft.com                ROGERS  Update                    KB958488                ROGERS\Admin                       1/30/2017
http://support.microsoft.com/?kbid=976902   ROGERS  Update                    KB976902                ROGERS\Администратор               11/21/2010
```

Потом запускает создает и запускает некий `local.exe` (это `meterpreter`), перемещает его в папку с драйверами на принтер (`C:\Windows\System32\spool\drivers\color\local.exe`) и прописывает в реестре в автозапуск:
```
\> reg  add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run" /v easy /t REG_SZ /d "C:\Windows\System32\spool\drivers\color\local.exe"
```

# Повышение привилегий

В той же папке с дровами на принтер хацкер создает файл эксплойта [MS16-032](https://gist.github.com/benichmt1/af52401c7f2d6984dea6ba60b44aa1aa)

`C:\Windows\System32\spool\drivers\color\Invoke-MS16-032.ps1`

И запускает
`\> powershell  -exec bypass`

Что дает ему консоль от имени админа.

Дальше идет блок команд, найденный изначально:

```
\> whoami
\> user joker joker /add
\> net  localgroup Администраторы joker /add
```

# Закрепление в системе

Залогинившись под джокером, хаkер отключает смену пароля на системе:

`\> reg  ADD HKLM\SYSTEM\CurrentControlSet\services\Netlogon\Parameters /v DisablePasswordChange /t REG_DWORD /d 1 /f`

И запускает [mimikatz.exe](https://github.com/gentilkiwi/mimikatz):

`\> mimikatz.exe`

Читает его логи:

`\> NOTEPAD.EXE" C:\Windows\Tasks\mimikatz.log`

И получает все пароли в этой системе.

Видимо что-то пошло не так, еще 1 запуск:

`\> mimikatz.exe  "privilege::debug" "sekurlsa::logonPasswords" "exit"`

Дальше он создает файл [peloader_local.exe](https://github.com/JeremyWildsmith/PELoader) - тулза для внедрения вредоносов в другие приложения.

###  Попробуем залогиниться под джокером.

Видим на его рабочем столе `Google Chrome` и `корзину`.

Заходим в корзину, видим там два `хрома`, `7z установщик` и `peloader_local.exe` - делаем вывод что была произведена модификация хрома для получения reverse shell'а. Это подтверждается тем, что хром (с рабочего стола) не работает.

# Итог

Юзер `backup` скачал малварь, которую ему прислали по почте, хаkер получил обратный шелл, поднял себе привилегии при помощи эксплойта MS16-032, оставил несколько бэкдоров, в том числе и физическийю