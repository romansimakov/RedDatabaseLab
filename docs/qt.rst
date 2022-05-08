.. _qt-tutorial:

**********************************
Разработка приложения на Qt
**********************************

В данной лабораторной работе рассматривается пример создания простейшего приложения в среде QtCreator, работающего с БД.

Структура БД
============

Для разработки приложения необходимо создать БД следующей структуры:

.. image:: img/qt/db.jpeg

Создание проекта и интерфейса
=============================

Создаем новый проект

.. image:: img/qt/new_project.png

В качестве шаблона выбираем “Приложение/Приложение Qt Widgets”

.. image:: img/qt/project_type.png

Далее выбираем все опции по умолчанию, указав имя проекта и его расположение.
В дереве объектов дважды кликаем по форме и приступаем к размещению элементов на форме. Для этого используются ``Push Button``, меню и ``QTreeView``
В меню создаем ряд пунктов для открытия, закрытия БД, выхода и показа справки.

.. image:: img/qt/menu.png

Остальные компоненты размещаем как на рисунке.

.. image:: img/qt/form.png

Написание кода
==============

Перед написанием добавим в файл проекта ``.pro`` модуль sql

.. code-block::

    QT += core gui sql

Обработчик нажатия пунктов меню пишется следующим образом:
1. Выделяем нужный пункт меню на форме.
2. Внизу также будет список пунктов. На нужном кликаем правой кнопкой мыши и выбираем “Переход к слоту”. Далее выбираем “triggered“.
3. Открывается редактор кода, в котором мы увидим заготовку метода, который будет вызван при нажатии на пункт меню.

.. image:: img/qt/handler.png

Добавляем код соединения с БД, т.е. обработчик пункта меню “Файл/Открыть”

.. code-block:: c++

    // Connection is already open
    if (abonentModel)  // Field of MainWindow
        return;
    
    QSqlDatabase db = QSqlDatabase::addDatabase("QIBASE", "phones");
    db.setHostName("localhost");
    db.setDatabaseName(QFileDialog::getOpenFileName(this, "Open Database", "/var/rdb", "Database files (*.fdb);;All files (*)"));
     
    const bool ok = db.open("SYSDBA", "masterkey");
     
    if (!ok)
    {
        ShowMessage("Ошибка подключения");
        return;
    }
    
    abonentModel = new QSqlTableModel(this, db);
    abonentModel->setTable("ABONENT");
    abonentModel->setEditStrategy(QSqlTableModel::OnManualSubmit);
    abonentModel->select();
    ui->abonentView->setModel(abonentModel);

Код для закрытия БД

.. code-block:: c++

    delete abonentModel;
    abonentModel = nullptr;
    QSqlDatabase::database("phones").close();

Можно протестировать. БД должна открываться и данные должны появлятся в таблице.
Функции ShowMessage нужна для вывода сообщений на экран. Она определяется следующим образом.

.. code-block:: c++

    void ShowMessage(const QString text)
    {
        QMessageBox msg;
        msg.setText(text);
        msg.exec();
    }

Для написания обработчика нажания кнопки кликните на ней на форме правой кнопкой мыши и в “Перейти к слоту” выберете ``whenClicked``.
Ниже все обработчики кнопок. Говорящие названия обработчиков описывают действия.

.. code-block:: c++

    void MainWindow::on_btnAdd_clicked()
    {
        if (!abonentModel->insertRow(0))
            ShowMessage(abonentModel->lastError().text());
    }
    
    void MainWindow::on_btnSave_clicked()
    {
        if (!abonentModel->submitAll())
            ShowMessage(abonentModel->lastError().text());
    }
    
    void MainWindow::on_btnDel_clicked()
    {
        QModelIndex i = ui->abonentView->selectionModel()->currentIndex();
        if (!abonentModel->removeRow(i.row()))
            ShowMessage(abonentModel->lastError().text());
    }
    
    void MainWindow::on_btnCancel_clicked()
    {
        abonentModel->revertAll();
    }
    
    void MainWindow::on_btnRefresh_clicked()
    {
        abonentModel->select();
    }

.. important:: На этом разработка первого приложения завершена. Необходимо его тщательно протестировать.
