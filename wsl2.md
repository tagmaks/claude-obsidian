# Ubuntu 26.04 в WSL 2: настройка с ZRAM

Это подробная инструкция для настройки существующего дистрибутива Ubuntu в WSL 2. Команды проверены на Ubuntu 26.04.1 LTS и WSL 2.7.8. Если у вас другая версия Ubuntu, сначала проверьте наличие пакетов через `apt-cache policy`.

Профиль ниже соответствует текущему выбору: **2 логических процессора и 2 ГБ памяти**. Это глобальный лимит WSL 2: он действует на все дистрибутивы WSL 2, а не только на Ubuntu. Измените значения, если этой памяти недостаточно для ваших сборок.

`swap=0` отключает дисковый swap WSL. Не устанавливайте его до тех пор, пока ZRAM не запущен и не проверен: ZRAM становится единственным swap в Linux.

Эта инструкция настраивает Linux-среду, но не создаёт базу знаний. Репозиторий `claude-obsidian` — исходный код продукта, а не пользовательский vault. Для нового vault используйте штатный `init` с предварительным просмотром, а для существующего — `adopt`; подробности — в [руководстве по установке](docs/install-guide.md).

## 1. Проверить WSL и Ubuntu

Выполните в PowerShell:

```powershell
wsl --version
wsl --list --verbose
```

Убедитесь, что Ubuntu отображается в списке и использует версию `2`. Если имя дистрибутива отличается, подставляйте точное имя из столбца `NAME`.

Если Ubuntu установлена как WSL 1, преобразуйте её:

```powershell
wsl --set-version Ubuntu-26.04 2
```

После удаления ненужных дистрибутивов можно сделать Ubuntu дистрибутивом по умолчанию:

```powershell
wsl --set-default Ubuntu-26.04
```

Запустите Ubuntu:

```powershell
wsl --distribution Ubuntu-26.04
```

В терминале Ubuntu проверьте версию и Python:

```bash
cat /etc/os-release
python3 --version
```

## 2. Обновить Ubuntu и установить зависимости

Сначала оставьте стандартный WSL swap включённым. Он остаётся запасным вариантом, пока ZRAM не установлен и не проверен.

Выполните внутри Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
  build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev \
  libsqlite3-dev wget curl llvm libncurses-dev xz-utils tk-dev \
  libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev git \
  python3-pip python3-venv
```

Для Ubuntu 26.04 используйте `libncurses-dev`: пакет `libncursesw5-dev` из старой команды может отсутствовать. Не устанавливайте pip-пакеты в системный Python; для них используйте virtualenv.

Проверьте, что пакет ZRAM доступен, и установите его:

```bash
apt-cache policy zram-config
sudo apt install -y zram-config
```

Если у `apt-cache policy` нет кандидата, не отключайте WSL swap. Сначала проверьте актуальную инструкцию для своей версии Ubuntu.

## 3. Запустить и проверить ZRAM

Ubuntu должна работать с systemd. Если `systemctl` недоступен, проверьте `/etc/wsl.conf`: параметр `systemd=true` должен находиться в секции `[boot]`. Не добавляйте повторную секцию `[boot]`; после изменения `/etc/wsl.conf` перезапустите WSL из PowerShell.

Включите службу ZRAM:

```bash
sudo systemctl enable --now zram-config.service
systemctl status --no-pager zram-config.service
swapon --show
zramctl
```

Для `zram-config` из Ubuntu 26.04 размер устройства рассчитывается как половина доступной Linux памяти. Фактический размер зависит от памяти, которую WSL видит при старте; ориентируйтесь на `zramctl`, а не на заранее заданное число.

Перед продолжением убедитесь, что служба активна и в `swapon --show` появился `/dev/zram0`. Пока ZRAM не активна, не ставьте `swap=0`.

## 4. Настроить swappiness без дублирования строк

Постоянные настройки sysctl храните в отдельном файле, а не дописывайте новую строку при каждом запуске в `/etc/sysctl.conf`.

Если файл уже существует, сначала прочитайте и сохраните его содержимое. Затем откройте его редактором:

```bash
sudoedit /etc/sysctl.d/99-wsl-zram.conf
```

Добавьте или оставьте в нём одну настройку:

```ini
vm.swappiness=10
```

Примените значение в текущем сеансе и проверьте его:

```bash
sudo sysctl -w vm.swappiness=10
sysctl vm.swappiness
```

После следующего запуска WSL значение должно снова быть `10`.

## 5. Установить NVM и Node.js LTS

Команда `curl -o- https://githubusercontent.com | bash` не указывает на установщик NVM. Используйте официальный репозиторий `nvm-sh/nvm` и не запускайте установщик через `sudo`: он устанавливает NVM в домашний каталог пользователя и добавляет инициализацию в `~/.bashrc`.

Пример закрепляет проверенную версию NVM `v0.40.8`. Скачайте установщик, проверьте его синтаксис и просмотрите файл перед запуском:

```bash
NVM_VERSION=v0.40.8
NVM_INSTALLER="$HOME/nvm-install-${NVM_VERSION}.sh"

curl --fail --location --show-error \
  "https://raw.githubusercontent.com/nvm-sh/nvm/${NVM_VERSION}/install.sh" \
  --output "$NVM_INSTALLER"
bash -n "$NVM_INSTALLER"
less "$NVM_INSTALLER"
bash "$NVM_INSTALLER"
rm -f -- "$NVM_INSTALLER"
```

Загрузите NVM в текущий shell и установите LTS:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

nvm install --lts
nvm alias default 'lts/*'
nvm use --lts
```

В интерактивных Bash-сессиях NVM загружается из `~/.bashrc`. В неинтерактивных командах `bash -lc` может не читать эту часть `~/.bashrc`; если скрипту нужен NVM, явно выполните `export NVM_DIR=...` и подключите `"$NVM_DIR/nvm.sh"` как в примере выше.

Windows Node.js и глобальные npm-пакеты остаются отдельными от Linux NVM. Предупреждение установщика о Windows-пакетах не означает, что они будут удалены или перенесены в Ubuntu.

## 6. Настроить Git и (опционально) Python virtualenv

Чтобы Git в Ubuntu не переписывал LF-переводы строк при checkout:

```bash
git config --global core.autocrlf input
git config --global --get core.autocrlf
```

Ожидаемое значение — `input`. Эта настройка относится к Git пользователя Ubuntu и не заменяет `.gitattributes` репозитория.

Ядро `claude-obsidian` использует стандартную библиотеку Python; отдельный virtualenv для него не обязателен. Если он нужен для других Python-инструментов, создавайте его отдельно от пользовательского vault:

```bash
mkdir -p "$HOME/.venvs"
python3 -m venv "$HOME/.venvs/wsl-tools"
source "$HOME/.venvs/wsl-tools/bin/activate"
python --version
```

Обычная папка с названием `obsidian-vault` ещё не является инициализированным vault. Не помещайте виртуальное окружение внутрь настоящего vault.

## 7. Создать глобальный `.wslconfig`

Откройте файл из PowerShell:

```powershell
notepad "$env:USERPROFILE\.wslconfig"
```

Если файл уже существует, сначала прочитайте его и объедините настройки; не стирайте неизвестные параметры. Для выбранного профиля используйте:

```ini
[wsl2]
memory=2GB
processors=2
swap=0
networkingMode=nat

[experimental]
autoMemoryReclaim=dropCache
sparseVhd=true
```

`memory`, `processors`, `swap` и `networkingMode` относятся к секции `[wsl2]`. `autoMemoryReclaim` и `sparseVhd` — экспериментальные ключи и должны находиться в `[experimental]`. `sparseVhd=true` автоматически включает sparse-режим для новых VHD; это не сжимает уже существующие диски дистрибутивов.

В этом профиле используется NAT. Mirrored networking на некоторых Windows-конфигурациях не запускается, если IPv6 отключён на хосте; WSL в таком случае сообщает о проблеме и возвращается к NAT. Не меняйте глобальные настройки IPv6 Windows только ради этого профиля.

## 8. Перезапустить WSL и проверить результат

Сохраните работу во всех WSL-дистрибутивах. Затем в PowerShell выполните `wsl --shutdown`. Команда останавливает все запущенные дистрибутивы; запускать её от имени администратора обычно не требуется.

```powershell
wsl --shutdown
wsl --list --verbose
wsl --distribution Ubuntu-26.04
```

После повторного запуска Ubuntu проверьте:

```bash
free -h
nproc
swapon --show
zramctl
sysctl vm.swappiness
systemctl is-active zram-config.service
systemctl is-active systemd-sysctl.service
systemctl --failed --no-pager
git config --global --get core.autocrlf
nvm current
node --version
npm --version
command -v node
```

Для выбранного профиля ожидайте около 2 ГБ памяти, `nproc` равный `2`, `vm.swappiness = 10`, активную службу ZRAM и `/dev/zram0` в `swapon --show`. При `swap=0` WSL-файла подкачки быть не должно; Linux swap должен предоставляться ZRAM. Путь `node` должен находиться под `$HOME/.nvm/`.

Если ZRAM не активна, проверьте `systemctl status zram-config.service` и `journalctl -u zram-config.service -b --no-pager`. Не отключайте WSL swap до устранения причины.

## 9. Удалить ненужный WSL-дистрибутив (необязательно)

Перед удалением проверьте точное имя:

```powershell
wsl --list --verbose
```

Для резервной копии сначала убедитесь, что на диске достаточно места:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\WSL-Backups" | Out-Null
wsl --export NixOS "$env:USERPROFILE\WSL-Backups\NixOS.tar"
```

Только если вы готовы удалить все файлы этого дистрибутива (включая домашний каталог и настройки), выполните:

```powershell
wsl --unregister NixOS
wsl --set-default Ubuntu-26.04
```

`wsl --unregister` необратим без предварительно созданного экспорта. Не запускайте эту команду для имени, которое не проверили через `wsl --list --verbose`.

## Справка

- [Параметры `.wslconfig` и `wsl.conf` от Microsoft](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)
- [Официальный репозиторий NVM](https://github.com/nvm-sh/nvm)
- [Установка и отдельный пользовательский vault claude-obsidian](docs/install-guide.md)
