_(Please contact me if you need English version of this instruction — it is supposed to be mostly used by Russian[only]-speaking folks.)_

Репозиторий делает из пустого Убунту-сервера полностью готовое окружение, запускающее minstall от lleo-движка.

Может использоваться как для настройки боевого сервера, так и для запуска окружения для разработки; запускается на практически любой современной ОС _(теоретически — включая Windows, хотя это не тестировалась)_.

Скрипт можно повторно применять к уже развёрнутому окружению — он ничего не сломает, но зато обновит пакеты и (чаще всего) починит сломанные настройки.

## Подготовка к запуску

1. Нам понадобится Python и его менеджер зависимостей `pip` на вашем компьютере. Скрипты написаны на языке конфигурации систем `Ansible`, который запускается при помощи Питона.

  Под Убунту достато сделать `sudo apt-get install python-pip`, и всё установится как надо. Под другими линуксами — найдите Python и Pip у себя в пакетном менеджере. Под Windows — [установка Python](https://www.python.org/downloads/windows/) и [установка Pip](https://pip.pypa.io/en/latest/installing/).

2. Ставим собственно Ansible: `sudo pip install ansible`.

  Проверить, что всё встало, можно командой `ansible --version`. Она должна вывести что-то вроде

  ```
  ansible 1.9.4
  configured module search path = None
  ```


## Запуск скрипта на боевой сервер

От удалённого сервера требуется современный линукс _(тестировалась Ubuntu Trusty, большинство deb-подобных ОС должны работать)_; к этому линуксу должен быть настроен ssh-доступ **по ключу**.

Ключ можно загрузить на удалённый сервер, например, так: `ssh-copy-id remote_user@dnevnik.example.com`. После этого `ssh remote_user@dnevnik.example.com` должен срабатывать, не спрашивая пароля.

Скрипт включит на удалённой машине файрволл и донастроит ssh для большей безопасности. **После применения скрипта входить на сервер по ssh с паролем больше не получится!**

Естественно, `remote_user` должен уметь `sudo` на удалённом сервере; без этого в любом случае не получится ставить там никакой софт.

#### Важно

Пожалуйста, перед запуском убедитесь что файл `servers` наводит скрипт на правильный сервер и правильного пользователя.

Кроме того, поменять пароли в `settings.yml` кажется хорошей идеей.


#### Наконец, запускаем скрипт:

(на своей рабочей машине)

```bash
ansible-playbook -i servers bootstrap_server.yml -K
```

Скрипт сам приконнектится к серверу, пожужжит немножко, всё там установит и поднимет — остаётся только зайти на http://dnevnik.example.com и проапдейтиться (войдя под админским паролем из `settings.yml`).

## Запуск окружения разработчика

Окружение запускается в отдельной вируальной машине с линуксом Ubuntu Trusty — вне зависимости от того, какая ОС у вас на десктопе. Для изоляции используется VirtualBox И Vagrant.

[Ставим VirtualBox](https://www.virtualbox.org/wiki/Downloads).

[Ставим Vagrant](https://www.vagrantup.com/downloads).

#### Запускаем локальное окружение

```bash
vagrant up
```

Эта команда запустит виртуальную машину, всё там настроит и выставит движок на локальном порту 8080. Поскольку движок жёстко привязан к домену (и должен быть доступен извне, чтобы апдейтиться), **без домена он не заработает**. Поэтому нам потребуется следующий шаг.

#### Делаем окружение доступным по доменному имени

```bash
vagrant share
```

Команда выдаст случайное доменное имя, по которому к виртуальному окружению можно коннектиться из внешнего интернета. Фокус работает, в том числе, без прямого IP и за любыми НАТами.

Пока команда запущена — сервер доступен извне. Как только она остановлена через `Ctrl+C`, внешний доступ прекращается. **Доступной становится только виртуалка с движком**, сам десктоп не выставляется наружу.

Сходив по этому доменному имени, мы попадём в админку апдейтера движка, которая скачает наисвежайшую версию в текущую папку, в подпапку `dnevnik/`. Файлы в этой подпапке можно редактировать локально — и виртуальное окружение подхватит эти изменения на лету, автоматически.

Воспользуемся этим, и привяжем движок к нашему случайному домену, отредактировав строчку `$httpsite =` в `dnevnik/config.php`.

#### Делаем окружение доступным по собственному, постоянному доменному имени

Если у нас есть контроль над доменом `example.com`, то нам надо будет делегировать ДНС-запись вроде `*.shared.example.com` серверам VagrantCloud, через CNAME на `share.vagrantcloud.com`.

Кроме того, использование нашего домена надо активировать в админке VagrantCloud.
Для этого регистрируемся на https://atlas.hashicorp.com (HashiCorp — компания, создавшая и обслуживающая Vagrant).
Идём в админку https://atlas.hashicorp.com/settings/organizations/<username>/configuration , и в самом низу добавляем наш домен `shared.example.com` (в этот раз — без звёздочки).

Теперь на локальной машине можно залогиниться в VagrantCloud (`vagrant login`) и пользоваться поддоменами:

```bash
vagrant share --domain shared.example.com --name dnevnik
```

Выбранный домен по-прежнему надо руками вносить в `dnevnik/config.php`.

#### Останавливаем локальное окружение

```bash
vagrant halt
```

#### Удаляем вируальную машину

Для ускорения последующих запусков виртуалка с рабочим окружением сохраняется на диске между запусками.
Если она капитально сломана (или просто на неё жалко места) — виртуалку можно удалить:

```bash
vagrant destroy
```
