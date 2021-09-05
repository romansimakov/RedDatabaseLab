.. _Flask: https://flask.palletsprojects.com/en/2.0.x/tutorial/
.. _СУБД Ред База Данных: https://reddatabase.ru

**********************************
Разработка web-приложение на Flask
**********************************

В данной лабораторной работе рассматривается пример создания простейшего веб-приложения, назвываемого **Flaskr**. Пользователи могут регистрироваться, входить в систему, создавать посты, редактировать и удалять свои посты.

За основу взята пошаговая инструкция с официального сайта `Flask`_, однако в качестве СУБД выступает встраиваемая версия российской `СУБД Ред База Данных`_.

Создание приложения
===================

Прежде всего необходимо создать каталог, в котором будет располагаться проект.

.. code-block:: bash

    $ mkdir redflaskr
    $ cd redflaskr

Далее будем предполагать, что вся работа выполняется в каталоге ``redflaskr``. Все пути к файлам будем указывать относительно него.

Теперь необходимо настроить виртуальное окружение для Python (Python virtual environment) и установить Flask и драйвер СУБД Ред База Данных.

Виртуальное окружение используется для управления зависимостями проекта как при разработке так и при разворачивании на продуктиве. Чем больше проектов есть у разработчика, тем более вероятно что они требуют различных версий библиотек Python или даже разных версий самого Python. Новые версии одних библиотек могут разрушить совместимости в другом проекте.

Виртуальное окружение (virtual environment)
    Независимая группа библиотек для каждого проекта. Пакеты установленные для одного проекта не влияют на другие проекты или пакеты операционной системы.

Python поставляется с модулем ``venv`` для создания виртуального окружения.

Для создания виртуального окружения выполните команду

.. code-block:: bash

    $ python3 -m venv venv

Перед началом работы над проектом активируйте соответствующее окружение:

.. code-block:: bash

    $ . venv/bin/activate

В активированном окружении установите Flask и драйвер для СУБД Ред База Данных:

.. code-block:: bash

    $ pip install Flask
    $ pip install fdb

Проекты на Python используют пакеты для организации кода и мы этим воспользуемся.

Каталог проекта будет содержать:

* :flaskr: Пакет Python, содержащий код приложения и другие файлы.
* :venv: Виртуальное окружение, в котором будет установлен Flask, драйвер СУБД fdb и другие зависимости.

Приложение Flask это объект (instance) класса Flask. Все, что связано с приложением (настройки, URL адреса, прочее) будет настраиваться в этом объекте.

Наиболее простой путь создания приложения - это создать глобальный объект непосредственно в начале программы, однако по мере роста проекта это может принести проблемы.

Вместо создания объекта глобально, мы будем создавать его внутри функции. Такая функция называется *фабрикой приложения* (application factory). Все настройки, регистрации и т.п. будут происходить внутри функции, после чего объект приложения будет возвращен.

Фабрика приложения
------------------

Создайте каталог ``flaskr``, а внутри файл ``__init__.py``, который содержит *фабрику приложения* и говорит Python, что катало ``flaskr`` должен рассматриваться как пакет.

.. code-block:: bash

    $ mkdir flaskr

``flaskr/__init__.py``

.. code-block:: python

    import os

    from flask import Flask

    def create_app(test_config=None):
        # create and configure the app
        app = Flask(__name__, instance_relative_config=True)
        app.config.from_mapping(
            SECRET_KEY='dev',
            DATABASE=os.path.join(app.instance_path, 'flaskr.fdb'),
            USER='sysdba',
            PASSWORD='masterkey',
            LIBRARY=os.path.join(app.root_path, 'rdb/libfbclient.so')
        )

        if test_config is None:
            # load the instance config, if it exists, when not testing
            app.config.from_pyfile('config.py', silent=True)
        else:
            # load the test config if passed in
            app.config.from_mapping(test_config)

        # ensure the instance folder exists
        try:
            os.makedirs(app.instance_path)
        except OSError:
            pass

        # a simple page that says hello
        @app.route('/hello')
        def hello():
            return 'Hello, World!'

        return app

``create_app`` это функция *фабрика приложения*. Позже она будет дополнена, но и сейчас она многое делает:

#. :code:`app = Flask(__name__, instance_relative_config=True)` создает объект приложения Flask.
    :__name__: имя текущего модуля Python. Приложению необходимо знать где оно располагается, чтобы установить некоторые пути.
    :instance_relative_config: говорит приложению, что файлы конфигурации размещаются относительно каталога ``instance``. Он размещается вне каталога ``flaskr`` и содержит локальные данные: конфигурационные файлы, БД.

#. :code:`app.config.from_mapping` устанавливает значения параметров конфигурации по умолчанию.
    :SECRET_KEY: используется классом Flask и расширениями для обеспечения безопасности хранимых данных. Значение ``dev`` позволяет удобно разрабатывать приложения, но должно быть заменено случайным значением при поставке приложения заказчику.
    :DATABASE: путь к файлу БД. БД размещается в каталоге ``instance``. В зависимости от нужд приложения, может быть любым, в том числе псевдонимом БД на удаленном сервере. В нашем случае мы воспользуемся встроенным сервером.
    :USER: Имя пользователя, от которого будет производиться соединение.
    :PASSWORD: Пароль. Для встроенного сервера игнорируется.
    :LIBRARY: путь до клиентской библиотеки ``libfbclient.so``, которую мы установим в следующей части.

#. :code:`app.config.from_pyfile` перезаписывает значения параметров конфигурации значениями из файла ``config.py`` каталога ``instance``, если он существует. Например, при поставке приложения в нем можно указать реальное значение ``SECRET_KEY``.

#. :code:`os.makedirs()` гарантирует существование каталога ``app.instance_path``. Flask не создает каталог автоматически, но он нужен для файла БД.

#. :code:`@app.route()` создает простой маршрут, чтобы убедиться что приложение работает, прежде чем продолжить его разрабатывать. Это связывает URL ``/hello`` и функцию, которая сформирует ответ. В данном случае строку 'Hello, World!'.

Запуск приложения
-----------------

Теперь можно запустить приложение, используя команду :code:`flask`. Укажите Flask где искать приложение и запустите его в режиме разработчика.

.. warning:: Вы должны быть в каталоге redflaskr, но не в его подкаталогах.

Режим разработчика показывает интерактивный отладчик когда страница выбрасывает исключение и перезапускает сервер, когда вы делаете изменения в коде. Его можно оставить запущенным и просто обновлять страницу в браузере по мере разработки.

.. code-block:: bash

    $ export FLASK_APP=flaskr
    $ export FLASK_ENV=development
    $ flask run

Вы увидите вывод, подобный этому:

.. code-block:: console

  * Serving Flask app "flaskr"
  * Environment: development
  * Debug mode: on
  * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
  * Restarting with stat
  * Debugger is active!
  * Debugger PIN: 855-212-761

Перейдите по адресу http://127.0.0.1:5000/hello в браузере и вы увидите сообщение "Hello, World!".

Работа с БД
===========

Приложение будет использовать `СУБД Ред База Данных`_ для хранения пользователей и их постов.

Данная СУБД поддерживает SQL-2011 и может работать как во встроенном режиме без установки сервера, так и в традиционном клиент-сервером варианте. Для веб-приложения удобным является использование встроенного режима.

Установка встроенной СУБД
-------------------------

Скачайте последнюю версию СУБД с официального сайта https://reddatabase.ru/downloads.
Для работы будет достаточно открытой редакции ``XXX-linux-x86_64.tar.gz`` и распакуйте его в любой каталог, например ``/tmp``.

В каталоге ``flaskr`` сделайте подкаталог ``rdb`` для установки встроенной версии СУБД Ред База Данных, в в нем подкаталог ``plugins``.

.. code-block:: bash

    $ mkdir flaskr/rdb
    $ mkdir flaskr/rdb/plugins

Скопируйте из распакованного архива СУБД файлы ``libfbclient.so`` и ``libEngine12.so`` в только что созданные каталоги.

.. code-block:: bash

    $ cp /tmp/RedDatabase-X.X.X-x86_64/lib/libfbclient.so.X.X.X flaskr/rdb/libfbclient.so
    $ cp /tmp/RedDatabase-X.X.X-x86_64/plugins/libEngine12.so flaskr/rdb/plugins

Если СУБД установлена на этом сервере или любом другом, то потребуется лишь библиотека клиента ``libfbclient.so``. Можно ее и не копировать, а просто указать ее расположение в файле конфигурации, в параметре ``LIBRARY``.

Подключение к БД
----------------

При работе с БД первое что необходимо сделать, создать подключение. Все запросы выполняются через подключение, которое закрывается когда работа выполнена.

В web-приложениях подключение обычно привязывается к запросу. В какой-то момент обработки запроса оно создается, а перед отправкой ответа закрывается.

Кроме этого необходимо написать код для инициализации БД.

``flaskr/db.py``

.. code-block:: python

    import fdb

    import click
    from flask import current_app, g
    from flask.cli import with_appcontext


    def init_db():
        try:
            conn = fdb.connect(
                dsn=current_app.config['DATABASE'],
                user=current_app.config['USER'],
                password=current_app.config['PASSWORD'],
                fb_library_name=current_app.config['LIBRARY']
            )
            conn.drop_database()
        except Exception as e:
            print(e)

        
        conn = fdb.create_database(
            dsn=current_app.config['DATABASE'],
            user=current_app.config['USER'],
            password=current_app.config['PASSWORD'],
            fb_library_name=current_app.config['LIBRARY']
        )

        metadata = [
            '''
            RECREATE TABLE users (
                id integer generated by default as identity primary key,
                username varchar(256) UNIQUE NOT NULL,
                password varchar(256) NOT NULL
            )
            ''',
            '''
            RECREATE TABLE posts (
                id integer generated by default as identity primary key,
                author_id INTEGER NOT NULL REFERENCES users (id),
                created TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
                title varchar(120) NOT NULL,
                body varchar(5000) NOT NULL
            )
            '''
        ]

        cursor = conn.cursor()

        for query in metadata:
            cursor.execute(query)

        conn.commit()


    def get_db():
        if 'db' not in g:
            g.db = fdb.connect(
                dsn=current_app.config['DATABASE'],
                user=current_app.config['USER'],
                password=current_app.config['PASSWORD'],
                fb_library_name=current_app.config['LIBRARY']
            )

        return g.db


    def close_db(e=None):
        db = g.pop('db', None)

        if db is not None:
            db.close()


    @click.command('init-db')
    @with_appcontext
    def init_db_command():
        """Clear the existing data and create new tables."""
        init_db()
        click.echo('Initialized the database.')


    def init_app(app):
        app.teardown_appcontext(close_db)
        app.cli.add_command(init_db_command)

:code:`get_db` создает подключение к БД.

:g: специальный объект, уникальный для каждого запроса. Он используется для хранения данных, которые могут использоваться множеством функции во время обработки запросы. Подключение создается и повторно используется, если :code:`get_db` вызывается не первый раз.

:current_app: другой специальный объект, который указывает на приложение Flask, обрабатывающее запрос.

:fdb.connect: устанавливает подключение к БД используя параметры конфигурации. Файл БД создается в функции :code:`init_db`.

:code:`close_db` закрывает подключение к БД, если ``g.db`` установлен.

:code:`init_db` создает БД и необходимые объекты.

:metadata: список SQL команд для создания объектов БД: таблицы пользователей ``users`` и таблицы постов ``posts``.

Вначале функция пытается установить соединение с имеющейся БД для того, чтобы ее удалить. В случае неудачи печатается исключение для целей отладки. Далее в люблом случае создается новая БД, используя все теже параметры конфигурации.

:conn.cursor(): создает объект *курсор*, с помощью которого можно выполнять все запросы к СУБД.

В цикле выполняются все запросы из списка метаданных и в завершении производиться завершение транзакции и применение всех изменений.

:code:`click.command()` определяет команду командной строки "init-db", которая вызывает функцию ``init_db`` и сообщает об успешности выполнения инициализации пользователю.

:code:`init_app(app)` регистрирует созданные функции.

Функции ``close_db`` и ``init_db_command`` должны быть зарегистрированы в объекте приложения, иначе они не будут использоваться.

:app.teardown_appcontext(): говорит Flask вызвать указанную функцию при очистке после отправки ответа.

:app.cli.add_command(): добавляет новую команду, которая может вызываться с командой ``flask``.

Импортируйте и добавьте вызов этой функции в *фабрике приложения*.

``flaskr/__init__.py``

.. code-block:: python

    def create_app():
    app = ...
    # existing code omitted

    from . import db
    db.init_app(app)

    return app

Инициализация БД
----------------

Теперь, когда команды ``init-db`` зарегистрирована, она может быть вызвана используя команду ``flask``, аналогично команде ``run``.

.. warning:: Если у вас все еще запущен сервер с предыдущего этапа, необходимо его остановить или выполнить команду в другом терминале. Помните что необходимо находиться в каталоге ``redflaskr`` и активировать виртуальное окружение.

Запустите команду

.. code-block:: bash

    $ flask init-db
    Initialized the database.

В каталоге ``instance`` должен появится файл ``flaskr.fdb``.

