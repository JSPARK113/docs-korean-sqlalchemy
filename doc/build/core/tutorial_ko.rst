.. _sqlexpression_toplevel:

================================
SQL 표현식 언어 튜토리얼
================================

SQLAlchemy 표현식 언어는 관계형 데이터베이스 구조와 표현식을 파이썬 구문을 사용하여 나타내는 시스템을 제공한다.
이 구문들은 데이터베이스 백엔드 사이의 다양한 구현 차이를 추상화해서 제공하면서
아래에 있는 데이터베이스와 가능한한 유사하도록 모델링 되었다.
구문들은 백엔드 사이의 동일한 컨셉을 동일한 구조로
표현하려고 하는 반면 백엔드의 특정 서브셋에 유일하게 있는 유용한 컨셉들은
숨기지 않는다. 그러므로 표현식 언어는 백엔드에 중립적인 SQL 표현식을 쓰는 방식을
제공하지만 표현식이 백엔드 중립적이도록 강제하려고 하지는 않는다.

표현식 언어는 표현식 언어의 최상단에 빌드되어 있는 개별적 API인 객체 관계형 매퍼와는
다르다. :ref:`ormtutorial_toplevel`\ 에 설명되어있는 ORM은 그 자체로 응용된 표현식 언어의
예시가 되는 고수준의 추상적인 사용 패턴을 제공하지만 표현식 언어는 직접적으로 관계형 데이터베이스의
원시적인 구문을 표현하는 시스템을 제공한다.

ORM과 표현식 언어의 사용 패턴 사이에 겹치는 부분이 존재하는 반면, 유사점은 처음
나타나는 것보다 더 피상적이다. ORM은 하부의 스토리지 모델로부터 투명하게 유지되고
갱신되는 사용자 정의 `domain model <http://en.wikipedia.org/wiki/Domain_model>`_\ 의
관점에서 데이터와 구조에 접근한다. 표현식 언어는 데이터베이스에서 개별적으로 소비되는
메세지로 명시적으로 구성되는 리터럴 스키마와 SQL 표현식 관점에서 접근한다.

성공적인 어플리케이션은 비록 어플리케이션 개념을 개별적인 데이터베이스 메세지로 번역하고
개별적인 데이터베이스 결과 세트에서 어플리케이션 개념으로 번역하는 자체 시스템을
정의해야 할 필요가 있지만 표현식 언어만 사용해서 제작될 수 있다.
대신에 ORM으로 만들어진 어플리케이션은 고급 수준의 시나리오에서 종종 표현식 언어를
특정 데이터베이스와의 상호작용이 필요한 부분에 직접적으로 사용해야 한다.

아래의 튜토리얼은 doctest 형식이다.
``>>>`` 라인은 파이썬 커맨드 프롬프트에 입력할 수 있다는
것을 뜻하고 그 아래의 텍스트는 예상되는 반환 값을 의미한다.
이 튜토리얼을 이해하기 위해 미리 알아야 하는 것은 없다.

버전 확인
=============

**1.2 버전** 이상의 SQLAlchemy를 사용하고 있는지 확인한다::

    >>> import sqlalchemy
    >>> sqlalchemy.__version__ # doctest:+SKIP
    1.2.0

데이터베이스 연결
==============================

이 튜토리얼에서는 인메모리(in-memory) SQLite 데이터베이스를 사용한다.
이 방식을 쓰면 실제 데이터베이스를 정의할 필요없이 테스트를 할 수 있다.
:func:`~sqlalchemy.create_engine` 명령으로 연결한다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine('sqlite:///:memory:', echo=True)

``echo`` 플래그는 SQLAlchemy가 파이썬의 기본 ``logging`` 모듈을 통해서 로그를 기록하도록
설정하는 숏컷이다. 사용하도록 설정해 놓으면 생성되는 모든 SQL을 볼 수 있을 것이다. 이 튜토리얼을
진행하면서 출력이 적게 생성되기를 바란다면 ``False``\ 로 설정해놓으면 된다.
이 튜토리얼은 SQL을 팝업창 뒤에서 포매팅하기 때문에 전면에 나타나지는 않을 것이다;
"SQL" 링크를 클릭해서 무엇이 생성되었는지 확인하라.

:func:`.create_engine`\ 의 반환값은 :class:`.Engine`\ 의 인스턴스이며,
사용중인 :term:`DBAPI`\ 와 데이터베이스의 세부 사항을 다루는 :term:`dialect`\ 가
적용이 된 데이터베이스의 코어 인터페이스다. 이 경우 SQLite dialect가
파이썬 빌트인 ``sqlite3`` 모듈에 대한 지시사항을 해석할 것이다.

.. sidebar:: 레이지 컨넥팅(Lazy Connecting)

    :class:`.Engine`\ 는 :func:`.create_engine`\ 에서 반환되었을 때에는
    실제로 데이터베이스에 연결을 시도하지 않는다.
    실제 연결은 데이터베이스에 대한 작업을 수행하도록 최초로 요청받았을 때 이루어진다.

:meth:`.Engine.execute`\ 나 :meth:`.Engine.connect` 같은 메서드가 처음 호출되었을 때,
:class:`.Engine`\ 는 데이터베이스로 SQL을 전송하기 위한 실제 :term:`DBAPI` 연결을 구축한다.

.. seealso::

    :ref:`database_urls`\ 에는 여러 종류의 데이터베이스에 연결하는 :func:`.create_engine` 예시와
    함께 추가적인 정보가 있는 링크를 포함하고 있다.

Define and Create Tables
========================

SQL 표현식 언어는 대부분의 경우 테이블의 컬럼에 대해 표현식을 생성한다.
SQLAlchemy에서 컬럼은 대부분 :class:`~sqlalchemy.schema.Column`\ 이라고 하는
객체로 표현되며 모든 경우에 :class:`~sqlalchemy.schema.Column` 은
:class:`~sqlalchemy.schema.Table`\ 과 연결되어 있다.
:class:`~sqlalchemy.schema.Table` 객체와 연결된 자식 객체의 집합은 **데이터베이스 메타데이터**
라고 한다. 이 튜토리얼에서는 명시적으로 :class:`~sqlalchemy.schema.Table` 객체를 레이아웃할
것이지만 SA는 존재하는 데이터베이스에서 :class:`~sqlalchemy.schema.Table` 전체 객체 세트를 자동적으로
"임포트"할 수도 있다(이 프로세스를 **테이블 리플렉션**\ 이라고 한다).

우리는 일반 SQL의 CREATE TABLE 명령문과 유사한 :class:`~sqlalchemy.schema.Table` 구문을 사용해서
모든 테이블을 :class:`~sqlalchemy.schema.MetaData`\ 라고 불리는 카탈로그에 정의할 수 있다.
우리는 두 개의 테이블을 만들 것이며, 하나는 어플리케이션의 "users"를 나타낸다, 다른 하나는
"users" 테이블의 각 행에 대한 "email addresses"를 나타낸다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import Table, Column, Integer, String, MetaData, ForeignKey
    >>> metadata = MetaData()
    >>> users = Table('users', metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('name', String),
    ...     Column('fullname', String),
    ... )

    >>> addresses = Table('addresses', metadata,
    ...   Column('id', Integer, primary_key=True),
    ...   Column('user_id', None, ForeignKey('users.id')),
    ...   Column('email_address', String, nullable=False)
    ...  )

:class:`~sqlalchemy.schema.Table` 객체를 정의하거나, 존재하는 데이터베이스에서
자동적으로 객체를 생성하는 방법에 대한 모든 내용은 :ref:`metadata_toplevel`\ 에서
확인할 수 있다.

그 다음, :class:`~sqlalchemy.schema.MetaData`\ 에 우리가 실제로
SQLite 데이터베이스 안에 우리가 선택한 테이블을 생성하겠다는 지시를 전달하기 위해,
:func:`~sqlalchemy.schema.MetaData.create_all`\ 를 사용해서 우리의 데이터베이스를
가리키는 ``engine`` 인스턴스를 전달할 것이다. 그러면 테이블 생성 전에
먼저 테이블의 존재여부를 확인해서, 여러 번 호출을 해도 안전하다:

.. sourcecode:: pycon+sql

    {sql}>>> metadata.create_all(engine)
    SE...
    CREATE TABLE users (
        id INTEGER NOT NULL,
        name VARCHAR,
        fullname VARCHAR,
        PRIMARY KEY (id)
    )
    ()
    COMMIT
    CREATE TABLE addresses (
        id INTEGER NOT NULL,
        user_id INTEGER,
        email_address VARCHAR NOT NULL,
        PRIMARY KEY (id),
        FOREIGN KEY(user_id) REFERENCES users (id)
    )
    ()
    COMMIT

.. note::

    CREATE TABLE 신택스에 친숙한 사용자들은 VARCHAR 컬럼이 length 지정 없이 생성되었다는
    점을 알아차렸을 것이다; SQLite나 PostgreSQL에서 이는 유용한 데이터 타입이지만, 다른
    곳에서는 허용되지 않는다. 그래서 만약 이 튜토리얼을 다른 데이터베이스에서 실행햐고 있고
    SQLAlchemy를 사용해 CREATE TABLE를 발행하고 싶으면 "length"는 아래와 같이
    :class:`~sqlalchemy.types.String`\ 에 제공된다::

        Column('name', String(50))

    :class:`~sqlalchemy.types.String`\ 의 length 필드뿐만 아니라 유사한
    :class:`~sqlalchemy.types.Integer`, :class:`~sqlalchemy.types.Numeric`\ 의
    precision/scale 필드는 테이블을 생성할 때를 제외하고는 SQLAlchemy에 의해 참조되지 않는다.

    게다가, Firebird와 Oracle은 새로운 primary key 식별자를 생성하기 위해서 시퀀스를 요구하며
    SQLAlchemy는 지시 없이 그런 것들을 가정하거나 생성하지 않는다.
    그 부분에 대해서는 :class:`~sqlalchemy.schema.Sequence` 구문을 사용하면 된다::

        from sqlalchemy import Sequence
        Column('id', Integer, Sequence('user_id_seq'), primary_key=True)

    매우 간단한 전체 :class:`~sqlalchemy.schema.Table`\ 는 따라서::

        users = Table('users', metadata,
           Column('id', Integer, Sequence('user_id_seq'), primary_key=True),
           Column('name', String(50)),
           Column('fullname', String(50)),
           Column('password', String(12))
        )

    우리는 주로 파이썬 사용만을 위한 최소한의 구문과 더 엄격한 요구사항이 있는 특정
    백엔드 세트의 CREATE TABLE을 내보내기 위해 사용되는 구문의 차이점을 강조하기 위해
    위의 더 자세한 :class:`~.schema.Table` 구문을 분리해서 포함시켰다.

.. _coretutorial_insert_expressions_ko:

Insert Expressions
==================

만들어볼 첫 SQL 표현식은 INSERT 명령문을 나타내는
:class:`~sqlalchemy.sql.expression.Insert` 구문이다.
이것은 보통 타겟 테이블과 관련해 생성된다::

    >>> ins = users.insert()

이 구문이 만드는 SQL 샘플을 보려면 ``str()`` 함수를 사용해라::

    >>> str(ins)
    'INSERT INTO users (id, name, fullname) VALUES (:id, :name, :fullname)'

위에서 INSERT 명령문은 ``users`` 테이블의 모든 컬럼에 이름을 지정한다.
이는 명시적으로 INSERT의 VALUES 절을 설정하는 ``values()`` 메서드를 이용하면 제한할 수 있다::

    >>> ins = users.insert().values(name='jack', fullname='Jack Jones')
    >>> str(ins)
    'INSERT INTO users (name, fullname) VALUES (:name, :fullname)'

위에서 ``values`` 메서드가 VALUES 절을 두 개 컬럼으로 제한한 반면,
``values``\ 에 넣으려는 실제 데이터는 문자열로 렌더링 되지 않았다.
대신 명명된 바인드 파라미터(bind parameter)를 얻었다. 밝혀진대로, 데이터는
:class:`~sqlalchemy.sql.expression.Insert` 구문에 저장되지만,
일반적으로 명령문이 실제로 실행돼야 보여진다. 데이터는 리터럴 값으로 구성됐기 때문에 SQLAlchemy는
자동적으로 그것들에 대해 바인드 파라미터를 생성한다.
우선 이 명령어의 컴파일된 형태를 보는 걸 통해 데이터를 확인할 수 있다::

    >>> ins.compile().params  # doctest: +SKIP
    {'fullname': 'Jack Jones', 'name': 'jack'}

Executing
===================

:class:`~sqlalchemy.sql.expression.Insert`\ 의 흥미로운 부분은 그것을 실행하는 것이다.
이 튜토리얼은 일반적으로 SQL 구문을 실행하는 가장 명쾌한 방식에 중점을 둘 것이며,
나중에 그것을 할 수 있는 "숏컷"에 대해서 간단히 다룰 것이다.
생성한 ``engine`` 객체는 SQL을 데이터베이스에 보낼 수 있는 연결을 위한 장소다.
연결하려면 ``connect()`` 메서드를 사용한다::

    >>> conn = engine.connect()
    >>> conn
    <sqlalchemy.engine.base.Connection object at 0x...>

:class:`~sqlalchemy.engine.Connection` 객체는 적극적으로 확인된 DBAPI 연결 리소스를 나타낸다.
:class:`~sqlalchemy.sql.expression.Insert` 객체를 보내고 무슨 일이 일어나는지 보자:

.. sourcecode:: pycon+sql

    >>> result = conn.execute(ins)
    {opensql}INSERT INTO users (name, fullname) VALUES (?, ?)
    ('jack', 'Jack Jones')
    COMMIT

이제 INSERT 문이 데이터베이스에 보내졌다. "명명된" 바인드 파라미터 대신
위치상의 "물음표" 바인드 파라미터를 얻었다. 무엇 때문인가?
:class:`~sqlalchemy.engine.Connection`\ 은 실행될 때, SQLite **dialect**\ 를 사용해
명령문을 생성하기 때문이다. 우리가 ``str()`` 함수를 사용할 때, 명령문은 이 dialect를 알아차리지 못하고,
명명된 파라미터를 사용하는 기본값으로 돌아간다. 아래처럼 수동으로 이를 볼 수 있다:

.. sourcecode:: pycon+sql

    >>> ins.bind = engine
    >>> str(ins)
    'INSERT INTO users (name, fullname) VALUES (?, ?)'

``execute()``\ 를 호출할 때 얻은 ``result`` 변수는 어떤가?
SQLAlchemy :class:`~sqlalchemy.engine.Connection` 객체는 DBAPI 연결을 참조하므로,
:class:`~sqlalchemy.engine.ResultProxy` 객체로 알려진 ``result``\ 는 DBAPI 커서 객체와
유사하다. INSERT의 경우에, :attr:`.ResultProxy.inserted_primary_key`:\ 를
사용하는 명령문에서 생성된 프라이머리 키 값과 같은 중요한 정보를 얻을 수 있다:

.. sourcecode:: pycon+sql

    >>> result.inserted_primary_key
    [1]

``1`` 값은 SQLite에 의해 자동으로 생성되지만,
이는 단지 :class:`~sqlalchemy.sql.expression.Insert`\ 에서 ``id`` 컬럼을
명시하지 않았기 때문이다. 그렇지 않으면 명확한 값이 사용됐을 것이다.
어떤 경우에도, 데이터베이스에 따라 그것들을 생성하는 방법이 달라도,
SQLAlchemy는 새롭게 생성된 프라이머리 키 값을 얻을 수 있다.
각각의 데이터베이스의 :class:`~sqlalchemy.engine.interfaces.Dialect`\ 는
정확한 값(혹은 값; :attr:`.ResultProxy.inserted_primary_key`\ 는 복합 프라이머리 키를
지원하도록 리스트를 반환한다)을 정하는 데 필요한 특정 단계를 알고 있다.
여기에는 ``cursor.lastrowid``\ 를 사용하는 것부터, 데이터베이스에 특화된 함수 선택,
``INSERT..RETURNING`` 문법 사용까지 다양한 방법이 있다. 이것들은 모두 명확하게 발생한다.


.. _execute_multiple:

Executing Multiple Statements
=============================

위의 삽입 예제는 표현식 언어 구문의 다양한 동작을 보여주기 위해 의도적으로 작성됐다.
일반적인 경우, :class:`~sqlalchemy.sql.expression.Insert`\ 는 주로
:class:`~sqlalchemy.engine.Connection`\ 에 있는 ``execute()`` 메서드에 보내진 파라미터에
대해 컴파일 된다. :class:`~sqlalchemy.sql.expression.Insert`\ 와 함께
``values`` 키워드를 사용할 필요는 없다.
전역 :class:`~sqlalchemy.sql.expression.Insert`\ 문을 다시 만들어보고 "일반적인"
방법으로 사용해보자:

.. sourcecode:: pycon+sql

    >>> ins = users.insert()
    >>> conn.execute(ins, id=2, name='wendy', fullname='Wendy Williams')
    {opensql}INSERT INTO users (id, name, fullname) VALUES (?, ?, ?)
    (2, 'wendy', 'Wendy Williams')
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

위에서 우리는 ``execute()`` 메서드의 모든 3개 컬럼을 명시했기 때문에,
컴파일된 :class:`~.expression.Insert`\ 는 3개의 컬럼을 모두 포함했다.
:class:`~.expression.Insert` 명령문은 실행 시간에 우리가 지정한 파라미터를 기반으로 컴파일 된다.
더 적은 파라미터를 지정하면, :class:`~.expression.Insert`\ 는 VALUES 절에 더 적은 입력값을
갖게 된다.

DBAPI의 ``executemany()`` 메서드를 이용해 여러개를 삽입하려면, 여기서 해볼 이메일 주소를 추가하는
것처럼, 삽입하려는 파라미터의 구별되는 집합을 담은 딕셔너리의 리스트를 보낼 수 있다:

.. sourcecode:: pycon+sql

    >>> conn.execute(addresses.insert(), [
    ...    {'user_id': 1, 'email_address' : 'jack@yahoo.com'},
    ...    {'user_id': 1, 'email_address' : 'jack@msn.com'},
    ...    {'user_id': 2, 'email_address' : 'www@www.org'},
    ...    {'user_id': 2, 'email_address' : 'wendy@aol.com'},
    ... ])
    {opensql}INSERT INTO addresses (user_id, email_address) VALUES (?, ?)
    ((1, 'jack@yahoo.com'), (1, 'jack@msn.com'), (2, 'www@www.org'), (2, 'wendy@aol.com'))
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

위에서, SQLite가 각 ``addresses`` 행에 대해 프라이머리 키 식별자를 자동으로 생성하는 것에
다시 의존한다.

파라미터의 여러 집합이 실행될 때, 각 딕셔너리는 키의 **같은** 집합을 갖고 있어야 한다. 예를 들어,
같은 딕셔너리에 대해 다른 사람들보다 더 적은 키를 가질 수 없다.
왜냐하면 이것은 :class:`~sqlalchemy.sql.expression.Insert` 명령문은 리스트에 있는
**첫번째** 딕셔너리에 대해 컴파일 되기 때문이다. 그리고 모든 다음 인수 딕셔너리들은
그 명령문과 호환될 수 있다고 가정한다.

실행의 "executemany" 형식은 :func:`.insert`\ 와 :func:`.update`,
:func:`.delete` 구문에 대해 사용할 수 있다.


.. _coretutorial_selecting:

Selecting
====================

테스트 데이터베이스가 몇 개의 데이터를 갖게 하려고 insert로 시작했다.
데이터의 더 흥미로운 점은 select 하는 것이다! 나중에 UPDATE와 DELETE 문을 다룰 것이다.
SELECT 문을 생성하는 데 사용되는 기본 구문은 :func:`.select` 함수다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import select
    >>> s = select([users])
    >>> result = conn.execute(s)
    {opensql}SELECT users.id, users.name, users.fullname
    FROM users
    ()

위에서 기본적인 :func:`.select` 호출을 실행하고, select의 COLUMNS 절에 ``users`` 테이블을
놓은 다음 실행한다. SQlAlchemy는 ``users`` 테이블을 각 컬럼의 집합으로 확장하고, FROM 절을
생성했다. 반환 된 결과는 :class:`~sqlalchemy.engine.ResultProxy` 객체이며, 이 객체는
:func:`~sqlalchemy.engine.ResultProxy.fetchone`\ 와
:func:`~sqlalchemy.engine.ResultProxy.fetchall` 같은 메서드를 포함하는 DBAPI 커서처럼
동작한다. 행을 가져오는 가장 쉬운 방법은 반복하는 것이다:

.. sourcecode:: pycon+sql

    >>> for row in result:
    ...     print(row)
    (1, u'jack', u'Jack Jones')
    (2, u'wendy', u'Wendy Williams')

위에서 각 행을 인쇄하면 간단한 튜플과 같은 결과를 만든다는 것을 확인했다. 각 행의 데이터에 접근하는
더 많은 옵션이 있다. 가장 일반적인 방법 중 하나는 컬럼의 문자열 이름을 이용한 딕셔너리 접근이다:

.. sourcecode:: pycon+sql

    {sql}>>> result = conn.execute(s)
    SELECT users.id, users.name, users.fullname
    FROM users
    ()

    {stop}>>> row = result.fetchone()
    >>> print("name:", row['name'], "; fullname:", row['fullname'])
    name: jack ; fullname: Jack Jones

정수 인덱스도 제대로 작동한다:

.. sourcecode:: pycon+sql

    >>> row = result.fetchone()
    >>> print("name:", row[1], "; fullname:", row[2])
    name: wendy ; fullname: Wendy Williams

그러나 나중에 유용성이 분명해질 다른 방법은 :class:`~sqlalchemy.schema.Column` 객체를
키로 직접 사용하는 것이다:

.. sourcecode:: pycon+sql

    {sql}>>> for row in conn.execute(s):
    ...     print("name:", row[users.c.name], "; fullname:", row[users.c.fullname])
    SELECT users.id, users.name, users.fullname
    FROM users
    ()
    {stop}name: jack ; fullname: Jack Jones
    name: wendy ; fullname: Wendy Williams

대기(pending) 상태의 행이 남아있는 결과 집합은 버리기 전에 명확하게 닫아져야 한다.
:class:`~sqlalchemy.engine.ResultProxy`\ 는 객체가 가비지(garbage) 수집되는 동안
참조하는 커서와 연결 리소스가 각각 닫히고 연결 풀로 반환되지만, 일부 데이터베이스 API는 이런 일에
매우 까다로워서 명확하게 지정하는 것이 좋다:

.. sourcecode:: pycon+sql

    >>> result.close()

COLUMNS 절에 있는 컬럼을 더 주의해서 제어하고 싶다면, :class:`~sqlalchemy.schema.Table`\ 의
개별적인 :class:`~sqlalchemy.schema.Column` 객체를 참조한다.
이것들은 :class:`~sqlalchemy.schema.Table` 객체의 ``c`` 속성에서 명명된 속성으로
사용할 수 있다:

.. sourcecode:: pycon+sql

    >>> s = select([users.c.name, users.c.fullname])
    {sql}>>> result = conn.execute(s)
    SELECT users.name, users.fullname
    FROM users
    ()
    {stop}>>> for row in result:
    ...     print(row)
    (u'jack', u'Jack Jones')
    (u'wendy', u'Wendy Williams')

FROM 절의 재밌는 점을 보자. 생성된 명령문은 두개의 구분되는 섹션인
"SELECT columns" 부분과 "FROM table" 부분 있는데, :func:`.select` 구문은 오직
컬럼이 포함된 목록만 갖고 있다. 어떻게 작동하는가? :func:`.select` 문에 *두 개의* 테이블을 넣어보자:

.. sourcecode:: pycon+sql

    {sql}>>> for row in conn.execute(select([users, addresses])):
    ...     print(row)
    SELECT users.id, users.name, users.fullname, addresses.id, addresses.user_id, addresses.email_address
    FROM users, addresses
    ()
    {stop}(1, u'jack', u'Jack Jones', 1, 1, u'jack@yahoo.com')
    (1, u'jack', u'Jack Jones', 2, 1, u'jack@msn.com')
    (1, u'jack', u'Jack Jones', 3, 2, u'www@www.org')
    (1, u'jack', u'Jack Jones', 4, 2, u'wendy@aol.com')
    (2, u'wendy', u'Wendy Williams', 1, 1, u'jack@yahoo.com')
    (2, u'wendy', u'Wendy Williams', 2, 1, u'jack@msn.com')
    (2, u'wendy', u'Wendy Williams', 3, 2, u'www@www.org')
    (2, u'wendy', u'Wendy Williams', 4, 2, u'wendy@aol.com')

**2개의 모든** 테이블을 FROM 절에 넣었다. 그러나 동시에 난장판이 됐다.
SQL 조인에 익숙한 사람은 이것이 **곱집합(Cartesian product)**\ 이라는 것을 안다.
이는 ``users`` 테이블의 각 행이 ``addresses`` 테이블의 각 행에 대해 만들어지는 것이다:

.. sourcecode:: pycon+sql

    >>> s = select([users, addresses]).where(users.c.id == addresses.c.user_id)
    {sql}>>> for row in conn.execute(s):
    ...     print(row)
    SELECT users.id, users.name, users.fullname, addresses.id,
       addresses.user_id, addresses.email_address
    FROM users, addresses
    WHERE users.id = addresses.user_id
    ()
    {stop}(1, u'jack', u'Jack Jones', 1, 1, u'jack@yahoo.com')
    (1, u'jack', u'Jack Jones', 2, 1, u'jack@msn.com')
    (2, u'wendy', u'Wendy Williams', 3, 2, u'www@www.org')
    (2, u'wendy', u'Wendy Williams', 4, 2, u'wendy@aol.com')

훨씬 나아 보이도록 :func:`.select`\ 에 표현식을 추가한다. 이는
``WHERE users.id = addresses.user_id``\ 을 명령문에 추가하는 효과가 있으며,
``users``\ 와 ``addresses`` 행의 조인이 의미있도록 만들었다. 하지만 표현식을 보면?
이는 단지 2개의 다른 :class:`~sqlalchemy.schema.Column` 객체 사이에서 파이썬
일치 연산자를 사용한 것이다. 뭔가 있는 것이 분명하다.
``1 == 1``\ 과 ``1 == 2``\ 는 각각 WHERE 절이 아니라 ``True``\ 와
``False``\ 를 내놓는다. 정확히 표현식이 무엇을 하는지 보자:

.. sourcecode:: pycon+sql

    >>> users.c.id == addresses.c.user_id
    <sqlalchemy.sql.elements.BinaryExpression object at 0x...>

놀랍게도 이건 ``True``\ 도 아니고 ``False``\ 도 아니다! 이건 뭘까?

.. sourcecode:: pycon+sql

    >>> str(users.c.id == addresses.c.user_id)
    'users.id = addresses.user_id'

보다시피 파이썬의 ``__eq__()`` 내장 함수 덕분에, ``==`` 연산자는 지금까지 만든
:class:`~.expression.Insert`\ 와 :func:`.select` 객체와 매우 비슷한 객체를 생성한다.
``str()``\ 을 호출하면 SQL이 생성된다. 이제 작업하는 것이 궁극적으로 같은 타입의 객체라는 것을 알 수
있다. SQlAlchemy는 이런 표현식들의 기본 클래스를 :class:`~.expression.ColumnElement`\ 라고
한다.


Operators
=========

SQLAlchemy의 연산자 패러다임을 우연히 발견했으니, 몇가지 기능을 살펴보자.
두 개의 컬럼을 서로 어떻게 같게 하는지 봤다:

.. sourcecode:: pycon+sql

    >>> print(users.c.id == addresses.c.user_id)
    users.id = addresses.user_id

리터럴 값(SQlAlchemy 절 객체가 아닌)을 사용하면, 바인드 파라미터를 얻는다:

.. sourcecode:: pycon+sql

    >>> print(users.c.id == 7)
    users.id = :id_1

리터럴 ``7``\ 은 :class:`~.expression.ColumnElement` 결과에 포함된다.
:class:`~sqlalchemy.sql.expression.Insert` 객체와 같은 트릭을 써서 그것을 볼 수 있다:

.. sourcecode:: pycon+sql

    >>> (users.c.id == 7).compile().params
    {u'id_1': 7}

밝혀진대로, 대부분의 파이썬 연산자는 같다, 같지 않다 등의 SQL 표현식을 만든다:

.. sourcecode:: pycon+sql

    >>> print(users.c.id != 7)
    users.id != :id_1

    >>> # None converts to IS NULL
    >>> print(users.c.name == None)
    users.name IS NULL

    >>> # reverse works too
    >>> print('fred' > users.c.name)
    users.name < :name_1

두 개의 정수 컬럼을 더하면, 더하기 표현식을 얻는다:

.. sourcecode:: pycon+sql

    >>> print(users.c.id + addresses.c.id)
    users.id + addresses.id

흥미롭게도, :class:`~sqlalchemy.schema.Column`\ 의 타입은 중요하다!
만약 컬럼을 기반으로 한 2개의 문자열을 ``+``\ 와 함께 사용하면
(맨 처음 :class:`~sqlalchemy.schema.Column` 객체에
:class:`~sqlalchemy.types.Integer`\ 와 :class:`~sqlalchemy.types.String`\ 을 같은
타입을 넣었다는 것을 상기해라), 뭔가 다른 것을 얻는다:

.. sourcecode:: pycon+sql

    >>> print(users.c.name + users.c.fullname)
    users.name || users.fullname

``||``\ 는 대부분의 데이터베이스에서 사용되는 문자열 연결 연산자다. 그러나 전부는 아니다.
MySQL 사용자들, 걱정말아라:

.. sourcecode:: pycon+sql

    >>> print((users.c.name + users.c.fullname).
    ...      compile(bind=create_engine('mysql://'))) # doctest: +SKIP
    concat(users.name, users.fullname)

위 내용은 MYSQL 데이터 베이스에 연결된 :class:`~sqlalchemy.engine.Engine`\ 용으로
생성된 SQL을 보여준다. ``||`` 연산자는 이제 MySQL의 ``concat()`` 함수로 컴파일한다.

실제 사용할 수 없는 연산자를 만나면, 항상 :meth:`.Operators.op` 메서드를 사용할 수 있다.
이는 필요한 어떤 연산자든 생성한다:

.. sourcecode:: pycon+sql

    >>> print(users.c.name.op('tiddlywinks')('foo'))
    users.name tiddlywinks :name_1

이 함수는 또한 비트 연산자를 명시적으로 만드는 것에도 사용될 수 있다. 예::

    somecolumn.op('&')(0xff)

위는 ``somecolumn`` 값의 비트연산자 AND다.

:meth:`.Operators.op`\ 를 사용할 때, 특히 연산자가 결과 컬럼로 보내질 표현식에서 사용될 때,
표현식의 반환 타입은 중요하다. 이 경우 일반적으로 예상되지 않는다면, :func:`.type_coerce` 사용해
타입을 명시적으로 지정해줘야 한다::

    from sqlalchemy import type_coerce
    expr = type_coerce(somecolumn.op('-%>')('foo'), MySpecialType())
    stmt = select([expr])

불린 자료형 연산자는 :meth:`.Operators.bool_op` 메서드를 사용해라. 이는 표현식의 반환 타입이
불린 자료형으로 처리되도 한다::

    somecolumn.bool_op('-->')('some value')

.. versionadded:: 1.2.0b3  Added the :meth:`.Operators.bool_op` method.

Operator Customization
----------------------

급하게 사용자 정의 연산자를 사용하려면 :meth:`.Operators.op`\ 가 편리하지만,
코어 방식은 타입 수준에서 연산자 시스템의 기본적인 사용자 정의 및 확장을 지원한다.
기존 연산자의 동작을 타입별로 수정할 수 있으며, 특정 유형의 일부인 모든 컬럼 표현식에서 사용할 수 있는
새로운 연산을 정의할 수 있다. 설명은 :ref:`types_operators`\ 을 봐라.


Conjunctions
============

우리는 :func:`.select` 구문 내부에 있는 연산자 중 일부를 자랑하고 싶다.
그러나 그것들을 좀 더 합쳐야 하기 때문에, 먼저 conjunction을 소개한다.
conjunction은 AND나 OR 같은 작은 단어다. NOT도 떠오를 것이다.
:func:`.and_`, :func:`.or_`, :func:`.not_`\ 도 SQLAlchemy가 제공하는
상응하는 함수에서 작동할 수 있다.
(:meth:`~.ColumnOperators.like`\ 도 함께 제공한다.):

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import and_, or_, not_
    >>> print(and_(
    ...         users.c.name.like('j%'),
    ...         users.c.id == addresses.c.user_id,
    ...         or_(
    ...              addresses.c.email_address == 'wendy@aol.com',
    ...              addresses.c.email_address == 'jack@yahoo.com'
    ...         ),
    ...         not_(users.c.id > 5)
    ...       )
    ...  )
    users.name LIKE :name_1 AND users.id = addresses.user_id AND
    (addresses.email_address = :email_address_1
       OR addresses.email_address = :email_address_2)
    AND users.id <= :id_1

파이썬 연산자의 우선 순위 때문에 괄호를 잘 살펴야하지만, 재정리된 비트연산자 AND, OR, NOT을
쓸 수도 있다:

.. sourcecode:: pycon+sql

    >>> print(users.c.name.like('j%') & (users.c.id == addresses.c.user_id) &
    ...     (
    ...       (addresses.c.email_address == 'wendy@aol.com') | \
    ...       (addresses.c.email_address == 'jack@yahoo.com')
    ...     ) \
    ...     & ~(users.c.id>5)
    ... )
    users.name LIKE :name_1 AND users.id = addresses.user_id AND
    (addresses.email_address = :email_address_1
        OR addresses.email_address = :email_address_2)
    AND users.id <= :id_1

이 모든 단어를 이용해서 AOL과 MSN의 이메일 주소를 가지며, 이름이 "m"과 "z" 사이의 문자로 시작하는
모든 유저를 선택해보자. 또한, 그들의 이메일 주소와 결합된 전체 이름이 담긴 컬럼을 생성할 것이다.
이 명령문에 :meth:`~.ColumnOperators.between`\ 과 :meth:`~.ColumnElement.label`
두 개의 새로운 구문을 추가할 것이다. :meth:`~.ColumnOperators.between`\ 은
BETWEEN 절을 만들고, :meth:`~.ColumnElement.label`\ 은 ``AS`` 키워드를 사용해
레이블을 만들기 위해 컬럼 표현식에서 사용된다. 이름이 없는 표현식에서 선택할 때 권장된다:

.. sourcecode:: pycon+sql

    >>> s = select([(users.c.fullname +
    ...               ", " + addresses.c.email_address).
    ...                label('title')]).\
    ...        where(
    ...           and_(
    ...               users.c.id == addresses.c.user_id,
    ...               users.c.name.between('m', 'z'),
    ...               or_(
    ...                  addresses.c.email_address.like('%@aol.com'),
    ...                  addresses.c.email_address.like('%@msn.com')
    ...               )
    ...           )
    ...        )
    >>> conn.execute(s).fetchall()
    SELECT users.fullname || ? || addresses.email_address AS title
    FROM users, addresses
    WHERE users.id = addresses.user_id AND users.name BETWEEN ? AND ? AND
    (addresses.email_address LIKE ? OR addresses.email_address LIKE ?)
    (', ', 'm', 'z', '%@aol.com', '%@msn.com')
    [(u'Wendy Williams, wendy@aol.com',)]

또 다시, SQlAlchemy는 명령문에 대한 FROM 절을 알아냈다. 실제로 다른 모든 비트들을 기반으로
FROM 절을 정할 것이다; column 절, where 절, 그리고 ORDER BY, GROUP BY, HAVING 같이
아직 다루지 않은 몇몇 다른 요소들.

:func:`.and_`\ 를 사용한 쉬운 방법은 여러 :meth:`~.Select.where` 절을 연속적으로
사용하는 것이다. 위의 내용은 이렇게도 쓸 수 있다:

.. sourcecode:: pycon+sql

    >>> s = select([(users.c.fullname +
    ...               ", " + addresses.c.email_address).
    ...                label('title')]).\
    ...        where(users.c.id == addresses.c.user_id).\
    ...        where(users.c.name.between('m', 'z')).\
    ...        where(
    ...               or_(
    ...                  addresses.c.email_address.like('%@aol.com'),
    ...                  addresses.c.email_address.like('%@msn.com')
    ...               )
    ...        )
    >>> conn.execute(s).fetchall()
    SELECT users.fullname || ? || addresses.email_address AS title
    FROM users, addresses
    WHERE users.id = addresses.user_id AND users.name BETWEEN ? AND ? AND
    (addresses.email_address LIKE ? OR addresses.email_address LIKE ?)
    (', ', 'm', 'z', '%@aol.com', '%@msn.com')
    [(u'Wendy Williams, wendy@aol.com',)]

연속적인 메서드 호출을 통해 :func:`.select` 구문을 만들 수 있는 방법을
:term:`method chaining`\ 이라고 부른다.


.. _sqlexpression_text:

SQL 문자열 직접 사용
=========================

마지막 예는 타이핑하기 힘들다. SQL 표현식을 프로그래밍 스타일로 구성 요소를 그룹화하는
파이썬 구문으로 이해하는 것은 어려울 수 있다. 그래서 SQLAlchemy에서는 SQL이 이미 알려져 있고
동적 기능을 지원하는 명령문이 꼭 필요하지 않은 경우에, 문자열을 사용할 수 있다.
:func:`~.expression.text` 구문은 거의 변하지 않는 데이터베이스에 전달되는
텍스트 명령문을 작성하는 데 사용된다.
아래에서 :func:`~.expression.text` 객체를 만들고 실행한다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import text
    >>> s = text(
    ...     "SELECT users.fullname || ', ' || addresses.email_address AS title "
    ...         "FROM users, addresses "
    ...         "WHERE users.id = addresses.user_id "
    ...         "AND users.name BETWEEN :x AND :y "
    ...         "AND (addresses.email_address LIKE :e1 "
    ...             "OR addresses.email_address LIKE :e2)")
    {sql}>>> conn.execute(s, x='m', y='z', e1='%@aol.com', e2='%@msn.com').fetchall()
    SELECT users.fullname || ', ' || addresses.email_address AS title
    FROM users, addresses
    WHERE users.id = addresses.user_id AND users.name BETWEEN ? AND ? AND
    (addresses.email_address LIKE ? OR addresses.email_address LIKE ?)
    ('m', 'z', '%@aol.com', '%@msn.com')
    {stop}[(u'Wendy Williams, wendy@aol.com',)]

위에서 우리는 바인딩된 파라미터가 명명된 콜론 형식을 사용한 :func:`~.expression.text`\ 에
명시되는 것을 볼 수 있다. 이 형식은 데이터베이스 백엔드에 상관없이 동일하다. 파라미터에 값을
보내기 위해, :meth:`~.Connection.execute` 메서드를 추가 인수로 전달했다.

Specifying Bound Parameter Behaviors
------------------------------------

:func:`~.expression.text` 구문은 :meth:`.TextClause.bindparams` 메서드를 사용해
미리 설정된 연결 값을 지원한다::

    stmt = text("SELECT * FROM users WHERE users.name BETWEEN :x AND :y")
    stmt = stmt.bindparams(x="m", y="z")

파라미터는 명시적으로 타입이 지정될 수도 있다::

    stmt = stmt.bindparams(bindparam("x", String), bindparam("y", String))
    result = conn.execute(stmt, {"x": "m", "y": "z"})

형식에 데이터타입에서 제공되는 파이썬이나 특정 SQL에서의 처리가 필요할 때,
바운드 파라미터의 형식을 지정하는 것이 필요하다.


.. seealso::

    :meth:`.TextClause.bindparams` - full method description

.. _sqlexpression_text_columns:

Specifying Result-Column Behaviors
----------------------------------

:meth:`.TextClause.columns` 메서드를 사용한 결과 컬럼에 대한 정보도 지정할 수 있다.
이 메서드를 통해 이름을 기준으로 반환 타입을 명시할 수 있다::

    stmt = stmt.columns(id=Integer, name=String)

혹은, 형식이 지정되든 아니든 위치(순서)를 통해 전체 컬럼 표현식을 전달할 수 있다.
이 경우 컬럼을 원문 SQL에 명시적으로 나열하는 것이 좋다.
컬럼 표현식과 SQL의 상관관계가 위치에 따라 수행되기 때문이다::

    stmt = text("SELECT id, name FROM users")
    stmt = stmt.columns(users.c.id, users.c.name)

:meth:`.TextClause.columns` 메서드를 호출하면, 전체 :attr:`.TextAsFrom.c`\ 와
다른 "선택 가능한" 연산자들을 지원하는 :class:`.TextAsFrom` 객체를 얻는다::

    j = stmt.join(addresses, stmt.c.id == addresses.c.user_id)

    new_stmt = select([stmt.c.id, addresses.c.id]).\
        select_from(j).where(stmt.c.name == 'x')

:meth:`.TextClause.columns`\ 의 위치 형식은 SQL 문자열을 기존 Core나 ORM 모델과 관련지을 때
특히 유용하다. 이름 충돌이나 원문 SQL의 결과 컬럼 이름에 대한 다른 문제 없이 컬럼 표현식을 직접적으로
사용할 수 있기 때문이다:

.. sourcecode:: pycon+sql

    >>> stmt = text("SELECT users.id, addresses.id, users.id, "
    ...     "users.name, addresses.email_address AS email "
    ...     "FROM users JOIN addresses ON users.id=addresses.user_id "
    ...     "WHERE users.id = 1").columns(
    ...        users.c.id,
    ...        addresses.c.id,
    ...        addresses.c.user_id,
    ...        users.c.name,
    ...        addresses.c.email_address
    ...     )
    {sql}>>> result = conn.execute(stmt)
    SELECT users.id, addresses.id, users.id, users.name,
        addresses.email_address AS email
    FROM users JOIN addresses ON users.id=addresses.user_id WHERE users.id = 1
    ()
    {stop}

위에서, "id"라고 명명된 결과의 3개 컬럼이 있지만, 이것들을 컬럼 표현식과 위치에따라 결합시킬 것이기 때문에,
실제 컬럼 객체를 키로 사용해서 결과 컬럼을 가져올때 이름은 발행되지 않을 것이다.
가져온 ``email_address`` 컬럼::

    >>> row = result.fetchone()
    >>> row[addresses.c.email_address]
    'jack@yahoo.com'

문자열 컬럼 키를 사용하면, 이름 기반 매칭의 일반적인 규칙은 여전히 적용되고,
``id`` 값에 대한 모호한 컬럼 에러를 얻는다::

    >>> row["id"]
    Traceback (most recent call last):
    ...
    InvalidRequestError: Ambiguous column name 'id' in result set column descriptions

: class :`.Column` 객체를 사용해 결과 세트의 컬럼에 액세스하는 것은 일반적이지 않아 보일 수 있지만,
이는 실제로 :class:`~.orm.query.Query` 객체의 표면 아래에서 투명하게 발생하는 ORM에서
사용하는 유일한 시스템이다. 이런 방식으로 :meth:`.TextClause.columns` 메서드는
ORM에서 사용할 텍스트 명령문에 매우 적합하다. :ref:`orm_tutorial_literal_sql`\ 의 예는
간단한 사용법을 보여준다.

.. versionadded:: 1.1

    이제 :meth:`.TextClause.columns` 메서드는 일반 텍스트 SQL 결과 세트에 위치상으로 일치될
    컬럼 표현식을 수용한다. 테이블 메타데이터나 ORM 모델을 SQL 문자열과 일치시킬 때,
    SQL 문에서 컬럼 이름이 일치하거나 고유할 필요가 없다.

.. seealso::

    :meth:`.TextClause.columns` - 전체 메서드 설명

    :ref:`orm_tutorial_literal_sql` - ORM 레벨 쿼리를 :func:`.text`\ 와 통합하기


Using text() fragments inside bigger statements
-------------------------------------------------

:func:`~.expression.text`\ 를 :func:`~.expression.select` 내에서 자유로울 수 있는
SQL의 조각을 만드는데 사용할 수 있으며, :func:`~.expression.text` 객체를
대부분의 빌더 함수에 대한 인수로 허용한다. 아래를 보면, :func:`.select` 객체 내
:func:`~.expression.text`\ 의 사용법을 결합했다. :func:`~.expression.select` 구문은
명령문의 "기하학적 구조"를 제공하고, :func:`~.expression.text` 구문은 형식 내 텍스트 내용을
제공한다. 사전 설정된 :class:`.Table` 메타데이터를 참조할 필요없이 명령문을 빌드할 수 있다:

.. sourcecode:: pycon+sql

    >>> s = select([
    ...        text("users.fullname || ', ' || addresses.email_address AS title")
    ...     ]).\
    ...         where(
    ...             and_(
    ...                 text("users.id = addresses.user_id"),
    ...                 text("users.name BETWEEN 'm' AND 'z'"),
    ...                 text(
    ...                     "(addresses.email_address LIKE :x "
    ...                     "OR addresses.email_address LIKE :y)")
    ...             )
    ...         ).select_from(text('users, addresses'))
    {sql}>>> conn.execute(s, x='%@aol.com', y='%@msn.com').fetchall()
    SELECT users.fullname || ', ' || addresses.email_address AS title
    FROM users, addresses
    WHERE users.id = addresses.user_id AND users.name BETWEEN 'm' AND 'z'
    AND (addresses.email_address LIKE ? OR addresses.email_address LIKE ?)
    ('%@aol.com', '%@msn.com')
    {stop}[(u'Wendy Williams, wendy@aol.com',)]

.. versionchanged:: 1.0.0
   :func:`.select` 구문은 문자열 SQL 조각이 :func:`.text`\ 로 강제될 때 경고를 보낸다.
   :func:`.text`\ 은 명시적으로 사용돼야 한다. 배경에 대해서는 :ref:`migration_2992`\를 봐라.


.. _sqlexpression_literal_column:

Using More Specific Text with :func:`.table`, :func:`.literal_column`, and :func:`.column`
-------------------------------------------------------------------------------------------

명령문의 주요 요소 중 일부로 :func:`~.expression.column`,
:func:`~.expression.literal_column`, :func:`~.expression.table`\ 를 사용해서
구조의 레벨을 다른 방향으로 되돌릴 수 있다. 이 구문을 사용하면, :func:`~.expression.text`\ 를
사용하는 것보다 더 많은 표현식 기능을 얻을 수 있다. 코어(Core)에 저장된 문자열을 사용하는 방법에 대한
더 많은 정보를 제공하지만, 여전히 메타데이터에 기반한 전체 :class:`.Table`\ 을 사용할 필요는 없다.
아래에서, 두개의 핵심 :func:`~.expression.literal_column` 객체에 대해
:class:`.String` 데이터타입을 지정해, 문자열 관련 연결 연산자를 사용할 수 있도록 한다.
또한, :func:`~.expression.literal_column`\ 도 사용해, 그대로 렌더링되는
``users.fullname``\ 과 같은 테이블 한정 표현식을 사용한다.
:func:`~.expression.column`\ 를 사용하면 인용할 수 있는 개별적인 컬럼 이름이 나타난다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select, and_, text, String
    >>> from sqlalchemy.sql import table, literal_column
    >>> s = select([
    ...    literal_column("users.fullname", String) +
    ...    ', ' +
    ...    literal_column("addresses.email_address").label("title")
    ... ]).\
    ...    where(
    ...        and_(
    ...            literal_column("users.id") == literal_column("addresses.user_id"),
    ...            text("users.name BETWEEN 'm' AND 'z'"),
    ...            text(
    ...                "(addresses.email_address LIKE :x OR "
    ...                "addresses.email_address LIKE :y)")
    ...        )
    ...    ).select_from(table('users')).select_from(table('addresses'))

    {sql}>>> conn.execute(s, x='%@aol.com', y='%@msn.com').fetchall()
    SELECT users.fullname || ? || addresses.email_address AS anon_1
    FROM users, addresses
    WHERE users.id = addresses.user_id
    AND users.name BETWEEN 'm' AND 'z'
    AND (addresses.email_address LIKE ? OR addresses.email_address LIKE ?)
    (', ', '%@aol.com', '%@msn.com')
    {stop}[(u'Wendy Williams, wendy@aol.com',)]

레이블에 따라 정렬하거나 그룹화하기
------------------------------------

때때로 문자열을 숏컷으로 사용하길 원하는 곳은, 명령문이 "ORDER BY"나 "GROUP BY" 절에
참조하고 싶은 레이블이 붙여진 컬럼 요소를 갖고 있을 때다. 다른 경우는 "OVER"나 "DISTINCT" 절의 필드를
포함한다. 그런 레이블이 :func:`.select`\ 에 있다면, 문자열을 바로
:meth:`.select.order_by`\ 나 :meth:`.select.group_by`\ 등에 보내 직접 참조할 수 있다.
이는 명명된 레이블을 참조하며, 표현식이 두번 렌더링되는 것을 막아줄 것이다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func
    >>> stmt = select([
    ...         addresses.c.user_id,
    ...         func.count(addresses.c.id).label('num_addresses')]).\
    ...         order_by("num_addresses")

    {sql}>>> conn.execute(stmt).fetchall()
    SELECT addresses.user_id, count(addresses.id) AS num_addresses
    FROM addresses ORDER BY num_addresses
    ()
    {stop}[(2, 4)]

문자열 이름을 전달해서 :func:`.asc`\ 나 :func:`.desc` 같은 수정자를 이용할 수 있다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func, desc
    >>> stmt = select([
    ...         addresses.c.user_id,
    ...         func.count(addresses.c.id).label('num_addresses')]).\
    ...         order_by(desc("num_addresses"))

    {sql}>>> conn.execute(stmt).fetchall()
    SELECT addresses.user_id, count(addresses.id) AS num_addresses
    FROM addresses ORDER BY num_addresses DESC
    ()
    {stop}[(2, 4)]

여기 문자열은 기능은 이미 :meth:`~.ColumnElement.label` 메서드를 사용해서
특정하게 명명된 레이블을 만들 때에 맞춰져 있다. 다른 경우에 표현식 시스템이 가장 효과적으로
렌더링할 수 있도록 항상 :class:`.ColumnElement` 객체를 직접적으로 참조한다.
아래에, 한 번 이상 나타나는 컬럼 이름을 통해 정렬하고자 할 때,
어떻게 :class:`.ColumnElement`\ 를 이용해 모호함을 없애는지 설명한다:

.. sourcecode:: pycon+sql

    >>> u1a, u1b = users.alias(), users.alias()
    >>> stmt = select([u1a, u1b]).\
    ...             where(u1a.c.name > u1b.c.name).\
    ...             order_by(u1a.c.name)  # using "name" here would be ambiguous

    {sql}>>> conn.execute(stmt).fetchall()
    SELECT users_1.id, users_1.name, users_1.fullname, users_2.id,
    users_2.name, users_2.fullname
    FROM users AS users_1, users AS users_2
    WHERE users_1.name > users_2.name ORDER BY users_1.name
    ()
    {stop}[(2, u'wendy', u'Wendy Williams', 1, u'jack', u'Jack Jones')]


앨리어스(Alias) 사용하기
====================================

SQL의 alias는 SELECT 혹은 테이블의 "이름이 바뀐" 버전에 해당하는데,
"SELECT .. FROM sometable AS someothername"을 실행할 때마다 발생한다.
``AS``\ 는 테이블에 새 이름을 생성한다. Alias는 어떤 테이블이나 서브쿼리를 고유한 이름으로
참조할 수 있도록 하는 주요 구문이다. 테이블의 경우에, 이는 같은 테이블이 FROM 절에서 여러번
명명될 수 있도록 한다. SELECT 문의 경우에, 명령문으로 나타난 컬럼의 부모 이름을 제공해
이 이름에 관해 참조할 수 있다.

SQLAlchemy에는, :class:`.Table`, :func:`.select` 구문이나 다른 선택가능한 것들은
:class:`.Alias` 구문을 생성하는 :meth:`.FromClause.alias` 메서드를 통해 alias로
변환될 수 있다. 예를 들어, 사용자 ``jack``\ 에게 두 개의 특정 이메일 주소가 있다는 것을
안다고 하자. 이 두 주소의 결합을 통해 어떻게 jack을 찾을 수 있을까? 이를 위해 각 주소마다 한번씩
``addresses`` 테이블에 join을 사용한다. ``addresses``\ 에 대해 두 개의
:class:`.Alias` 구문을 생성하고, :func:`.select` 구문 안에서 둘 다 사용한다:

.. sourcecode:: pycon+sql

    >>> a1 = addresses.alias()
    >>> a2 = addresses.alias()
    >>> s = select([users]).\
    ...        where(and_(
    ...            users.c.id == a1.c.user_id,
    ...            users.c.id == a2.c.user_id,
    ...            a1.c.email_address == 'jack@msn.com',
    ...            a2.c.email_address == 'jack@yahoo.com'
    ...        ))
    {sql}>>> conn.execute(s).fetchall()
    SELECT users.id, users.name, users.fullname
    FROM users, addresses AS addresses_1, addresses AS addresses_2
    WHERE users.id = addresses_1.user_id
        AND users.id = addresses_2.user_id
        AND addresses_1.email_address = ?
        AND addresses_2.email_address = ?
    ('jack@msn.com', 'jack@yahoo.com')
    {stop}[(1, u'jack', u'Jack Jones')]

:class:`.Alias` 구문은 마지막 SQL 결과에서 ``addresses_1``\ 과 ``addresses_2`` 이름을
생성한다. 이 이름들의 생성은 명령문 안의 구문 위치에 의해 정해진다. 두번째 ``a2`` alias만을
이용해 쿼리를 생성한다면, 이름은 ``addresses_1``\ 으로 나온다. 이름의 생성은 또한 *결정적*\ 이다.
이는 동일한 SQLAlchemy 명령문 구문은 특정 dialect가 렌더링될 때마다 동일한 SQL 문자열을
생성한다는 것을 의미한다.

밖에서는 :class:`.Alias` 구문 자체를 사용해 alias를 참조하므로, 생성된 이름에 대해
고려하지 않아도 된다. 그러나 디버깅을 위해 :meth:`.FromClause.alias` 메서드에 문자열 이름을
전달해 지정할 수 있다::

    >>> a1 = addresses.alias('a1')

물론 Alias는 SELECT 문 자체를 포함해 SELECT에서 가능한 모든 것에 사용할 수 있다.
모든 명령문의 alias를 만들어, 생성한 :func:`.select`\ 로 ``users`` 테이블을 다시 join하게
만들 수 있다. ``correlate(None)`` 지시문은 내부 ``users`` 테이블과 외부의 것을 "연관"시키려는
SQLAlchemy의 시도를 막는다:

.. sourcecode:: pycon+sql

    >>> a1 = s.correlate(None).alias()
    >>> s = select([users.c.name]).where(users.c.id == a1.c.id)
    {sql}>>> conn.execute(s).fetchall()
    SELECT users.name
    FROM users,
        (SELECT users.id AS id, users.name AS name, users.fullname AS fullname
            FROM users, addresses AS addresses_1, addresses AS addresses_2
            WHERE users.id = addresses_1.user_id AND users.id = addresses_2.user_id
            AND addresses_1.email_address = ?
            AND addresses_2.email_address = ?) AS anon_1
    WHERE users.id = anon_1.id
    ('jack@msn.com', 'jack@yahoo.com')
    {stop}[(u'jack',)]

Join 사용하기
================

SELECT 표현식을 구성할 수 있는 중반에 도달했다. SELECT의 다음 초석은 JOIN 표현식이다.
우리는 이미 :func:`.select` 구문의 컬럼 절이나 where 절에 두 개 테이블을 배치하는
예를 통해 join을 해봤다. 진짜 "JOIN"나 "OUTERJOIN" 구문을 만들려면
:meth:`~.FromClause.join`\ 과 :meth:`~.FromClause.outerjoin` 메서드를 사용해야 한다.
이 메서드는 보통 join의 왼쪽 테이블부터 접근한다:

.. sourcecode:: pycon+sql

    >>> print(users.join(addresses))
    users JOIN addresses ON users.id = addresses.user_id

예민한 독자는 더 놀라운 것들을 보게 될 것이다. SQLAlchemy는 어떻게 두 테이블을 JOIN 하는지 알아냈다!
join의 ON 조건은 이 튜토리얼의 시작 부분에서 ``addresses`` 테이블 방식으로 만들어진
:class:`~sqlalchemy.schema.ForeignKey` 객체를 기반으로 자동 생성됐다.
이미 ``join()`` 구문은 테이블을 join 하는데 더 나은 방법인 것으로 보인다.

같은 이름을 사용하는 모든 사용자에 대해 그들의 이메일 주소를 사용자이름으로 join하는 것 같이,
원하는 표현식이 무엇이든 join 할 수 있다:

.. sourcecode:: pycon+sql

    >>> print(users.join(addresses,
    ...                 addresses.c.email_address.like(users.c.name + '%')
    ...             )
    ...  )
    users JOIN addresses ON addresses.email_address LIKE users.name || :name_1

:func:`.select` 구문을 만들면, SQLAlchemy는 앞서 언급한 테이블을 살펴본 후,
명령문의 FROM 절에 놓는다. 그러나 JOIN을 사용하면, 어떤 FROM 절을 원하는지 알기 때문에
:meth:`~.Select.select_from` 메서드를 사용한다:

.. sourcecode:: pycon+sql

    >>> s = select([users.c.fullname]).select_from(
    ...    users.join(addresses,
    ...             addresses.c.email_address.like(users.c.name + '%'))
    ...    )
    {sql}>>> conn.execute(s).fetchall()
    SELECT users.fullname
    FROM users JOIN addresses ON addresses.email_address LIKE users.name || ?
    ('%',)
    {stop}[(u'Jack Jones',), (u'Jack Jones',), (u'Wendy Williams',)]

:meth:`~.FromClause.outerjoin` 메서드는 ``LEFT OUTER JOIN``\ 을 생성하며,
:meth:`~.FromClause.join`\ 과 같은 방식으로 사용된다:

.. sourcecode:: pycon+sql

    >>> s = select([users.c.fullname]).select_from(users.outerjoin(addresses))
    >>> print(s)
    SELECT users.fullname
        FROM users
        LEFT OUTER JOIN addresses ON users.id = addresses.user_id

이것은 ``outerjoin()``\ 이 만든 결과다.
물론, 9 버전 이전의 오라클을 사용하고, 오라클 전용 SQL을 사용하기 위해 ``OracleDialect``\ 를 통해
엔진을 설정하지 않은 경우에 이렇게 나온다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.dialects.oracle import dialect as OracleDialect
    >>> print(s.compile(dialect=OracleDialect(use_ansi=False)))
    SELECT users.fullname
    FROM users, addresses
    WHERE users.id = addresses.user_id(+)

이 SQL이 무엇을 의미하는지 모르겠어도 걱정하지 말아라! 오라클 DBA의 비밀 부족은 그들의 흑마법이
발견되는 것을 원치 않는다;).

.. seealso::

    :func:`.expression.join`

    :func:`.expression.outerjoin`

    :class:`.Join`

그 외의 모든 것
======================

지금까지 SQL 표현식 생성이 개념을 설명했다. 남은 것은 같은 테마의 다양한 변형이다.
이제 알아야 할 나머지 중요한 것들을 정리할 것이다.


.. _coretutorial_bind_param:

바인드 파라미터 객체
-------------------------

이 모든 예에서, SQLAlchemy은 리터럴 표현식이 발생하는 곳에 바인드 파라미터를 생성하는 것으로
분주하다. 고유한 이름으로 고유한 바인드 파라미터를 지정하고,
동일한 명령문을 반복적으로 사용할 수도 있다. :func:`.bindparam` 구문을 주어진 이름으로
바운드 파라미터를 생성하는데 사용할 수 있다. SQLAlchemy는 항상 API 측에서 이름으로 바운드 파라미터를
참조하지만, 데이터베이스 dialect는 실행 시 적절하게 명명되거나 위치 스타일로 변환한다.
여기서는 SQLite의 위치로 변환한다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import bindparam
    >>> s = users.select(users.c.name == bindparam('username'))
    {sql}>>> conn.execute(s, username='wendy').fetchall()
    SELECT users.id, users.name, users.fullname
    FROM users
    WHERE users.name = ?
    ('wendy',)
    {stop}[(2, u'wendy', u'Wendy Williams')]

:func:`.bindparam`\ 의 다른 중요한 측면은 형식이 할당될 수 있다는 것이다.
바인드 파라미터의 형식은 표현식 내에서의 동작과 데이터베이스로 전송되기 전에 연결된 데이터의 처리 방법을
결정할 것이다:

.. sourcecode:: pycon+sql

    >>> s = users.select(users.c.name.like(bindparam('username', type_=String) + text("'%'")))
    {sql}>>> conn.execute(s, username='wendy').fetchall()
    SELECT users.id, users.name, users.fullname
    FROM users
    WHERE users.name LIKE ? || '%'
    ('wendy',)
    {stop}[(2, u'wendy', u'Wendy Williams')]

같은 이름의 :func:`.bindparam` 구문은 여러번 사용될 수도 있다. 실행 파라미터에는
하나의 명명된 값만 필요하다:

.. sourcecode:: pycon+sql

    >>> s = select([users, addresses]).\
    ...     where(
    ...        or_(
    ...          users.c.name.like(
    ...                 bindparam('name', type_=String) + text("'%'")),
    ...          addresses.c.email_address.like(
    ...                 bindparam('name', type_=String) + text("'@%'"))
    ...        )
    ...     ).\
    ...     select_from(users.outerjoin(addresses)).\
    ...     order_by(addresses.c.id)
    {sql}>>> conn.execute(s, name='jack').fetchall()
    SELECT users.id, users.name, users.fullname, addresses.id,
        addresses.user_id, addresses.email_address
    FROM users LEFT OUTER JOIN addresses ON users.id = addresses.user_id
    WHERE users.name LIKE ? || '%' OR addresses.email_address LIKE ? || '@%'
    ORDER BY addresses.id
    ('jack', 'jack')
    {stop}[(1, u'jack', u'Jack Jones', 1, 1, u'jack@yahoo.com'), (1, u'jack', u'Jack Jones', 2, 1, u'jack@msn.com')]

.. seealso::

    :func:`.bindparam`

함수
---------

SQL 함수는 :data:`~.expression.func` 키워드를 통해 만들어진다. 키워드는 속성 접근을 통해
함수를 생성한다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import func
    >>> print(func.now())
    now()

    >>> print(func.concat('x', 'y'))
    concat(:concat_1, :concat_2)

"생성"은 **모든** SQL 함수가 선택된 단어를 기반으로 만들어졌다는 것을 의미한다::

    >>> print(func.xyz_my_goofy_function())
    xyz_my_goofy_function()

특정 행동 규칙이 적용되는 특정 함수 이름을 SQLAlchemy에서 알 수 있다.
예를 들어 "ANSI" 함수는 CURRENT_TIMESTAMP와 같이 뒤에 괄호가 붙지 않는다:

.. sourcecode:: pycon+sql

    >>> print(func.current_timestamp())
    CURRENT_TIMESTAMP

함수는 select 문의 컬럼 절에서 가장 일반적으로 사용되며, 유형뿐만 아니라 레이블도 지정할 수 있다.
문자열 이름을 기준으로 결과 행에 결과를 지정할 수 있도록 함수에 레이블을 지정하는 것을 추천한다.
또한, 결과 설정 처리가 필요하면(유니코드 변환이나 날짜 변환과 같은) 형식을 지정해야 한다.
아래에서는 결과 함수 ``scalar()``\ 를 사용해 첫 행의 첫 열을 읽은 다음 결과를 닫는다.
레이블이 존재해도 이 경우에는 중요하지 않다:

.. sourcecode:: pycon+sql

    >>> conn.execute(
    ...     select([
    ...            func.max(addresses.c.email_address, type_=String).
    ...                label('maxemail')
    ...           ])
    ...     ).scalar()
    {opensql}SELECT max(addresses.email_address) AS maxemail
    FROM addresses
    ()
    {stop}u'www@www.org'

전체 결과 집합을 반환하는 함수를 지원하는 PostgreSQL와 오라클 같은 데이터베이스는 선택 가능한 단위로
조합할 수 있으며, 명령문에서 사용할 수 있다. 데이터베이스 함수 ``calculate()``\ 는
``x``\ 와 ``y``\ 를 파라미터로 받으며, ``q``, ``z``, ``r``\ 로 이름을 지정할
3개 컬럼을 반환한다. ``calculate()`` 함수와 같이 바인드 파라미터뿐만 아니라 "어휘" 컬럼 객체를
사용해 구성할 수 있다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import column
    >>> calculate = select([column('q'), column('z'), column('r')]).\
    ...        select_from(
    ...             func.calculate(
    ...                    bindparam('x'),
    ...                    bindparam('y')
    ...                )
    ...             )
    >>> calc = calculate.alias()
    >>> print(select([users]).where(users.c.id > calc.c.z))
    SELECT users.id, users.name, users.fullname
    FROM users, (SELECT q, z, r
    FROM calculate(:x, :y)) AS anon_1
    WHERE users.id > anon_1.z

``calculate`` 문을 다른 바인드 파라미터로 두 번 사용하고 싶으면,
:func:`~sqlalchemy.sql.expression.ClauseElement.unique_params` 함수가
복사본을 만들어 주고 바인드 파라미터를 "고유한" 함수로 표시해, 충돌하는 이름을 분리해줄 것이다.
우리는 선택 가능한 두 개의 별도 alias를 만든다:

.. sourcecode:: pycon+sql

    >>> calc1 = calculate.alias('c1').unique_params(x=17, y=45)
    >>> calc2 = calculate.alias('c2').unique_params(x=5, y=12)
    >>> s = select([users]).\
    ...         where(users.c.id.between(calc1.c.z, calc2.c.z))
    >>> print(s)
    SELECT users.id, users.name, users.fullname
    FROM users,
        (SELECT q, z, r FROM calculate(:x_1, :y_1)) AS c1,
        (SELECT q, z, r FROM calculate(:x_2, :y_2)) AS c2
    WHERE users.id BETWEEN c1.z AND c2.z

    >>> s.compile().params # doctest: +SKIP
    {u'x_2': 5, u'y_2': 12, u'y_1': 45, u'x_1': 17}

.. seealso::

    :data:`.func`

.. _window_functions:

윈도우 함수
-------------------

:data:`~.expression.func`\ 에서 생성된 함수를 포함해, 모든 :class:`.FunctionElement`\ 는
:meth:`.FunctionElement.over` 메서드를 통해 OVER 절인 "윈도우 함수"로 전환될 수 있다::

    >>> s = select([
    ...         users.c.id,
    ...         func.row_number().over(order_by=users.c.name)
    ...     ])
    >>> print(s)
    SELECT users.id, row_number() OVER (ORDER BY users.name) AS anon_1
    FROM users

:meth:`.FunctionElement.over`\ 는 :paramref:`.expression.over.rows`\ 나
:paramref:`.expression.over.range` 파라미터를 사용한 범위 지정을 지원한다::

    >>> s = select([
    ...         users.c.id,
    ...         func.row_number().over(
    ...                 order_by=users.c.name,
    ...                 rows=(-2, None))
    ...     ])
    >>> print(s)
    SELECT users.id, row_number() OVER
    (ORDER BY users.name ROWS BETWEEN :param_1 PRECEDING AND UNBOUNDED FOLLOWING) AS anon_1
    FROM users

:paramref:`.expression.over.rows`\ 와 :paramref:`.expression.over.range`\ 는
각각 범위에 대한 음과 양의 정수 조합을 포함하는 두 개의 튜플을 허용한다. 0은 "현재 행"을 나타내며,
`None``\ 은 "무한"을 나타낸다. 더 자세한 내용은 :func:`.over`\ 에 대한 예를 봐라.

.. versionadded:: 1.1 support for "rows" and "range" specification for
   window functions

.. seealso::

    :func:`.over`

    :meth:`.FunctionElement.over`


합집합과 다른 집합 연산
----------------------------------

합집합은 UNION과 UNION ALL 두가지 형태로 제공된다. 이는 모듈 수준 함수인
:func:`~.expression.union`\ 과 :func:`~.expression.union_all`\ 를 통해
사용할 수 있다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import union
    >>> u = union(
    ...     addresses.select().
    ...             where(addresses.c.email_address == 'foo@bar.com'),
    ...    addresses.select().
    ...             where(addresses.c.email_address.like('%@yahoo.com')),
    ... ).order_by(addresses.c.email_address)

    {sql}>>> conn.execute(u).fetchall()
    SELECT addresses.id, addresses.user_id, addresses.email_address
    FROM addresses
    WHERE addresses.email_address = ?
    UNION
    SELECT addresses.id, addresses.user_id, addresses.email_address
    FROM addresses
    WHERE addresses.email_address LIKE ? ORDER BY addresses.email_address
    ('foo@bar.com', '%@yahoo.com')
    {stop}[(1, 1, u'jack@yahoo.com')]

:func:`~.expression.intersect`,
:func:`~.expression.intersect_all`,
:func:`~.expression.except_`\ 과 :func:`~.expression.except_all`\ 도
모든 데이터베이스에서 지원하진 않지만, 사용 가능하다:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.sql import except_
    >>> u = except_(
    ...    addresses.select().
    ...             where(addresses.c.email_address.like('%@%.com')),
    ...    addresses.select().
    ...             where(addresses.c.email_address.like('%@msn.com'))
    ... )

    {sql}>>> conn.execute(u).fetchall()
    SELECT addresses.id, addresses.user_id, addresses.email_address
    FROM addresses
    WHERE addresses.email_address LIKE ?
    EXCEPT
    SELECT addresses.id, addresses.user_id, addresses.email_address
    FROM addresses
    WHERE addresses.email_address LIKE ?
    ('%@%.com', '%@msn.com')
    {stop}[(1, 1, u'jack@yahoo.com'), (4, 2, u'wendy@aol.com')]

소위 "중복" 선택의 일반적인 문제는 괄호로 들어가 있기 때문에 발생한다. SQLite는 특히
괄호로 시작하는 문장을 좋아하지 않는다. "중복" 안에 "중복"이 들어가 있을 때, 그 요소 역시 중복인 경우에,
가장 바깥쪽 중복의 첫번째 요소에 ``.alias().select()``\ 를 적용하는 것이 종종 필요하다.
예를 들어 "except\_" 안에 "union"과 "select"를 중첩하려면, SQLite는 "union"을 서브쿼리로
명시하길 원한다:

.. sourcecode:: pycon+sql

    >>> u = except_(
    ...    union(
    ...         addresses.select().
    ...             where(addresses.c.email_address.like('%@yahoo.com')),
    ...         addresses.select().
    ...             where(addresses.c.email_address.like('%@msn.com'))
    ...     ).alias().select(),   # apply subquery here
    ...    addresses.select(addresses.c.email_address.like('%@msn.com'))
    ... )
    {sql}>>> conn.execute(u).fetchall()
    SELECT anon_1.id, anon_1.user_id, anon_1.email_address
    FROM (SELECT addresses.id AS id, addresses.user_id AS user_id,
        addresses.email_address AS email_address
        FROM addresses
        WHERE addresses.email_address LIKE ?
        UNION
        SELECT addresses.id AS id,
            addresses.user_id AS user_id,
            addresses.email_address AS email_address
        FROM addresses
        WHERE addresses.email_address LIKE ?) AS anon_1
    EXCEPT
    SELECT addresses.id, addresses.user_id, addresses.email_address
    FROM addresses
    WHERE addresses.email_address LIKE ?
    ('%@yahoo.com', '%@msn.com', '%@msn.com')
    {stop}[(1, 1, u'jack@yahoo.com')]

.. seealso::

    :func:`.union`

    :func:`.union_all`

    :func:`.intersect`

    :func:`.intersect_all`

    :func:`.except_`

    :func:`.except_all`


.. _scalar_selects:

Scalar Selects
--------------

스칼라 선택(scalar select)은 정확히 한 행과 한 컬럼을 반환하는 SELECT다.
그런 다음 컬럼 표현식으로 사용할 수 있다. 스칼라 선택은 종종 :term:`correlated subquery`\ 이다.
이는 하나 이상의 FROM 절을 얻기 위해, 둘러싼 SELECT 문에 의지한다.

:func:`.select` 구문을 :meth:`~.SelectBase.as_scalar`\ 나
:meth:`~.SelectBase.label` 메서드를 호출해 컬럼 표현식처럼 작동하도록 수정할 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = select([func.count(addresses.c.id)]).\
    ...             where(users.c.id == addresses.c.user_id).\
    ...             as_scalar()

위 구문은 이제 :class:`~.expression.ScalarSelect` 객체이며,
더이상 class:`~.expression.FromClause` 계층의 일부가 아니다.
대신 표현식 구문의 :class:`~.expression.ColumnElement` 계열 내에 있다.
이 구문을 다른 :func:`.select` 내의 다른 컬럼과 동일하게 놓을 수 있다:

.. sourcecode:: pycon+sql

    >>> conn.execute(select([users.c.name, stmt])).fetchall()
    {opensql}SELECT users.name, (SELECT count(addresses.id) AS count_1
    FROM addresses
    WHERE users.id = addresses.user_id) AS anon_1
    FROM users
    ()
    {stop}[(u'jack', 2), (u'wendy', 2)]

스칼라 선택에 익명이 아닌 컬럼 이름을 적용하려면, :meth:`.SelectBase.label`\ 을 이용해서
생성해야 한다:

.. sourcecode:: pycon+sql

    >>> stmt = select([func.count(addresses.c.id)]).\
    ...             where(users.c.id == addresses.c.user_id).\
    ...             label("address_count")
    >>> conn.execute(select([users.c.name, stmt])).fetchall()
    {opensql}SELECT users.name, (SELECT count(addresses.id) AS count_1
    FROM addresses
    WHERE users.id = addresses.user_id) AS address_count
    FROM users
    ()
    {stop}[(u'jack', 2), (u'wendy', 2)]

.. seealso::

    :meth:`.Select.as_scalar`

    :meth:`.Select.label`

.. _correlated_subqueries:

상호연관 서브쿼리
---------------------

:ref:`scalar_selects`\ 의 예에서 각각의 임베디드된 select의 FROM 절은 ``users`` 테이블을
포함하지 않는다. 존재하고, 내부 SELECT 문에 적어도 하나의 FROM 절을 갖고 있는 경우에,
SQLAlchemy가 자동으로 FROM 객체를 둘러싼 쿼리(존재하는 경우에)에 :term:`correlates` 포함한다.
예:

.. sourcecode:: pycon+sql

    >>> stmt = select([addresses.c.user_id]).\
    ...             where(addresses.c.user_id == users.c.id).\
    ...             where(addresses.c.email_address == 'jack@yahoo.com')
    >>> enclosing_stmt = select([users.c.name]).where(users.c.id == stmt)
    >>> conn.execute(enclosing_stmt).fetchall()
    {opensql}SELECT users.name
    FROM users
    WHERE users.id = (SELECT addresses.user_id
        FROM addresses
        WHERE addresses.user_id = users.id
        AND addresses.email_address = ?)
    ('jack@yahoo.com',)
    {stop}[(u'jack',)]

자기상관은 보통 예정대로 작동하지만, 제어할수도 있다. 예를 들어, 명령문을 ``users`` 테이블을 제외하고
``addresses`` 테이블에만 연관시키고 싶다면, 두 테이블 모두 둘러싼 SELECT 안에 있더라도,
상관관계가 있을 수 있는 FROM 절을 지정하기 위해 :meth:`~.Select.correlate` 메서드를 사용할 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.id]).\
    ...             where(users.c.id == addresses.c.user_id).\
    ...             where(users.c.name == 'jack').\
    ...             correlate(addresses)
    >>> enclosing_stmt = select(
    ...         [users.c.name, addresses.c.email_address]).\
    ...     select_from(users.join(addresses)).\
    ...     where(users.c.id == stmt)
    >>> conn.execute(enclosing_stmt).fetchall()
    {opensql}SELECT users.name, addresses.email_address
     FROM users JOIN addresses ON users.id = addresses.user_id
     WHERE users.id = (SELECT users.id
     FROM users
     WHERE users.id = addresses.user_id AND users.name = ?)
     ('jack',)
     {stop}[(u'jack', u'jack@yahoo.com'), (u'jack', u'jack@msn.com')]

명령문에서 상호연관을 완전히 비활성화하기 위해 인수로 ``None``\ 를 전달할 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.id]).\
    ...             where(users.c.name == 'wendy').\
    ...             correlate(None)
    >>> enclosing_stmt = select([users.c.name]).\
    ...     where(users.c.id == stmt)
    >>> conn.execute(enclosing_stmt).fetchall()
    {opensql}SELECT users.name
     FROM users
     WHERE users.id = (SELECT users.id
      FROM users
      WHERE users.name = ?)
    ('wendy',)
    {stop}[(u'wendy',)]

:meth:`.Select.correlate_except`\ 메서드를 이용해 제외하는 방법으로 상호연관을 제어할 수도 있다.
``users``\ 를 제외한 모든 FROM 절을 연관시키도록 하는 방식으로 ``users`` 테이블에 대해
SELECT를 쓸 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.id]).\
    ...             where(users.c.id == addresses.c.user_id).\
    ...             where(users.c.name == 'jack').\
    ...             correlate_except(users)
    >>> enclosing_stmt = select(
    ...         [users.c.name, addresses.c.email_address]).\
    ...     select_from(users.join(addresses)).\
    ...     where(users.c.id == stmt)
    >>> conn.execute(enclosing_stmt).fetchall()
    {opensql}SELECT users.name, addresses.email_address
     FROM users JOIN addresses ON users.id = addresses.user_id
     WHERE users.id = (SELECT users.id
     FROM users
     WHERE users.id = addresses.user_id AND users.name = ?)
     ('jack',)
     {stop}[(u'jack', u'jack@yahoo.com'), (u'jack', u'jack@msn.com')]

.. _lateral_selects:

LATERAL correlation
^^^^^^^^^^^^^^^^^^^

LATERAL 상관관계는 선택가능한 단위가 단일 FROM 절 내에서 다른 선택가능 단위를 참조할 수 있도록 하는
SQL 상관관계의 특수 하위 카테고리다. 이는 SQL 표준의 일부지만, PostgreSQL의 최신 버전에서
지원되는 매우 특수한 사용 케이스다.

일반적으로 SELECT 문이 FROM 절 내에서 ``table1 JOIN (some SELECT) AS subquery``\ 를
참조하면, 오른쪽 서브쿼리는 왼쪽의 "table1" 표현식을 참조하지 않을 수 있다.
상관관계는 오직 SELECT로 완전히 감싸는 다른 SELECT의 일부 테이블만을 참조한다.
LATERAL 키워드는 아래와 같은 표현식을 허용하면서 이러한 동작을 전환할 수 있다:

.. sourcecode:: sql

    SELECT people.people_id, people.age, people.name
    FROM people JOIN LATERAL (SELECT books.book_id AS book_id
    FROM books WHERE books.owner_id = people.people_id)
    AS book_subq ON true

위의 경우, JOIN의 오른쪽에는 JOIN의 왼쪽과 관련해 "books"와 "people" 테이블을 참조하는
서브쿼리를 포함한다. SQLAlchemy Core는 위와 같은 명령문을 아래와 같이
:meth:`.Select.lateral` 메서드를 사용해 지원한다::

    >>> from sqlalchemy import table, column, select, true
    >>> people = table('people', column('people_id'), column('age'), column('name'))
    >>> books = table('books', column('book_id'), column('owner_id'))
    >>> subq = select([books.c.book_id]).\
    ...      where(books.c.owner_id == people.c.people_id).lateral("book_subq")
    >>> print(select([people]).select_from(people.join(subq, true())))
    SELECT people.people_id, people.age, people.name
    FROM people JOIN LATERAL (SELECT books.book_id AS book_id
    FROM books WHERE books.owner_id = people.people_id)
    AS book_subq ON true

위에서, 선택적 이름을 지정하는 것을 포함해 :meth:`.Select.lateral` 메서드가
:meth:`.Select.alias` 메서드와 매우 비슷하게 작동하는 것을 볼 수 있다.
그러나 :class:`.Alias` 대신 :class:`.Lateral` 구문을 사용한다.
:class:`.Lateral` 구문은 둘러싼 문장의 FROM 절 안에서 상관관계를 허용하는 특수 지시뿐만 아니라
LATERAL 키워드를 제공한다.

:meth:`.Select.lateral` 메서드는 :meth:`.Select.correlate`\ 과
:meth:`.Select.correlate_except` 메서드와 정상적으로 상호작용한다.
단 상관관계 규칙은 명령문의 FROM 절 안에 있는 모든 다른 테이블에도 적용된다.
상관관계는, 기본적으로 이 테이블에 대해 "자동"이며, 테이블이 :meth:`.Select.correlate`\ 에
지정돼 있으면 명시적이고, :meth:`.Select.correlate_except`\ 에 지정된 테이블을 제외하고
모든 테이블에 대해 명시적이다.


.. versionadded:: 1.1

    Support for the LATERAL keyword and lateral correlation.

.. seealso::

    :class:`.Lateral`

    :meth:`.Select.lateral`


Ordering, Grouping, Limiting, Offset...ing...
---------------------------------------------

정렬은 :meth:`~.SelectBase.order_by` 메서드에 컬럼 표현식을 전달해 실행된다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.name]).order_by(users.c.name)
    >>> conn.execute(stmt).fetchall()
    {opensql}SELECT users.name
    FROM users ORDER BY users.name
    ()
    {stop}[(u'jack',), (u'wendy',)]

오름차순과 내림차순은 :meth:`~.ColumnElement.asc`\ 와 :meth:`~.ColumnElement.desc`
수정자를 이용해 제어한다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.name]).order_by(users.c.name.desc())
    >>> conn.execute(stmt).fetchall()
    {opensql}SELECT users.name
    FROM users ORDER BY users.name DESC
    ()
    {stop}[(u'wendy',), (u'jack',)]

그룹화는 GROUP BY 절을 참조하며, 보통 집계(aggregate) 함수와 함께 사용돼 집계할 행의 그룹을
설정한다. 이는 :meth:`~.SelectBase.group_by` 메서드를 통해 제공된다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.name, func.count(addresses.c.id)]).\
    ...             select_from(users.join(addresses)).\
    ...             group_by(users.c.name)
    >>> conn.execute(stmt).fetchall()
    {opensql}SELECT users.name, count(addresses.id) AS count_1
    FROM users JOIN addresses
        ON users.id = addresses.user_id
    GROUP BY users.name
    ()
    {stop}[(u'jack', 2), (u'wendy', 2)]

GROUP BY를 적용한 수, HAVING을 이용해 집계 값에 대한 결과를 필터링할 수 있다.
:meth:`~.Select.having` 메서드를 통해 제공된다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.name, func.count(addresses.c.id)]).\
    ...             select_from(users.join(addresses)).\
    ...             group_by(users.c.name).\
    ...             having(func.length(users.c.name) > 4)
    >>> conn.execute(stmt).fetchall()
    {opensql}SELECT users.name, count(addresses.id) AS count_1
    FROM users JOIN addresses
        ON users.id = addresses.user_id
    GROUP BY users.name
    HAVING length(users.name) > ?
    (4,)
    {stop}[(u'wendy', 2)]

SELECT 문에서 중목을 다루는 가장 일반적인 시스템은 DISTINCT 수정자다.
:meth:`.Select.distinct` 메서드를 이용해 간단한 DISTINCT 절을 추가할 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.name]).\
    ...             where(addresses.c.email_address.
    ...                    contains(users.c.name)).\
    ...             distinct()
    >>> conn.execute(stmt).fetchall()
    {opensql}SELECT DISTINCT users.name
    FROM users, addresses
    WHERE (addresses.email_address LIKE '%' || users.name || '%')
    ()
    {stop}[(u'jack',), (u'wendy',)]

대부분의 데이터베이스 백엔드는 반환되는 행 수를 제한하는 시스템을 지원하며,
대다수는 주어진 "오프셋" 이후에 행을 반환하기 시작하는 방법도 가지고 있다.
PostgreSQL, MySQL, SQLite와 같은 일반적인 백엔드는 LIMIT와 OFFSET 키워드를 제공하는 반면,
다른 백엔드는 같은 효과를 얻기 위해 "윈도우 함수"와 행 id 같은 더 복잡한 기능을 참조해야 한다.
:meth:`~.Select.limit`\ 와 :meth:`~.Select.offset` 메서드는
현재 백엔드의 방법론을 쉽게 추상화한다:

.. sourcecode:: pycon+sql

    >>> stmt = select([users.c.name, addresses.c.email_address]).\
    ...             select_from(users.join(addresses)).\
    ...             limit(1).offset(1)
    >>> conn.execute(stmt).fetchall()
    {opensql}SELECT users.name, addresses.email_address
    FROM users JOIN addresses ON users.id = addresses.user_id
     LIMIT ? OFFSET ?
    (1, 1)
    {stop}[(u'jack', u'jack@msn.com')]


.. _inserts_and_updates:

Inserts, Updates and Deletes
============================

:meth:`~.TableClause.insert`\ 는 이 튜토리얼의 앞에서 설명했다.
:meth:`~.TableClause.insert`\ 는 INSERT를 생성하고,
:meth:`~.TableClause.update` 메서드는 UPDATE를 생성한다.
이 두 구문 모두 명령문의 VALUES나 SET 절을 지정하는 :meth:`~.ValuesBase.values`
메서드를 가지고 있다.

:meth:`~.ValuesBase.values` 메서드는 모든 컬럼 표현식을 값으로 받는다:

.. sourcecode:: pycon+sql

    >>> stmt = users.update().\
    ...             values(fullname="Fullname: " + users.c.name)
    >>> conn.execute(stmt)
    {opensql}UPDATE users SET fullname=(? || users.name)
    ('Fullname: ',)
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

"execute many"에서 :meth:`~.TableClause.insert`\ 나
:meth:`~.TableClause.update`\ 를 사용할 때, 인수 목록에서 참조할 수 있는
명명된 바인딩된 파라미터를 지정하려고 할수도 있다. 두 구문은 실행 시에
:meth:`~.Connection.execute`\ 에 보내진 딕셔너리에 전달된 모든 컬럼 이름에 대해
바인딩된 자리표시자(placeholder)를 자동으로 생성할 것이다.
그러나 명시적으로 목표한 명명된 파라미터를 합성 표현식과 함께 사용하려면
:func:`~.expression.bindparam` 구문을 사용해야 한다.
:func:`~.expression.bindparam`\ 를
:meth:`~.TableClause.insert`\ 나 :meth:`~.TableClause.update`\ 와 함께 사용할 때,
테이블 컬럼의 이름은 바인드 이름의 "자동" 생성을 위해 예약된다. 아래 예제와 같이 암시적으로
사용 가능한 바인드 이름과 명시적으로 명명된 파라미터의 사용을 결합할 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = users.insert().\
    ...         values(name=bindparam('_name') + " .. name")
    >>> conn.execute(stmt, [
    ...        {'id':4, '_name':'name1'},
    ...        {'id':5, '_name':'name2'},
    ...        {'id':6, '_name':'name3'},
    ...     ])
    {opensql}INSERT INTO users (id, name) VALUES (?, (? || ?))
    ((4, 'name1', ' .. name'), (5, 'name2', ' .. name'), (6, 'name3', ' .. name'))
    COMMIT
    <sqlalchemy.engine.result.ResultProxy object at 0x...>

UPDATE 명령문은 :meth:`~.TableClause.update` 구문을 사용해서 내보낼 수 있다.
이는 지정할 수 있는 추가적인 WHERE 절이 있다는 것만 제외하면, INSERT와 매우 유사하게 작동한다:

.. sourcecode:: pycon+sql

    >>> stmt = users.update().\
    ...             where(users.c.name == 'jack').\
    ...             values(name='ed')

    >>> conn.execute(stmt)
    {opensql}UPDATE users SET name=? WHERE users.name = ?
    ('ed', 'jack')
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

"execute many"에서 :meth:`~.TableClause.update`\ 를 사용할 때,
WHERE 절에서 명시적으로 명명된 바인딩된 파라미터를 사용할 수도 있다.
다시 말하지만, :func:`~.expression.bindparam`\ 는 이것을 달성하기 위해 사용되는 구문이다:

.. sourcecode:: pycon+sql

    >>> stmt = users.update().\
    ...             where(users.c.name == bindparam('oldname')).\
    ...             values(name=bindparam('newname'))
    >>> conn.execute(stmt, [
    ...     {'oldname':'jack', 'newname':'ed'},
    ...     {'oldname':'wendy', 'newname':'mary'},
    ...     {'oldname':'jim', 'newname':'jake'},
    ...     ])
    {opensql}UPDATE users SET name=? WHERE users.name = ?
    (('ed', 'jack'), ('mary', 'wendy'), ('jake', 'jim'))
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>


Correlated Updates
------------------

상관관계 업데이트(Correlated update)를 통해 다른 테이블이나 동일한 테이블에서
선택 항목을 사용해 테이블을 업데이트할 수 있다:

.. sourcecode:: pycon+sql

    >>> stmt = select([addresses.c.email_address]).\
    ...             where(addresses.c.user_id == users.c.id).\
    ...             limit(1)
    >>> conn.execute(users.update().values(fullname=stmt))
    {opensql}UPDATE users SET fullname=(SELECT addresses.email_address
        FROM addresses
        WHERE addresses.user_id = users.id
        LIMIT ? OFFSET ?)
    (1, 0)
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

.. _multi_table_updates:

Multiple Table Updates
----------------------

.. versionadded:: 0.7.4

PostgreSQL, Microsoft SQL Server, MySQL 백엔드는 모두 여러 테이블을 참조하는
UPDATE 문을 지원한다. PG와 MSSQL에서는 "UPDATE FROM" 문법이다.
이는 한번에 한 테이블을 업데이트 하지만, WHERE 절에서 직접 참조할 수 있는 부가적인 "FROM" 절을 통해
추가 테이블을 참조할 수 있다. MySQL에서는 쉼표로 분리된 하나의 UPDATE 문에 여러 테이블을
포함시킬 수 있다. SQLAlchemy :func:`.update` 구문은 WHERE 절에 여러 테이블을 지정해
두가지 방식을 모두 지원한다::

    stmt = users.update().\
            values(name='ed wood').\
            where(users.c.id == addresses.c.id).\
            where(addresses.c.email_address.startswith('ed%'))
    conn.execute(stmt)

위 명령문의 결과 SQL은 다음과 같이 렌더링 된다::

    UPDATE users SET name=:name FROM addresses
    WHERE users.id = addresses.id AND
    addresses.email_address LIKE :email_address_1 || '%'

MySQL을 사용할 경우, :meth:`.Update.values`\ 에 전달된 딕셔너리 형식을 사용해
각 테이블의 컬럼을 SET 절에 직접적으로 할당할 수 있다::

    stmt = users.update().\
            values({
                users.c.name:'ed wood',
                addresses.c.email_address:'ed.wood@foo.com'
            }).\
            where(users.c.id == addresses.c.id).\
            where(addresses.c.email_address.startswith('ed%'))

테이블은 SET 절에서 명시적으로 참조된다::

    UPDATE users, addresses SET addresses.email_address=%s,
            users.name=%s WHERE users.id = addresses.id
            AND addresses.email_address LIKE concat(%s, '%')

SQLAlchemy는 이런 구문이 지원되지 않는 데이터베이스에서 사용될 때 특별한 작업을 수행하지 않는다.
``UPDATE FROM`` 구문은 여러 테이블이 있는 경우 기본적으로 생성되며, 이 구문이 지원되지 않는 경우
데이터베이스에서 명령문이 거부된다.

.. _updates_order_parameters:

Parameter-Ordered Updates
-------------------------

SET 절을 렌더링 할 때 :func:`.update` 구문의 기본 동작은
:class:`.Table` 객체에 주어진 컬럼 순서를 이용해 렌더링 하는 것이다.
이것은 특정 컬럼이 있는 특정 UPDATE 문은 매번 동일하게 렌더링될 것임을 의미하므로 중요하다.
이는 클라이언트 측 또는 서버 측의 명령문 형식에 의존하는 쿼리 캐싱 시스템에 영향을 준다.
파라미터 자체는 :meth:`.Update.values` 메서드에 파이썬 딕셔너리 키로 전달되기 때문에
사용할 수 있는 다른 고정된 순서는 없다.

그러나 일부의 경우에는, UPDATE 문의 SET 절에서 렌더링된 파라미터의 순서가 중요할 수 있다.
이에 대한 주요 예는 MySQL을 사용하고, 다른 컬럼 값에 기반한 컬럼 값 업데이트를 제공할 때다.
다음 명령문의 최종 결과::

    UPDATE some_table SET x = y + 10, y = 20

는 다음과 같은 결과를 갖는다::

    UPDATE some_table SET y = 20, x = y + 10

이것은 MySQL에서, 개별 SET 절이 행단위 기준이 아니라 값단위 기준으로 완전하게 인식되고
각 SET 절을 인식할 때마다 행에 포함된 값이 변경되기 때문이다.

이 특정 사용 케이스에 맞춰
:paramref:`~sqlalchemy.sql.expression.update.preserve_parameter_order`
플래그를 사용할 수 있다. 이 플래그를 사용할 때 : meth :`.Update.values` 메서드의 인자로
**2-튜플의 파이썬 리스트**\ 를 제공한다::

    stmt = some_table.update(preserve_parameter_order=True).\
        values([(some_table.c.y, 20), (some_table.c.x, some_table.c.y + 10)])

2-튜플의 리스트는 순서가 있다는 점을 제외하면 본질적으로 파이썬 딕셔너리와 동일한 구조를 가진다.
위 형식을 사용하면 "y" 컬럼의 SET 절이 먼저 렌더링되고 "x" 컬럼의 SET 절이 렌더링된다.

.. versionadded:: 1.0.10 :paramref:`~sqlalchemy.sql.expression.update.preserve_parameter_order`
   플래그를 사용한 명시적인 UPDATE 파라미터의 정렬에 대한 지원 추가.


.. _deletes:

Deletes
-------

마지막으로 삭제다. :meth:`~.TableClause.delete` 구문을 이용하면 쉽게 실행할 수 있다:

.. sourcecode:: pycon+sql

    >>> conn.execute(addresses.delete())
    {opensql}DELETE FROM addresses
    ()
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

    >>> conn.execute(users.delete().where(users.c.name > 'm'))
    {opensql}DELETE FROM users WHERE users.name > ?
    ('m',)
    COMMIT
    {stop}<sqlalchemy.engine.result.ResultProxy object at 0x...>

Matched Row Counts
------------------

:meth:`~.TableClause.update`\ 와 :meth:`~.TableClause.delete` 모두
매치된 행 수와 관련이 있다. 이는 WHERE 절에 의해 매치된 행의 수를 나타낸다.
"매치된"이라는 것은 실제로 UPDATE 되지 않은 행을 포함한다.
값은 :attr:`~.ResultProxy.rowcount`\ 로서 사용 가능하다:

.. sourcecode:: pycon+sql

    >>> result = conn.execute(users.delete())
    {opensql}DELETE FROM users
    ()
    COMMIT
    {stop}>>> result.rowcount
    1

Further Reference
=================

Expression Language Reference: :ref:`expression_api_toplevel`

Database Metadata Reference: :ref:`metadata_toplevel`

Engine Reference: :doc:`/core/engines`

Connection Reference: :ref:`connections_toplevel`

Types Reference: :ref:`types_toplevel`
