---
title: Сборка образа
permalink: rails/100_basic/10_build.html
---

В этой главе мы соберём Docker-образ с демо-приложением, используя werf и [Dockerfile](https://docs.docker.com/engine/reference/builder/), а потом запустим приложение в контейнере локально.

## Подготовка

Установите werf и его зависимости, [следуя инструкциям]({{ site.url }}/installation.html).

{% offtopic title="В Windows пользователю также понадобятся права на создание символьных ссылок" %}
1. Откройте PowerShell-терминал с правами администратора.
1. Скопируйте в терминал функцию для добавления прав на создание символьных ссылок:
```powershell
function addSymLinkPermissions($accountToAdd){
    Write-Host "Checking SymLink permissions.."
    $sidstr = $null
    try {
        $ntprincipal = new-object System.Security.Principal.NTAccount "$accountToAdd"
        $sid = $ntprincipal.Translate([System.Security.Principal.SecurityIdentifier])
        $sidstr = $sid.Value.ToString()
    } catch {
        $sidstr = $null
    }
    Write-Host "Account: $($accountToAdd)" -ForegroundColor DarkCyan
    if( [string]::IsNullOrEmpty($sidstr) ) {
        Write-Host "Account not found!" -ForegroundColor Red
        return $false
    }
    Write-Host "Account SID: $($sidstr)" -ForegroundColor DarkCyan
    $tmp = [System.IO.Path]::GetTempFileName()
    Write-Host "Export current Local Security Policy" -ForegroundColor DarkCyan
    secedit.exe /export /cfg "$($tmp)" 
    $c = Get-Content -Path $tmp 
    $currentSetting = ""
    foreach($s in $c) {
        if( $s -like "SECreateSymbolicLinkPrivilege*") {
            $x = $s.split("=",[System.StringSplitOptions]::RemoveEmptyEntries)
            $currentSetting = $x[1].Trim()
        }
    }
    if( $currentSetting -notlike "*$($sidstr)*" ) {
        Write-Host "Need to add permissions to SymLink" -ForegroundColor Yellow

        Write-Host "Modify Setting ""Create SymLink""" -ForegroundColor DarkCyan

        if( [string]::IsNullOrEmpty($currentSetting) ) {
            $currentSetting = "*$($sidstr)"
        } else {
            $currentSetting = "*$($sidstr),$($currentSetting)"
        }
        Write-Host "$currentSetting"
    $outfile = @"
[Unicode]
Unicode=yes
[Version]
signature="`$CHICAGO`$"
Revision=1
[Privilege Rights]
SECreateSymbolicLinkPrivilege = $($currentSetting)
"@
    $tmp2 = [System.IO.Path]::GetTempFileName()
        Write-Host "Import new settings to Local Security Policy" -ForegroundColor DarkCyan
        $outfile | Set-Content -Path $tmp2 -Encoding Unicode -Force
        Push-Location (Split-Path $tmp2)
        try {
            secedit.exe /configure /db "secedit.sdb" /cfg "$($tmp2)" /areas USER_RIGHTS 
        } finally { 
            Pop-Location
        }
    } else {
        Write-Host "NO ACTIONS REQUIRED! Account already in ""Create SymLink""" -ForegroundColor DarkCyan
        Write-Host "Account $accountToAdd already has permissions to SymLink" -ForegroundColor Green
        return $true;
    }
}
```
1. Вызовем эту функцию, указав ей имя пользователя, из под которого вы будете запускать werf:
```powershell
addSymLinkPermissions("YOUR_USERNAME_HERE")
```
1. Выйдем из учётной записи Windows, после чего зайдите снова, чтобы изменения применились.
{% endofftopic %}

## Создадим новый репозиторий с демо-приложением

Все дальнейшие команды потребуется выполнять в PowerShell (для Windows) или Bash (для macOS и Linux).

В отдельной директории на своём компьютере выполним команды:
```bash
git clone https://github.com/werf/werf-guides
cp -r werf-guides/examples/rails/000_app rails-app
cd rails-app
git init
git add .
git commit -m "initial"
```

## Создадим Dockerfile

Реализуем логику сборки нашего приложения с [Dockerfile](https://docs.docker.com/engine/reference/builder/):

{% snippetcut name="Dockerfile" url="https://github.com/werf/werf-guides/blob/master/examples/rails/010_build/Dockerfile" %}
{% raw %}
```Dockerfile
FROM ruby:2.7.1
WORKDIR /app

# Установим системные зависимости
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev libxml2-dev libxslt1-dev curl

# Установим зависимости приложения
COPY Gemfile /app/Gemfile
COPY Gemfile.lock /app/Gemfile.lock
RUN bundle install

# Добавим в образ все остальные файлы из нашего репозитория, включая исходный код приложения
COPY . .
```
{% endraw %}
{% endsnippetcut %}

## Интеграция werf и Dockerfile

Создадим в корне репозитория основной файл конфигурации werf `werf.yaml`, в котором укажем, какой `Dockerfile` должен будет использоваться при сборке с werf:

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/rails/011_build_werf/werf.yaml" %}
{% raw %}
```yaml
project: werf-guided-rails  # Имя проекта, используется в имени Helm-релиза и имени Namespace.
configVersion: 1

---
image: basicapp  # Имя образа, используется в Helm-шаблонах и в части werf-команд.
dockerfile: Dockerfile  # Путь к Dockerfile, содержащему инструкции для сборки.
```
{% endraw %}
{% endsnippetcut %}

В `werf.yaml` может описываться сборка сразу нескольких образов. Также для сборки образа существует ряд дополнительных настроек, с которыми можно ознакомиться [по ссылке]({{ site.url }}/documentation/reference/werf_yaml.html#dockerfile-image-section-image).

## Сборка с werf

Перед выполнением сборки необходимо добавить наши изменения в коммит:
```bash
git add werf.yaml Dockerfile
git commit -m "Add build configuration"
```

> Чуть позже мы разберём, для чего изменения нужно добавлять в коммит перед сборкой/деплоем, и как обойтись без постоянного создания новых коммитов при локальной разработке.

Запустим сборку командой [`werf build`]({{ site.url }}/documentation/reference/cli/werf_build.html):
```bash
werf build
```

Результат выполнения команды при успешной сборке:
{% raw %}
```bash
┌ ⛵ image basicapp
│ ┌ Building stage basicapp/dockerfile
│ │ basicapp/dockerfile  Sending build context to Docker daemon  11.64MB
│ │ basicapp/dockerfile  Step 1/14 : FROM ruby:2.7.1
│ │ basicapp/dockerfile   ---> d8ca85855516
│ │ basicapp/dockerfile  Step 2/14 : WORKDIR /app
│ │ basicapp/dockerfile   ---> Using cache
...
│ │ basicapp/dockerfile  Successfully built 3a4ede4e9556
│ │ basicapp/dockerfile  Successfully tagged 0041b344-efe4-416d-baff-5e50fbb712b0:latest
│ ├ Info
│ │       name: werf-guided-rails:31e0e7436c3055fa816fc770ebda185bacb7e8ef53775b8e5488a83f-1611855308907
│ │       size: 929.2 MiB
│ └ Building stage basicapp/dockerfile (94.47 seconds)
└ ⛵ image basicapp (96.07 seconds)

Running time 96.38 seconds
```
{% endraw %}

## Запуск приложения

Запустить контейнер локально на основе собранного образа можно командой [werf run]({{ site.url }}/documentation/cli/main/run.html):
```bash
werf run --docker-options="--rm -p 3000:3000" basicapp -- bash -ec "bundle exec rails db:migrate RAILS_ENV=development && bundle exec puma"
```

Здесь [параметры Docker](https://docs.docker.com/engine/reference/run/) мы задали опцией `--docker-options`, а команду для выполнения в контейнере указали в конце, после двух дефисов.

Теперь приложение доступно на [http://127.0.0.1:3000/](http://127.0.0.1:3000/):

{% asset guides/rails/100_10_app_in_browser.png %}
