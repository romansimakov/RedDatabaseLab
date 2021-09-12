.. _Flask: https://flask.palletsprojects.com/en/2.0.x/tutorial/
.. _Jinja: https://jinja.palletsprojects.com/en/3.0.x/templates/
.. _СУБД Ред База Данных: https://reddatabase.ru

.. flask-tutorial:

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

Эскизы (Blueprints) и представления (Views)
======================

:Функция-представление: (View function) функция, которыя отвечает на запросы к приложению. Flask использует шаблоны, чтобы сопоставить входящие URL запросов функциям-представлениям.

:Эскиз: (Blueprint) способ организовать группы связанных представлений и другой код. Вместо регистрации представление и другого кода непосредственно в приложении, они регистрируются в эскизе. Далее эксиз регистрируется в приложении, в фабрике приложения.

Создание эскиза
---------------

Flaskr будет иметь два эскиза. Один для аутентификации, другой для функций работы с постами. Код для каждого эскиза будет размещатся в отдельных модулях. В первую очередь сделаем модуль аутентификации.

``flaskr/auth.py``

.. code-block:: python

    import functools

    from flask import (
        Blueprint, flash, g, redirect, render_template, request, session, url_for
    )
    from werkzeug.security import check_password_hash, generate_password_hash

    from flaskr.db import get_db

    bp = Blueprint('auth', __name__, url_prefix='/auth')

Этот код создает эскиз под названием 'auth'. Как и объект приложение, эскиз должен знать где его создали. Для этого ``__name__`` передается вторым параметром. ``url_prefix`` будет предшествовать всем URL адресам, связанным с этим эскизом.

Импортируйте и зарегистрируйте эскиз в *фабрике приложения*, используя ``app.register_blueprint()``. Добавьте следующий код в конец *фабрики приложения* перед возвращением объекта приложения.

``flaskr/__init__.py``

.. code-block:: python

    def create_app():
        app = ...
        # existing code omitted

        from . import auth
        app.register_blueprint(auth.bp)

        return app

Эскиз аутентификации будет иметь функции-представления для регистрации новых пользователей и входа/выхода существующих.

Первое представление: регистрация
---------------------------------

Когда пользователь заходит на страницу ``/auth/register`` представление ``register`` вернет HTML код с формой для заполнения. При отправке данных формы она будет проверена и либо будет показано сообщение об ошибке, либо будет создан новый пользователь и осуществлен переход на страницу входа.

Сейчас просто допишем код функции-представления к имеющемуся коду файлы ``flaskr/auth.py``. Далее будут написаны шаблоны для генерации HTML форм.

``flaskr/auth.py``

.. code-block:: python

    # existing code
    #...

    @bp.route('/register', methods=('GET', 'POST'))
    def register():
        if request.method == 'POST':
            username = request.form['username']
            password = request.form['password']
            db = get_db()
            error = None

            if not username:
                error = 'Username is required.'
            elif not password:
                error = 'Password is required.'

            if error is None:
                try:
                    db.cursor().execute(
                        "INSERT INTO users (username, password) VALUES (?, ?)",
                        (username, generate_password_hash(password)),
                    )
                    db.commit()
                except db.DatabaseError as e:
                    if e.args[2] == 335544665:  #isc_unique_key_violation
                        error = f"User {username} is already registered."
                else:
                    return redirect(url_for("auth.login"))

            flash(error)

        return render_template('auth/register.html')

Вот что делает эта функция:

1. :@bp.route: сопоставляет URL ``/register`` с функцией-представлением ``register``. Когда Flask получит запрос на ``/auth/register`` будет вызвана функция ``register`` и ее результат будет отправлен пользователю.
2. Для отправки формы пользователем используется метод ``POST``. Если это так, то проверяет входные параметры.
3. :request.form: специальный тип ассоциативного массива с параметрами формы, содержащий пары ключ-значение. Пользователь будет вводить туда ``username`` и ``password``.
4. Проверяем что введенные значение не пустые.
5. Если проверка успешна, вставляем данные нового пользователя в базу данных.

   * :db.cursor().execute: создает объект курсор из подключения и вызывает его метод execute для выполнения запроса на вставку в таблицу. Метод принимает SQL динамического запроса, с ``?`` вместо значений и список значений. Такой способ защищает от SQL инъекций и строго рекомендуется в отличие от конструирования запроса как статического в виде одной строки.
   * Для целей безопасности пароль не храниться в БД в открытом виде. Он хешируется функцией ``generate_password_hash()``. Поскольку этот метод меняет данные, то транзакцию необходимо подтвердить с помощью ``db.commit()``.
   * :db.DatabaseError: исключение, которое будет выброшено при ошибке выполнения запроса. Код ошибки ``335544665`` соответствует ошибке дубликата первичного ключа, т.е. возникнет когда такой пользователь уже есть.
  
6. После сохранения пользователя он перенаправляется на страницу входа. ``url_for()`` генерирует URL адрес представления на основе его имени. Это предпочтительнее прямой записи URL, т.к. позволяет менять адрес позднее, без смены остального кода. ``regirect()`` генерирует ответ для перенаправление на указанный (сгенерированный) URL адрес.
7. Если проверка параметров будет неудачной, то пользователю будет показана ошибка. ``flash()`` сохраняет сообщения и они будут показаны при следующем рендеринге шаблона.
8. При первом открытии страницы ``/auth/register`` или при ошибке проверки параметров, должна быть снова показана страница регистрации. ``render_template()`` рендерит шаблон, содержащий HTML, который мы вскоре напишем.

Представление: вход
-------------------

Это представление пишется по той же схеме, что и представление ``register`` выше, также дописывается к существующему коду.

``flaskr/auth.py``

.. code-block:: python

    # existing code
    #...

    @bp.route('/login', methods=('GET', 'POST'))
    def login():
        if request.method == 'POST':
            username = request.form['username']
            password = request.form['password']
            db = get_db()
            error = None
            user = db.cursor().execute('SELECT * FROM users WHERE username = ?', (username,) ).fetchonemap()

            if user is None:
                error = 'Incorrect username.'
            elif not check_password_hash(user['password'], password):
                error = 'Incorrect password.'

            if error is None:
                session.clear()
                session['user_id'] = user['id']
                return redirect(url_for('index'))

            flash(error)

        return render_template('auth/login.html')

Есть несколько отличий от представления ``register``:

1. Для последующего использования в переменной сохраняется пользователь из таблицы ``users``. ``fetchonemap()`` возвращает одну строку из запроса. Результат имеет тип *словарь*. Если запрос не вернул ни одной строки, результат будет ``None``. 
2. ``check_password_hash()`` хеширует пароль из формы и сравнивает его с сохраненным. Если равны хеши паролей, то и пароли совпадают.
3. ``session`` переменная типа *словарь*, которая хранит данные между запросами. Если аутентификация прошла успешно, тогда ``id`` пользователя сохраняется для новой сессии. Данные сохраняются в *cookie* которые отправляются в браузер. Браузер пришлет их обратно в запросах. Flask безопасно подписывает данные, так что они не могут подменены.

Теперь, когда идентификатор пользователя сохранен в сессии, он будет доступен последующим запросам. В начале каждого запроса, если пользователь прошел аутентификацию, информация о нем должна быть загружена и стать доступной другим представлениям.

``flaskr/auth.py``

.. code-block:: python

    # existing code
    #...

    @bp.before_app_request
    def load_logged_in_user():
        user_id = session.get('user_id')

        if user_id is None:
            g.user = None
        else:
            g.user = get_db().cursor().execute('SELECT * FROM users WHERE id = ?', (user_id,)).fetchonemap()

``@bp.before_app_request()`` регистрирует функцию, которая выполняется перед функциями-представлениями, безотносительно какой URL был запрошен. ``load_logged_in_user()`` проверяет сохранен ли в ``session`` и если да, то читает информацию о нем из БД, сохраняя в ``g.user``, который существует на протяжении всей обработки запроса. В противном случае ``g.user`` будет установлен в ``None``.

Представления: выход
--------------------

Для выхода необходимо удалить ``user_id`` из ``session``. Тогда ``load_logged_in_user`` не сможет загружать пользователя в последующих запросах.

``flaskr/auth.py``

.. code-block:: python

    # existing code
    #...

    @bp.route('/logout')
    def logout():
        session.clear()
        return redirect(url_for('index'))

Требования входа в других представлениях
----------------------------------------

Создание, редактирование и удаление постов блога требует чтобы пользователь был авторизован. *Декоратор* используется для проверки каждого представления, к которому он примеряется.

``flaskr/auth.py``

.. code-block:: python

    # existing code
    #...

    def login_required(view):
        @functools.wraps(view)
        def wrapped_view(**kwargs):
            if g.user is None:
                return redirect(url_for('auth.login'))

            return view(**kwargs)

        return wrapped_view

Этот *декоратор* возвращает новую функцию-представление, которая *оборачивает* оригинальное представление. Новая функция проверяет загружен ли пользователь и если нет перенаправляет на страницу входа. Если пользователь загружен, то вызывается оригинальная функция-представление и выполнение продолжается как обычно. Этот *декоратор* будет использован далее, при написании представлений блога.

Шаблоны (Templates)
===================

Функции-представления написаны, но если мы перейдем по любому адресу, мы получим ошибку ``TemplateNotFound``. Это потому что используется функция ``render_template``, но для нее еще не написаны шаблоны. Шаблоны располагаются в каталоге ``templates`` внутри пакета ``flaskr``.

Шаблоны - это файлы, содержащие статические данные вместе с *подстановками* (placeholders) для динамических данных.Шаблоны рендерятся с конкретными данными для получения финального документа. Flask использует библиотеку шаблонов Jinja_.

Jinja похожа на Python. Специальные разделители используются для отличия синтаксиса Jinja от статических данных шаблона. Содержимое между ``{{`` и ``}}`` является выражением, результат которого будет выведен в финальный документ. ``{%`` и ``%}`` выделяют управлящие потоки типа ``if`` и ``for``. В отличие от Python блоки выделяются в начале и конце тегами.

Базовый шаблон
--------------

Каждая страница приложения будет иметь одинаковую структуру с различным заполнением. Вместо написания HTML структуры для каждого шаблона, каждый шаблон будет *расширять* базовый и перезаписывать отдельные секции.

``flaskr/templates/base.html``

.. code-block:: html

    <!doctype html>
    <title>{% block title %}{% endblock %} - Flaskr</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <nav>
    <h1>Flaskr</h1>
    <ul>
        {% if g.user %}
        <li><span>{{ g.user['username'] }}</span>
        <li><a href="{{ url_for('auth.logout') }}">Log Out</a>
        {% else %}
        <li><a href="{{ url_for('auth.register') }}">Register</a>
        <li><a href="{{ url_for('auth.login') }}">Log In</a>
        {% endif %}
    </ul>
    </nav>
    <section class="content">
    <header>
        {% block header %}{% endblock %}
    </header>
    {% for message in get_flashed_messages() %}
        <div class="flash">{{ message }}</div>
    {% endfor %}
    {% block content %}{% endblock %}
    </section>

``g`` автоматически доступна в шаблонах. В зависимости от ``g.user`` показываются либо имя пользователя и ссылка на выход, либо ссылка на регистрацию и ссылка на вход. ``url_for()`` тоже автоматически доступна и используется для генерации URL адресов на представления вместо ручного их написания.

После заголовка страницы и перед основным содержимым шаблон в цикле проходит по всем сообщениям, возвращаемым ``get_flashed_messaged()``. Функция ``flask()`` использовалась в представлениям для показа сообщений об ошибках и этот код их показывает.

Здесь определены 3 блока, которые будут переписаны другими шаблонами:

1. ``{% block title %}`` будет изменять отображаемый в браузере заголовок вкладки.
2. ``{% block header %}`` подобен ``title``, но будет изменять заголовок, отображаемый на странице.
3. ``{% block content %}`` будет отображать содержимое каждой страницы, например форму входа или пост блога.

Этот базовый шаблон размещается прямо в каталоге ``templates``. Для организованного хранения остальных, разместим их в каталогах по названиям экскизов.

Регистрация
-----------

``flaskr/templates/auth/register.html``

.. code-block:: html

    {% extends 'base.html' %}

    {% block header %}
    <h1>{% block title %}Register{% endblock %}</h1>
    {% endblock %}

    {% block content %}
    <form method="post">
        <label for="username">Username</label>
        <input name="username" id="username" required>
        <label for="password">Password</label>
        <input type="password" name="password" id="password" required>
        <input type="submit" value="Register">
    </form>
    {% endblock %}

``{% extends 'base.html' %}`` говорит Jinja что этот шаблон заменяет блоки базового шаблона. Все отображаемое содержимое должно содержаться внутри блока ``{% block %}``, которые перезаписывают блоки базового шаблона.

Здесь используется полезный паттерн размещения блока ``{% block title %}`` внутри блока ``{% block header %}``. Это установит ``title block`` и затем итоговое значение будет использовано в ``header block``. Так что оба заголовка используют одно и тоже название, без необходимости писать его дважды.

Вход
----

Этот шаблон аналогичен шаблону регистрации, за исключением заголовка и кнопки submit.

``flaskr/templates/auth/login.html``

.. code-block:: html

    {% extends 'base.html' %}

    {% block header %}
    <h1>{% block title %}Log In{% endblock %}</h1>
    {% endblock %}

    {% block content %}
    <form method="post">
        <label for="username">Username</label>
        <input name="username" id="username" required>
        <label for="password">Password</label>
        <input type="password" name="password" id="password" required>
        <input type="submit" value="Log In">
    </form>
    {% endblock %}

Регистрация
-----------

Теперь когда все шаблоны аутентификации написаны, можно зарегистрировать пользователя. Убедитесь что сервер запущен и перейдите по адресу http://127.0.0.1:5000/auth/register.

Попробуйте нажать на кнопку "Register" без заполнения формы и посмотрите на ошибки.

При успешном заполнении имени пользователя и пароля, вы будете перенаправлены на страницу логина. Попробуйте ввести некорректный логин или пароль.

Если вы войдете в систему, то все равно должны увидеть ошибку, т.к. еще нет представления для ``index``.

Статические файлы
=================

Представления аутентификации и шаблоны выглядят слишком примитивными. Чтобы немного стилизовать HTML добавим CSS. Таблицы стилей являются статическими файлами.

Flask автоматически добавляет представление ``static``, которое принимает путь относительно каталога ``flaskr/static`` и обрабатывает его. Базовый шаблон ``base.html`` уже имеет статическую ссылку на файл ``style.css``:

.. code-block:: html

    {{ url_for('static', filename='style.css') }}

Кроме CSS могут быть и другие типы статических файлов: JavaScript, изображения и т.п. Все они располагаются в каталог ``flaskr/static`` и для ссылки используется ``url_for('static', filename='...')``.

Здесь мы не делаем упор на изучение CSS, так что просто скопируем содержимое файла в:

``flaskr/static/style.css``

.. code-block:: css

    html { font-family: sans-serif; background: #eee; padding: 1rem; }
    body { max-width: 960px; margin: 0 auto; background: white; }
    h1 { font-family: serif; color: #377ba8; margin: 1rem 0; }
    a { color: #377ba8; }
    hr { border: none; border-top: 1px solid lightgray; }
    nav { background: lightgray; display: flex; align-items: center; padding: 0 0.5rem; }
    nav h1 { flex: auto; margin: 0; }
    nav h1 a { text-decoration: none; padding: 0.25rem 0.5rem; }
    nav ul  { display: flex; list-style: none; margin: 0; padding: 0; }
    nav ul li a, nav ul li span, header .action { display: block; padding: 0.5rem; }
    .content { padding: 0 1rem 1rem; }
    .content > header { border-bottom: 1px solid lightgray; display: flex; align-items: flex-end; }
    .content > header h1 { flex: auto; margin: 1rem 0 0.25rem 0; }
    .flash { margin: 1em 0; padding: 1em; background: #cae6f6; border: 1px solid #377ba8; }
    .post > header { display: flex; align-items: flex-end; font-size: 0.85em; }
    .post > header > div:first-of-type { flex: auto; }
    .post > header h1 { font-size: 1.5em; margin-bottom: 0; }
    .post .about { color: slategray; font-style: italic; }
    .post .body { white-space: pre-line; }
    .content:last-child { margin-bottom: 0; }
    .content form { margin: 1em 0; display: flex; flex-direction: column; }
    .content label { font-weight: bold; margin-bottom: 0.5em; }
    .content input, .content textarea { margin-bottom: 1em; }
    .content textarea { min-height: 12em; resize: vertical; }
    input.danger { color: #cc2f2e; }
    input[type=submit] { align-self: start; min-width: 10em; }

Откройте ссылку http://127.0.0.1:5000/auth/login и сейчас страница должна выглядеть как на картинке.

.. image:: flaskr_login.png

Больше о CSS можно узнать из документации https://developer.mozilla.org/docs/Web/CSS

Эскиз блога
===========

Будут использованы теже подходы, что и для написания эскиза аутентификации. Блог показывает список всех постов, позволяя авторизованным пользователям создавать посты, а авторам менять и удалять посты.

Во время реализации каждого представления, оставляйте сервер запущенным (в режиме разработчика). После сохранения изменений, переходите по соответствующему URL в браузере и тестируйте его.

Объявим эскиз и добавим его в *фабрику приложения*.

``flaskr/blog.py``

.. code-block:: python

    from flask import (
        Blueprint, flash, g, redirect, render_template, request, url_for
    )
    from werkzeug.exceptions import abort

    from flaskr.auth import login_required
    from flaskr.db import get_db

    bp = Blueprint('blog', __name__)

Импортируйте и зарегистрируйте эскиз.

``flaskr/__init__.py``

.. code-block:: python

    def create_app():
        app = ...
        # existing code omitted

        from . import blog
        app.register_blueprint(blog.bp)
        app.add_url_rule('/', endpoint='index')

        return app

В отличие от эскиза аутентификации, эскиз блога не имеет ``url_prefix``. Таким образом представление ``index`` будет размещаться в корне ``/``, представление ``create`` по адресу ``/create`` и т.д. Блог - основная функция Flaskr и логично сделать ``index`` основным представлением.

Однако, точка входа для ``index`` будет ``blog.index``. Некоторые представления аутентификации ссылаются на простой ``index``. ``app.add_url_rule()`` связывает точку входа ``index`` с адресом ``/``, так что ``url_for('index')`` и ``url_for('blog.index')`` будут работать одинаково, генерируя одинаковый адрес URL.

В других приложениях вам может потребоваться дать другой ``url_prefix`` для эскиза и определить другой ``index`` для приложения, подобный представлению ``hello``.

Представление Index
-------------------

Индекс будет показывать все посты, начиная с последних. SQL запрос использует JOIN для получения аутентификационной информации из таблицы ``users``.

``flaskr/blog.py``

.. code-block:: python

    @bp.route('/')
    def index():
        db = get_db()
        posts = db.cursor().execute(
            'SELECT p.id, title, body, created, author_id, username'
            ' FROM posts p JOIN users u ON p.author_id = u.id'
            ' ORDER BY created DESC'
        ).fetchallmap()
        return render_template('blog/index.html', posts=posts)

``flaskr/templates/blog/index.html``

.. code-block:: html
    
    {% extends 'base.html' %}

    {% block header %}
    <h1>{% block title %}Posts{% endblock %}</h1>
    {% if g.user %}
        <a class="action" href="{{ url_for('blog.create') }}">New</a>
    {% endif %}
    {% endblock %}

    {% block content %}
    {% for post in posts %}
        <article class="post">
        <header>
            <div>
            <h1>{{ post['title'] }}</h1>
            <div class="about">by {{ post['username'] }} on {{ post['created'].strftime('%Y-%m-%d') }}</div>
            </div>
            {% if g.user['id'] == post['author_id'] %}
            <a class="action" href="{{ url_for('blog.update', id=post['id']) }}">Edit</a>
            {% endif %}
        </header>
        <p class="body">{{ post['body'] }}</p>
        </article>
        {% if not loop.last %}
        <hr>
        {% endif %}
    {% endfor %}
    {% endblock %}    

Когда пользователь авторизован, блок ``header`` добавляет ссылку на представление ``create``. Когда пользователь автор поста, он увидит ссылку "Edit" связанную с представлением ``update`` для поста. ``loop.last`` - это специальная переменная, доступная внутри циклов Jinja. Она используется для исключения печати разделительной линии для последнего поста.

Представление Create
--------------------

Представление ``create`` походе на представление ``register``. Либо отображается форма для заполнения данных, либо введенные данные проверяются и пост добавляется в базу данных или показывается ошибка.

*Декоратор* ``login_required``, который мы написали ранее, будет использован для представлений блога. Пользователь должен быть авторизован, чтобы открывать эти представления, иначе он будет перенаправлен на страницу входа.

``flaskr/blog.py``

.. code-block:: python

    @bp.route('/create', methods=('GET', 'POST'))
    @login_required
    def create():
        if request.method == 'POST':
            title = request.form['title']
            body = request.form['body']
            error = None

            if not title:
                error = 'Title is required.'

            if error is not None:
                flash(error)
            else:
                db = get_db()
                db.cursor().execute(
                    'INSERT INTO posts (title, body, author_id)'
                    ' VALUES (?, ?, ?)',
                    (title, body, g.user['id'])
                )
                db.commit()
                return redirect(url_for('blog.index'))

        return render_template('blog/create.html')

``flaskr/templates/blog/create.html``

.. code-block:: html

    {% extends 'base.html' %}

    {% block header %}
    <h1>{% block title %}New Post{% endblock %}</h1>
    {% endblock %}

    {% block content %}
    <form method="post">
        <label for="title">Title</label>
        <input name="title" id="title" value="{{ request.form['title'] }}" required>
        <label for="body">Body</label>
        <textarea name="body" id="body">{{ request.form['body'] }}</textarea>
        <input type="submit" value="Save">
    </form>
    {% endblock %}

Представление Update
--------------------

Представлениям ``update`` и ``delete`` необходимо извлечь пост по его идентификатору и сравнить автора с зарегистрированным пользователем. Чтобы избежать дублирования кода, напишем функцию получения поста и в дальнейшем используем ее в обоих представлениях.

``flaskr/blog.py``

.. code-block:: python

    def get_post(id, check_author=True):
        post = get_db().cursor().execute(
            'SELECT p.id, title, body, created, author_id, username'
            ' FROM posts p JOIN users u ON p.author_id = u.id'
            ' WHERE p.id = ?',
            (id,)
        ).fetchonemap()

        if post is None:
            abort(404, f"Post id {id} doesn't exist.")

        if check_author and post['author_id'] != g.user['id']:
            abort(403)

        return post

``abort()`` выбрасывает специальное исключение, которое возвращает код статуса HTTP. Она принимает текст ошибки. Код 404 означает что страница не найдена (Not Found), а код 403 означает что доступ к странице запрещен (Forbidden).

Аргумент ``check_author`` может быть полезен в будущем.

Код представления ``update``.

``flaskr/blog.py``

.. code-block:: python

    @bp.route('/<int:id>/update', methods=('GET', 'POST'))
    @login_required
    def update(id):
        post = get_post(id)

        if request.method == 'POST':
            title = request.form['title']
            body = request.form['body']
            error = None

            if not title:
                error = 'Title is required.'

            if error is not None:
                flash(error)
            else:
                db = get_db()
                db.cursor().execute(
                    'UPDATE posts SET title = ?, body = ?'
                    ' WHERE id = ?',
                    (title, body, id)
                )
                db.commit()
                return redirect(url_for('blog.index'))

        return render_template('blog/update.html', post=post)

В отличие от представлений, что мы писали до этого, ``update`` принимает аргумент, ``id``. На это указывает ``<int:id>``. Реальный адрес будет выглядеть, например, ``/1/update``. Flask извлечет ``1`` из адреса, убедиться что это значение типа ``int`` и передаст его как аргумент функции. Если не указать ``int:`` то аргумент будет ``string``. Чтобы генерировать адреса для страницы обновления, ``url_for()`` необходимо передать ``id`` для заполнения: ``url_for('blog.update', id=post['id'])``.

Представления ``create`` и ``update`` похожи друг на друга. Основное отличие в том, что ``update`` использует объект ``post`` и запрос ``UPDATE`` вместо ``INSERT``. Теоретически можно придумать единый шаблон для этих двух представлений, но для наших целей мы не будем усложнять.

``flaskr/templates/blog/update.html``

.. code-block:: html

    {% extends 'base.html' %}

    {% block header %}
    <h1>{% block title %}Edit "{{ post['title'] }}"{% endblock %}</h1>
    {% endblock %}

    {% block content %}
    <form method="post">
        <label for="title">Title</label>
        <input name="title" id="title"
        value="{{ request.form['title'] or post['title'] }}" required>
        <label for="body">Body</label>
        <textarea name="body" id="body">{{ request.form['body'] or post['body'] }}</textarea>
        <input type="submit" value="Save">
    </form>
    <hr>
    <form action="{{ url_for('blog.delete', id=post['id']) }}" method="post">
        <input class="danger" type="submit" value="Delete" onclick="return confirm('Are you sure?');">
    </form>
    {% endblock %}

Шаблон имеет две формы. Первая отправляет отредактированные данные на текущую страницу ``/<id>/update``. Вторая форма содержит единственную кнопку и определяет атрибут ``action`` ссылающийся на представление ``delete``. Кнопка использует JavaScript для запроса подтверждения действия.

Паттерн ``{{ request.form['title'] or post['title'] }}`` используется в зависимости от данных, которые надо показать в форме. Когда форма не отправлена, показываются данные оригинального поста, но если в форму переданы некорректные данные, необходимо показать это чтобы пользователь исправил ошибку. Для этого используется ``request.form``. ``request`` - это еще одна переменная, автоматически доступная в шаблонах.

Представление Delete
--------------------

Это представление не имеет своего шаблона, а кнопка удаления является частью шаблона ``update.html``. Таким образом, достаточно написать только функцию представление и обработать только метод ``POST``, а потом перенаправить на представление ``index``.

``flaskr/blog.py``

.. code-block:: python

    @bp.route('/<int:id>/delete', methods=('POST',))
    @login_required
    def delete(id):
        get_post(id)
        db = get_db()
        db.cursor().execute('DELETE FROM posts WHERE id = ?', (id,))
        db.commit()
        return redirect(url_for('blog.index'))

.. important:: На этом разработка первого приложения завершена. Необходимо его тщательно протестировать.

Для изучения того, как тестировать и распространять подобные приложения рекомендуем изучить материалы по Flask - https://flask.palletsprojects.com/en/2.0.x/tutorial

