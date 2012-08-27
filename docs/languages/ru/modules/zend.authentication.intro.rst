.. EN-Revision: none
.. _zend.authentication.introduction:

Введение
========

``Zend\Authentication`` предоставляет *API* для аутентификации и включает в себя
конкретные адаптеры для общих случаев применения.

Задачей ``Zend\Authentication`` является **аутентификация**, но не
**авторизация**. Аутентификацию в общих чертах можно
характеризовать как процесс определения, действительно ли
сущность является тем, чем она претендует быть (то есть
идентификация), на основании некоторого набора учетных данных.
Авторизация же - процесс определения того, можно ли
предоставить сущности доступ или разрешить выполнить
действие, и это полностью вне сферы компетенции ``Zend\Authentication``.
Дополнительную информацию об авторизации и контроле доступа в
Zend Framework смотрите в :ref:`Zend\Permissions\Acl  <zend.acl>`.

.. note::

  Класса ``Zend\Authentication\Authentication`` не существует ,
  аутентификацию обеспечивает ``Zend\Authentication\AuthenticationService``. 
  Этот класс использует основные адаптеры аутентификации и постоянные хранилища.

.. _zend.authentication.introduction.adapters:

Адаптеры
--------

Адаптер ``Zend\Authentication`` используется дла аутентификации посредством
определенного сервиса, такого как *LDAP*, *СУРБД* или файлового
хранилища. Адаптеры могут значительно различаться, но
некоторые основные черты характерны для всех. Например, все
адаптеры ``Zend\Authentication`` принимают учетные данные, выполненяют запрос
к аутентификационному сервису и возвращают результат.

Каждый адаптер ``Zend\Authentication`` реализует ``Zend\Authentication\Adapter\AdapterInterface``. Этот
интерфейс определяет лишь один метод: ``authenticate()``, который
должен быть реализован   для выполнения аутентификационного
запроса. Адаптер должен быть настроен до вызова ``authenticate()``,
настройка включает в себя установку учетных данных(например,
логин и пароль) и определение специфичных значений, таких как
настройки подключения к базе данных для адаптера таблиц БД.

Это пример адаптера, требующего установки логина и пароля для
аутентификации. Прочие детали, например каким образом
аутентификационный сервис запрашивается, были опущены для
кратости:

.. code-block:: php
   :linenos:
   use Zend\Authentication\Adapter\AdapterInterface;
  
   class My\Auth\Adapter implements AdapterInterface
   {
       /**
        * Устанавливает логин и пароль для аутентификации
        *
        * @return void
        */
       public function __construct($username, $password)
       {
           // ...
       }

       /**
        * Performs an authentication attempt
        *
        * @return \Zend\Authentication\Result
        * @throws \Zend\Authentication\Adapter\Exception\ExceptionInterface
        *               If authentication cannot be performed
        */
       public function authenticate()
       {
           // ...
       }
   }

Как указано в докблоке, ``authenticate()`` должен вернуть экземпляр
``Zend\Authentication\Result`` (или унаследованный от него). Если по какой либо
причине выполнение аутентификации невозможно, ``authenticate()``
должен бросить исключение, происходящее от ``Zend\Authentication\Adapter\Exception\ExceptionInterface``.

.. _zend.authentication.introduction.results:

Результат аутентификации
------------------------

Метод ``authenticate()`` адаптера ``Zend\Authentication`` возвращает экзмепляр
``Zend\Authentication\Result`` для представления результата попытки
аутентификации. Объект ``Zend\Authentication\Result`` заполняется адаптером при
создании, и следующие четыре метода представляют его базовый
набор операций:

- ``isValid()``- возвращает ``TRUE`` только в случае успешной попытки
  аутентификации.

- ``getCode()``- возвращает значение одной из констант ``Zend\Authentication\Result`` для
  обозначения успешности попытки или типа возникшей ошибки.
  Это может быть использовано в ситуации, когда разработчик
  хочет различать результаты попыток аутентификации. К
  примеру, он может вести детальную статистку всех попыток,
  либо выводить нестандартные сообщения для удобства
  пользователей. Но при этом стоит учитывать риски
  предоставления пользователям подробных сведений вместо
  стандартных сообщений о неудаче. Подробнее о возвращаемых
  значениях смотрите ниже.

- ``getIdentity()``- возвращает идентификатор, полученный в результе
  аутентификации.

- ``getMessages()``- возвращает массив сообщений о ошибках, возникших в
  процессе попытки аутентификации.

Разработчик может пожелать выполнять различные действия на
основании результата аутентификации. Например, он может найти
полезным блокировку аккаунтов после нескольких вводов
неправильного пароля, пометку IP после слишком многих попыток
аутентификации с несуществующими данными или вывод
настраиваемых сообщений о результате аутентификации.
Существуют следующие коды результата:

.. code-block:: php
   :linenos:
   use Zend\Authentication\Result;   

   Result::SUCCESS
   Result::FAILURE
   Result::FAILURE_IDENTITY_NOT_FOUND
   Result::FAILURE_IDENTITY_AMBIGUOUS
   Result::FAILURE_CREDENTIAL_INVALID
   Result::FAILURE_UNCATEGORIZED

Этот пример показывает, как разработчик может различным
образом обработать результат аутентификации, используя
значение кода:

.. code-block:: php
   :linenos:

   // в AuthController / loginAction
   $result = $this->_auth->authenticate($adapter);

   switch ($result->getCode()) {

       case Result::FAILURE_IDENTITY_NOT_FOUND:
           /** Выполнить действия при несуществующем идентификаторе **/
           break;

       case Result::FAILURE_CREDENTIAL_INVALID:
           /** Выполнить действия при некорректных учетных данных **/
           break;

       case Result::SUCCESS:
           /** Выполнить действия при успешной аутентификации **/
           break;

       default:
           /** Выполнить действия для остальных ошибок **/
           break;
   }

.. _zend.authentication.introduction.persistence:

Постоянное хранение идентификатора пользователя
-----------------------------------------------

Аутентификация запроса, содержащего учетные данные, важна
сама по себе, но также важно поддерживать сохранение
идентификатора без необходимости передачи учетных данных с
каждым запросом.

Протокол *HTTP* не имеет состояний, однако были разработаны такие
технологии как куки(cookies) и сессии для поддержки состояния на
стороне сервера между несколькими запросами к веб приложению.

.. _zend.authentication.introduction.persistence.default:

Сохранение идентификатора в сессии PHP, по умолчанию
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

По умолчанию, ``Zend\Authentication`` обеспечивает постоянное хранение
идентификатора полученного в результате успешной попытки
аутентификации в *PHP* сессии.

При успешной попытке, ``Zend\Authentication\AuthenticationService::authenticate()``
сохраняет идентификатор в постоянном хранилище. Если не настроено по другому, 
``Zend\Authentication\AuthenticationService``использует класс хранилища 
``Zend\Authentication\Storage\Session``, который в свою
очередь использует :ref:`Zend\Session <zend.session>`. Вместо него может быть
использован пользовательский класс, для этого нужно передать
``Zend\Authentication\AuthenticationService::setStorage()`` объект,
реализующий ``Zend\Authentication\Storage\StorageInterface``.

.. note::

   Если автоматическое сохранение идентификатора не подходит в
   каком-либо конкретном случае, тогда разработчику следует
   отказаться от использования класса ``Zend\Authentication\AuthenticationService`` и использовать
   адаптер напрямую.

.. _zend.authentication.introduction.persistence.default.example:

.. rubric:: Изменение пространства имен в сессии

``Zend_Auth_Storage_Session`` использует пространство имен '``Zend_Auth``'. Оно
может быть переопределено передачей другого значения
конструктору ``Zend_Auth_Storage_Session``, которое будет дальше передано
конструктору ``Zend_Session_Namespace``. Это нужно сделать до того, как
будет произведена попытка аутентификации, так как
``Zend_Auth::authenticate()`` выполняет автоматическое сохранение
идентификатора.

.. code-block:: php
   :linenos:

   // Получение синглтон экземпляра Zend_Auth
   $auth = Zend_Auth::getInstance();

   // Установка 'someNamespace' вместо 'Zend_Auth'
   $auth->setStorage(new Zend_Auth_Storage_Session('someNamespace'));

   /**
    * @todo подготовка адаптера, $authAdapter
    */

   // Аутентификация, сохранение результата, и хранение идентификатора
   // при успехе.
   $result = $auth->authenticate($authAdapter);

.. _zend.authentication.introduction.persistence.custom:

Реализация пользовательского хранилища
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Иногда разработчику может понадобиться использовать иной
механизм хранения идентификаторов, нежели предоставляется в
``Zend_Auth_Storage_Session``. В том случае он может реализовать
``Zend_Auth_Storage_Interface`` и передать экземпляр методу ``Zend_Auth::setStorage()``.

.. _zend.authentication.introduction.persistence.custom.example:

.. rubric:: Использование пользовательского хранилища

Для того, чтобы использовать иной класс хранилища
пользовательских идентификаторов, нежели ``Zend_Auth_Storage_Session``,
разработчик реализует ``Zend_Auth_Storage_Interface``:

.. code-block:: php
   :linenos:

   class MyStorage implements Zend_Auth_Storage_Interface
   {
       /**
        * Возвращает  true, если хранилище пусто
        *
        * @throws Zend_Auth_Storage_Exception В случае если невозможно
        *                                     определить, пусто ли
        *                                     хранилище
        * @return boolean
        */
       public function isEmpty()
       {
           /**
            * @todo реализация
            */
       }

       /**
        * Возвращает содержимое хранилища
        *
        * Поведение неопределено, когда хранилище пусто.
        *
        * @throws Zend_Auth_Storage_Exception Если получение содержимого
        *                                     хранилища невозможно
        * @return mixed
        */
       public function read()
       {
           /**
            * @todo реализация
            */
       }

       /**
        * Записывает $contents в хранилище
        *
        * @param  mixed $contents
        * @throws Zend_Auth_Storage_Exception Если запись содержимого в
        *                                     хранилище невозможна
        * @return void
        */
       public function write($contents)
       {
           /**
            * @todo реализация
            */
       }

       /**
        * Очищает содержмое хранилища
        *
        * @throws Zend_Auth_Storage_Exception Если очищение хранилища
        *                                     невозможно
        * @return void
        */
       public function clear()
       {
           /**
            * @todo реализация
            */
       }
   }

Для использования этого класса, ``Zend_Auth::setStorage()`` вызывается до
выполнения попытки авторизации:

.. code-block:: php
   :linenos:

   // Сказать Zend_Auth использовать пользовательский класс хранилища
   Zend_Auth::getInstance()->setStorage(new MyStorage());

   /**
    * @todo подготовка адаптера, $authAdapter
    */

   // Аутентификация, сохранение результата, и хранение идентификатора
   // при успехе.
   $result = Zend_Auth::getInstance()->authenticate($authAdapter);

.. _zend.authentication.introduction.using:

Использование
-------------

Существует два пути использования адаптеров ``Zend_Auth``:

. непрямое, через ``Zend_Auth::authenticate()``

. прямое, через метод адаптера ``authenticate()``

Следующий пример показывает, как использовать адаптер ``Zend_Auth``
через класс ``Zend_Auth``:

.. code-block:: php
   :linenos:

   // Получение синглтон экземпляра Zend_Auth
   $auth = Zend_Auth::getInstance();

   // Установка адаптера
   $authAdapter = new MyAuthAdapter($username, $password);

   // Попытка аутентификации, сохранение результата
   $result = $auth->authenticate($authAdapter);

   if (!$result->isValid()) {
       // Попытка неуспешна; вывести сообщения об ошибках
       foreach ($result->getMessages() as $message) {
           echo "$message\n";
       }
   } else {
       // Попытка успешна; идентификатор ($username) сохранен
       // в сессии
       // $result->getIdentity() === $auth->getIdentity()
       // $result->getIdentity() === $username
   }

После того как попытка аутентификации была произведена, как
показано в примере выше, теперь нужно только проверить,
существует ли аутентифицированный идентификатор:

.. code-block:: php
   :linenos:

   $auth = Zend_Auth::getInstance();
   if ($auth->hasIdentity()) {
       // Идентификатор существует; получить его
       $identity = $auth->getIdentity();
   }

Для удаления идентификатора из постоянного хранилища, просто
используйте метод ``clearIdentity()``. Обычно это используется для
реализации действия "Выйти":

.. code-block:: php
   :linenos:

   Zend_Auth::getInstance()->clearIdentity();

Когда автоматическое использование постоянного хранилища не
подходит, разработчик может просто обойти ``Zend_Auth`` и
использовать класс адаптера напрямую. Прямое использование
адаптера включает в себя настройку, подготовку объекта
адаптера и последующий вызов его метода, ``authenticate()``.
Специфичные для адаптера детали обсуждаются в документации
этого адаптера. Следующий пример напрямую использует
``MyAuthAdapter``:

.. code-block:: php
   :linenos:

   // Подготовка адаптера
   $authAdapter = new MyAuthAdapter($username, $password);

   // Попытка аутентификации, сохранение результата
   $result = $authAdapter->authenticate();

   if (!$result->isValid()) {
       // Попытка неуспешна; вывести сообщения об ошибках
       foreach ($result->getMessages() as $message) {
           echo "$message\n";
       }
   } else {
       // Попытка успешна;
       // $result->getIdentity() === $username
   }


