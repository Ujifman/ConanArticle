# ConanArticle

Статья про наше использование Conan для Habr.

**Title**: Как мы навели порядок в C++/Qt проекте с помощью Conan

==Сразу оговорюсь, что цель статьи - рассказать, о том как нам удалось выстроить достаточно стабильный flow работы.== перефразировать

## Пару слов о проекте

### Максимально коротко

Мы создаем систему по обмену аэронавигационной информацией между филиалами организации, которая разрабатывает и поддерживает структуру воздушного пространства в РФ. Система распределенная: в каждом филиале установлен автономный комплекс. Комплексы из разных филиалов обмениваются между собой изменениями аэронавигационной информации.

### Теперь подробнее

Комплекс представляет из себя несколько железных серверов в кластере, на котором работает набор backend сервисов (серверная система CentOS), и несколько рабочих мест операторов (ПК на Windows). На рабочих местах установлено Desktop приложение на C++/Qt, которое взаимодйствует с backend'ом. Система разрабатывается уже много лет, поэтому мы имеем неплохой багаж legacy.

Desktop приложение модульное, главный UI подгружает из динамических библиотек интерфейсы для выполнения различных задач. Каждый отдельный модуль имеет свою либу (читай `dll`/`so`). Кроме того модули используют общий код доступа к БД, доступа к backend сервисам, реализации логгирования и т.д. Общий код также выдлен в отдельные либы статические и динамические.

Некоторые backend сервисы также написаны на C++/Qt и используют библиотеки доступа к БД, те же самые, которые использует Desktop приложение.

Итого имеем на C++/Qt:

- 26 динамических библиотек (15 GUI модулей и 11 модулей с бизнес логикой)
- 11 статических библиотек
- 6 сторонних open source библиотек (докрученных для наших нужд)
- 2 Desktop приложения
- 2 backend сервиса

Все вышеперечисленное местами друг друга использует.

В итоге на упрощенном примере получается следующая карта зависимостей:

```mermaid
graph TD
app_client(ClientApp)
lib_users(UsersGuiModule)
lib_data(DataGuiModule)
gui_prim(GuiPrimitives)
lib_db(DbInterface)
templates(Templates)
app_backend(SomeBackendApp)

app_client --> lib_users & lib_data
lib_users & lib_data --> gui_prim
lib_users & lib_data --> lib_db
lib_users & gui_prim & lib_db--> templates
app_backend ---> lib_db & templates
```

Теперь эту схему масштабируем до нашего количества либ и возникает вопрос: как все это согласованно содержать и допиливать?

## Проблемы, которые надо решить

- Централизованное управление всем деревом зависимостей, типовые пайплайны сборки
- Независимая разработка модулей: для разработчика удобно запускать определенный GUI модуль отдельно от всего приложения - изолированно, для этого ему нужны актуальные версии всех зависимостей
- Неплохо бы разделять `dev` и `production` пространства и сделать удобным доступ для тестировщиков к свежим фичам, при этом случайно не выкатить в прод сырой код.
- Еще не забываем про Git Flow и ветвление: 1 фича - отдельная ветка; все мы знаем, что в одной ветке пилить плохо. Это все прекрасно работает, пока репозиторий один. Когда итоговое Desktop приложение включает в себя кучу репозиториев, а некоторые из них зависят от других репозиториев, мы приходим к весьма сложной и запутанной структуре.
- Можно еще подумать о сокращении времени CI/CD на build и билдить либы изолированно друг от друга и хранить
- 2 платформы Linux/Windows и минимум 2 версии Qt (переходные периоды в любом случае присутствуют)
- Иногда баг сквозит через несколько либ, и надо иметь возможность дебажить сквозь несколько зависимостей
- В основном мы используем QMake, но некоторые сторонние либы используют CMake, надо уметь с этим жить

## У самурая нет цели, есть только путь

Много много лет назад в одной отдаленной галактике... мы поняли, что так дальше жить нельзя, надо искать решение.

### Вариант 1, монорепозиторий

Когда проект стартовал, он весь состоял из Desktop приложения и БД, backend сервисов тогда еще не было. Это было лет 8 назад, и Desktop приложение было в монорепозитории. В корне лежал один большой Qt `.pro` файл, который через `subdirs` включал все GUI модули и внутренние либы. Все они плоско лежали в репозитории в подпапках, собирались по порядку и подключались уже бинарями.

Плюсы:

- Все в одном месте, просто фиксить баг, который сквозит через библиотеки или делать фичу, которая сквозит через библиотеки
- Нет проблем с ветвлением

Минусы:

- Дикое время сборки
- Высокая вероятность высокой связанности (coupling)
- Все разработчики в одном репозитории
- Очень сложно взять либу в другой проект
- Сложно организовать изолированное приложение для одного GUI модуля

### Вариант 2, git submodules

Мы начали искать варианты разделения на отдельные репозитории. И на тот момент единственным вариантом выглядел `git submodules`. В итоге распилив монорепозиторий мы получили, каждую либу в отдельном репозитории. Репозиторий главного Desktop приложения подключает их как git сабмодули, и в принципе репозиторий Desktop приложения на вид не изменился. Зато появилась возможность отдельно работать с GUI модулем. И тут возникла новая сложность.

Возникла проблема с подключением либ с общим кодом (например доступа к БД), если ее вложить в каждый репозиторий GUI модуля, то когда они все собираются в репозитории главного GUI приложения получается куча дублирований одного и того же общего кода.

Поэтому было решено либы с общим кодом не подключать в GUI модули git сабмодулями. В репозиторий главного приложения уложить GUI модули и либы с общим кодом на одном уровне в корне. Тогда инклюды в GUI модулях всегда идут на шаг вверх и в папку нужного модуля с общим кодом.

Это привело к тому, что код в репозитории GUI модуля, у которого есть зависимости, несамостоятельный, и собрать его просто клонировав репозиторий не получится. Для того, чтобы запустить код модуля отдельно отприложения мы создавали отдельный репозиторий с сэмплом, и в него включали сам модуль и его зависимости. В итоге куча лишних репозиториев.

**Упрощенный пример репозитория главного приложения:**

```mermaid
graph LR
app_client(ClientApp Repo)
lib_users(UsersGuiModule Repo)
lib_data(DataGuiModule Repo)
gui_prim(GuiPrimitives Repo)
lib_db(DbInterface Repo)
templates(Templates Repo)

app_client --> lib_users & lib_data & gui_prim & lib_db & templates
```

**Упрощенный пример репозитория бэкенд приложения:**

```mermaid
graph LR
lib_db(DbInterface Repo)
templates(Templates Repo)
app_backend(SomeBackendApp Repo)

app_backend ---> lib_db & templates
```

**Упрощенный пример репозитория с сэмплом:**

```mermaid
graph LR
app_client(UsersGuiSample Repo)
lib_users(UsersGuiModule Repo)
gui_prim(GuiPrimitives Repo)
lib_db(DbInterface Repo)
templates(Templates Repo)

app_client --> lib_users & gui_prim & lib_db & templates
```

Плюсы:

- Разбиение на модули по изолированным репозиториям
- Сложно фиксить баг или делать фичу, которая сквозит через несколько либ (это проблема высокой связанности, поэтому сложности автоматически стимулируют лучше разделять, так что это необходимые сложности которые идут во благо)

Минусы:

- Постоянная возня с `git submodules update`
- Каждое изменение ветки подмодуля приводит к коммиту в репозитории, который его включает, чтобы обновить ссылку подмодуля
- Куча лишних репозиториев с сэмплами, с такими же проблемами с ветвлением
- Все такое же дикое время сборки
- И самое сладкое, полнейший дурдом с ветвлением

### Вариант 3, пакетный менеджер

И вот в один прекрасный день мой коллега (спасибо @madmax) нашел его:

![2022-05-15-14-59-43.png](README.assets/2022-05-15-14-59-43.png)

Точнее его:

![2022-05-15-15-00-18.png](README.assets/2022-05-15-15-00-18.png)

Мы потратили примерно 2 месяца на осознание, набивание шишек, построение и разрушение костылей и велосипедов. Потом пришел бизнес и сказал, что пора уже код писать.
Мы запустились на том, что получилось.
Потом еще пару лет эволюции и сейчас наша концепция выглядит вполне живой и бодрой.

### Инкапсуляция логики сборки

Вся логика сборки под разные платформы инкаспулирована в общий `conanfile.py`, который также является Conan пакетом и инклюдится в либы.
Также мы в него уложили логику прокатки unit тестов, сбора покрытия, изменение логики сборки в зависимости от того является ли библиотека header only, статической, динамической или app.

==TODO что вошло в conanfile общий, список крупно==
== mermaid схему наследования в Conanfile==

Посмотреть на наш conafile можно по ссылке ==link==.



Пример рецепта для библиотеки:

```python
from conans import ConanFile, CMake, tools
import os


class AixmDbLegacyConan(ConanFile):
    name = "AixmDbLegacy"
    version = "2.58.1"
    url = "https://git.monitorsoft.ru/cpp-libs/AixmDb"
    generators = "qmake"
    python_requires = "CommonConanFile/0.8@monsoft/stable"
    python_requires_extend = "CommonConanFile.DynamicLibConanFile"
    exports_sources = "src/*", "test_unit/*", "AixmDbLegacy.pro", "AixmDbLegacy_TestUnit.pro"
    run_tests_headless = False
    unit_test_executables = [
        os.sep.join([".", "test_package", "DbPrimitives", "AixmDbLegacy_Test_DbPrimitives"]),
        os.sep.join([".", "test_package", "GmlHandler", "AixmDbLegacy_Test_GmlHandler"]),
        os.sep.join([".", "test_package", "AixmDb", "AixmDbLegacy_Test_AixmDb"]),
        os.sep.join([".", "test_package", "Integrational", "AixmDbLegacy_Test_Integrational"])
    ]

    build_requires = (
                "CommonQmakePri/[~1.0.1]@monsoft/stable",
                "ZhrGeo/[~1.1]@monsoft/stable",
                "QTester/[~1.0.1]@monsoft/stable", # for tests
                "FakeIt/2.0.2@hinrikg/stable") # for tests
    requires =  (
                "Lib/[~2.24.0]@monsoft/stable",
                "Templates/[~1.7.2]@monsoft/stable",
                "Sax/[~1.1]@monsoft/stable")
```

### Управление пространствами dev/prod

Идентификация пакета Conan выглядит так `Lib/[~2.24.0]@monsoft/stable`.
`<Имя пакета>/<Версия semver>@<user>/<channel>`
Все наши пакеты (которые используют общий рецепт `CommonConanFile`) из ветки `dev` собираются в канал `dev`, а из ветки `master` в канал `stable`.

Таким образом слияние feature-ветки в `dev` приводит к выходу новой `dev` версии, при этом `stable` простарнство не затрагивается. При релизе мы сливаем все либы из `dev` в `master` и получаем обновление `stable` версий пакетов.

Самая главная фича тут в подмене канала. Все зависимости прописаны на канал `stable`, но когда мы понимаем, что собираемся в `dev` пространстве, то при выполнении `conan install` выставляем env (`OVERRIDE_CONAN_CHANNEL`), на который реагирует наш рецепт сборки, и он подменяет все пакеты в `requires` и `build_requires` с `monsoft/stable` на `monsoft/dev`. Таким образом мы по всему дереву зависимостей получаем подмену канала.

У тестировщика есть команда для установки приложения через Conan из `dev` и `stable`. Ему не надо ничего собирать, ставить IDE, компилятор или еще что-то, чтобы добыть самый свежий бинарь.

Разработчик прислал ветку на ревью -> ветку слили в `dev` -> прошла сборка на билд сервере  -> новая версия в канале `dev` -> у тестировщика самая свежая версия.

### Отладка сквозных багов: editable пакеты

==описать editable==

### Дружба с IDE

==выложить в репу qtCreator и описать==

### Примеры CI/CD

![2022-05-15-15-33-19.png](README.assets/2022-05-15-15-33-19.png)

## Итоговый flow

==может сюда итог подвести==

## Что получилось в итоге

- Каждая библиотека, приложение лежит в своем репозитории и собирается в Conan пакет и выкладывается на Conan сервер
- На Conan сервере есть 2 канала `dev` и `stable`. На `dev` кладутся пакеты собранные из `dev` ветки, на `production` - из `master` ветки
- У тестировщика всегда есть доступ к самым последним фичам в `dev`, при этом легко может переключиться и получить полное приложение из `stable` канала
- Легко можно с помощью `editable` отлавливать сквозные баги
- Общий код реализации сборки, unit тестов (+ coverage), подмены канала, можно централизованно менять для всех
- Абсолютно идентичный и простой код CI/CD
- Возможность увидеть полное дерево зависимостей
- Версионирование по semver для библиотк и как следствие
- Из коробки разделение бинарей библиотек по ОС, версиям компилятора, версиям qt и еще чему угодно, что придет нам в голову
- В виде Conan пакетов можно подключать и совсем не `cpp` вещи, например файлы описывающие модель данных для mock*а* в unit естах (само описание модели данных лежит в отдельной репе с смаодельным синтаксисом)

## Заключение

Все плюсы перечислены выше, само собой у подхода есть и минусы, серебряной пули не бывает.

Минусы:

- блокировка слияния фич перед релизом ==todo write==
- перезапуск сборок скриптами
- дополнительные действия для отладки сковзных багов
- определенный порог входа для разработчиков
- басфактор в поддержке
- высокий порог вхождения

==добавить минусов==
