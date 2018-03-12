ORM 설정
=================

.. contents::
    :local:
    :class: faq
    :backlinks: none

.. _faq_mapper_primary_key:

프라이머리 키가 없는 테이블은 어떻게 매핑하는가?
----------------------------------------------------------------------------

SQLAlchemy ORM은 특정 테이블을 매핑할 때 프라이머리 키로 표시한 컬럼이
하나 이상 있어야 한다. 복수 컬럼 프라이머리 키 즉, 복합 키도 물론 사용할 수 있다.
사실 ORM에서 프라이머리 키로 표시한 열이 실제로 데이터베이스에서 반드시 프라이머리 키일
필요는 없다.(물론 데이터베이스에서 프라이머리 키인 편이 좋다.)
이는 해당 열이 프라이머리 키처럼 하나의 행에 대해 유니크하고 널값이 아닌
지정자가 필요하기 때문에 생긴 조건이다.

대부분의 ORM은 객체가 프라이머리 키를 가지도록 요구한다.
왜냐하면 메모리 상의 객체는 데이터베이스 테이블의 특정한 하나의 행과
대응해야 하기 때문이다.
이렇게 하면 UPDATE 또는 DELETE 문으로 테이블의 해당 행을 제외한 다른 행에
영향을 미치지 않는다. 하지만 프라이머리 키의 중요성은 이뿐이 아니다.
SQLAlchemy에서는 모든 매핑 객체가 :class:`.Session` 안에서
아이덴티티 맵(:term:`identity map`)이라는 패턴을 써서
데이터베이스의 특정 행과 유니크하게 연결된다.
이 패턴은 SQLAlchemy가 채택한 작업 단위(unit of work) 패턴의 핵심이며
대부분의 ORM 사용에 있어 필수적이다.


.. note::

    여기에서는 SQLAlchemy ORM에 대해서만 이야기하고 있다.
    Core를 사용하여 :class:`.Table` 객체나 :func:`.select` 를 직접 사용하면
    현재 테이블이나 관련 테이블에서 프라이머리 키를 필요로 하지 않을 수도 있다.
    (하지만 실제 SQL에서는 특정한 행을 갱신하거나 지우기 위해 반드시
    프라이머리 키에 해당하는 무었인가가 있어야 한다.)

대부분의 경우, 테이블은 후보 키(:term:`candidate key`)라는 것을 가진다.
후보 키는 행을 유니크하게 구별할 수 있는 하나 혹은 복수의 열을 말한다.
만약 테이블에 이게 없다면 중복된 행을 가질 수도 있고
`1차 정규형  <http://en.wikipedia.org/wiki/First_normal_form>`_\ 이 아니고
매핑할 수도 없는 테이블이라는 뜻이다.
테이블에 후보 키 역할을 하는 열을 직접 설정할 수도 있다.::

    class SomeClass(Base):
        __table__ = some_table_with_no_pk
        __mapper_args__ = {
            'primary_key':[some_table_with_no_pk.c.uid, some_table_with_no_pk.c.bar]
        }

더 좋은 방법은 메타데이터에서 이 열에 ``primary_key=True`` 속성을 가하는 것이다.::

    class SomeClass(Base):
        __tablename__ = "some_table_with_no_pk"

        uid = Column(Integer, primary_key=True)
        bar = Column(String, primary_key=True)

관계형 데이터베이스의 모든 테이블은 프라이머리 키가 있어야 한다.
다대다(many-to-many) 관계의 테이블에서는 프라이머리 키가
두 개의 연관 열의 조합이 될 수 있다.::

    CREATE TABLE my_association (
      user_id INTEGER REFERENCES user(id),
      account_id INTEGER REFERENCES account(id),
      PRIMARY KEY (user_id, account_id)
    )


How do I configure a Column that is a Python reserved word or similar?
----------------------------------------------------------------------

Column-based attributes can be given any name desired in the mapping. See
:ref:`mapper_column_distinct_names`.

How do I get a list of all columns, relationships, mapped attributes, etc. given a mapped class?
-------------------------------------------------------------------------------------------------

This information is all available from the :class:`.Mapper` object.

To get at the :class:`.Mapper` for a particular mapped class, call the
:func:`.inspect` function on it::

    from sqlalchemy import inspect

    mapper = inspect(MyClass)

From there, all information about the class can be accessed through properties
such as:

* :attr:`.Mapper.attrs` - a namespace of all mapped attributes.  The attributes
  themselves are instances of :class:`.MapperProperty`, which contain additional
  attributes that can lead to the mapped SQL expression or column, if applicable.

* :attr:`.Mapper.column_attrs` - the mapped attribute namespace
  limited to column and SQL expression attributes.   You might want to use
  :attr:`.Mapper.columns` to get at the :class:`.Column` objects directly.

* :attr:`.Mapper.relationships` - namespace of all :class:`.RelationshipProperty` attributes.

* :attr:`.Mapper.all_orm_descriptors` - namespace of all mapped attributes, plus user-defined
  attributes defined using systems such as :class:`.hybrid_property`, :class:`.AssociationProxy` and others.

* :attr:`.Mapper.columns` - A namespace of :class:`.Column` objects and other named
  SQL expressions associated with the mapping.

* :attr:`.Mapper.mapped_table` - The :class:`.Table` or other selectable to which
  this mapper is mapped.

* :attr:`.Mapper.local_table` - The :class:`.Table` that is "local" to this mapper;
  this differs from :attr:`.Mapper.mapped_table` in the case of a mapper mapped
  using inheritance to a composed selectable.

.. _faq_combining_columns:

I'm getting a warning or error about "Implicitly combining column X under attribute Y"
--------------------------------------------------------------------------------------

This condition refers to when a mapping contains two columns that are being
mapped under the same attribute name due to their name, but there's no indication
that this is intentional.  A mapped class needs to have explicit names for
every attribute that is to store an independent value; when two columns have the
same name and aren't disambiguated, they fall under the same attribute and
the effect is that the value from one column is **copied** into the other, based
on which column was assigned to the attribute first.

This behavior is often desirable and is allowed without warning in the case
where the two columns are linked together via a foreign key relationship
within an inheritance mapping.   When the warning or exception occurs, the
issue can be resolved by either assigning the columns to differently-named
attributes, or if combining them together is desired, by using
:func:`.column_property` to make this explicit.

Given the example as follows::

    from sqlalchemy import Integer, Column, ForeignKey
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()

    class A(Base):
        __tablename__ = 'a'

        id = Column(Integer, primary_key=True)

    class B(A):
        __tablename__ = 'b'

        id = Column(Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey('a.id'))

As of SQLAlchemy version 0.9.5, the above condition is detected, and will
warn that the ``id`` column of ``A`` and ``B`` is being combined under
the same-named attribute ``id``, which above is a serious issue since it means
that a ``B`` object's primary key will always mirror that of its ``A``.

A mapping which resolves this is as follows::

    class A(Base):
        __tablename__ = 'a'

        id = Column(Integer, primary_key=True)

    class B(A):
        __tablename__ = 'b'

        b_id = Column('id', Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey('a.id'))

Suppose we did want ``A.id`` and ``B.id`` to be mirrors of each other, despite
the fact that ``B.a_id`` is where ``A.id`` is related.  We could combine
them together using :func:`.column_property`::

    class A(Base):
        __tablename__ = 'a'

        id = Column(Integer, primary_key=True)

    class B(A):
        __tablename__ = 'b'

        # probably not what you want, but this is a demonstration
        id = column_property(Column(Integer, primary_key=True), A.id)
        a_id = Column(Integer, ForeignKey('a.id'))



I'm using Declarative and setting primaryjoin/secondaryjoin using an ``and_()`` or ``or_()``, and I am getting an error message about foreign keys.
------------------------------------------------------------------------------------------------------------------------------------------------------------------

Are you doing this?::

    class MyClass(Base):
        # ....

        foo = relationship("Dest", primaryjoin=and_("MyClass.id==Dest.foo_id", "MyClass.foo==Dest.bar"))

That's an ``and_()`` of two string expressions, which SQLAlchemy cannot apply any mapping towards.  Declarative allows :func:`.relationship` arguments to be specified as strings, which are converted into expression objects using ``eval()``.   But this doesn't occur inside of an ``and_()`` expression - it's a special operation declarative applies only to the *entirety* of what's passed to primaryjoin or other arguments as a string::

    class MyClass(Base):
        # ....

        foo = relationship("Dest", primaryjoin="and_(MyClass.id==Dest.foo_id, MyClass.foo==Dest.bar)")

Or if the objects you need are already available, skip the strings::

    class MyClass(Base):
        # ....

        foo = relationship(Dest, primaryjoin=and_(MyClass.id==Dest.foo_id, MyClass.foo==Dest.bar))

The same idea applies to all the other arguments, such as ``foreign_keys``::

    # wrong !
    foo = relationship(Dest, foreign_keys=["Dest.foo_id", "Dest.bar_id"])

    # correct !
    foo = relationship(Dest, foreign_keys="[Dest.foo_id, Dest.bar_id]")

    # also correct !
    foo = relationship(Dest, foreign_keys=[Dest.foo_id, Dest.bar_id])

    # if you're using columns from the class that you're inside of, just use the column objects !
    class MyClass(Base):
        foo_id = Column(...)
        bar_id = Column(...)
        # ...

        foo = relationship(Dest, foreign_keys=[foo_id, bar_id])

.. _faq_subqueryload_limit_sort:

Why is ``ORDER BY`` required with ``LIMIT`` (especially with ``subqueryload()``)?
---------------------------------------------------------------------------------

A relational database can return rows in any
arbitrary order, when an explicit ordering is not set.
While this ordering very often corresponds to the natural
order of rows within a table, this is not the case for all databases and
all queries.   The consequence of this is that any query that limits rows
using ``LIMIT`` or ``OFFSET`` should **always** specify an ``ORDER BY``.
Otherwise, it is not deterministic which rows will actually be returned.

When we use a SQLAlchemy method like :meth:`.Query.first`, we are in fact
applying a ``LIMIT`` of one to the query, so without an explicit ordering
it is not deterministic what row we actually get back.
While we may not notice this for simple queries on databases that usually
returns rows in their natural
order, it becomes much more of an issue if we also use :func:`.orm.subqueryload`
to load related collections, and we may not be loading the collections
as intended.

SQLAlchemy implements :func:`.orm.subqueryload` by issuing a separate query,
the results of which are matched up to the results from the first query.
We see two queries emitted like this:

.. sourcecode:: python+sql

    >>> session.query(User).options(subqueryload(User.addresses)).all()
    {opensql}-- the "main" query
    SELECT users.id AS users_id
    FROM users
    {stop}
    {opensql}-- the "load" query issued by subqueryload
    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id FROM users) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id

The second query embeds the first query as a source of rows.
When the inner query uses ``OFFSET`` and/or ``LIMIT`` without ordering,
the two queries may not see the same results:

.. sourcecode:: python+sql

    >>> user = session.query(User).options(subqueryload(User.addresses)).first()
    {opensql}-- the "main" query
    SELECT users.id AS users_id
    FROM users
     LIMIT 1
    {stop}
    {opensql}-- the "load" query issued by subqueryload
    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id FROM users LIMIT 1) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id

Depending on database specifics, there is
a chance we may get a result like the following for the two queries::

    -- query #1
    +--------+
    |users_id|
    +--------+
    |       1|
    +--------+

    -- query #2
    +------------+-----------------+---------------+
    |addresses_id|addresses_user_id|anon_1_users_id|
    +------------+-----------------+---------------+
    |           3|                2|              2|
    +------------+-----------------+---------------+
    |           4|                2|              2|
    +------------+-----------------+---------------+

Above, we receive two ``addresses`` rows for ``user.id`` of 2, and none for
1.  We've wasted two rows and failed to actually load the collection.  This
is an insidious error because without looking at the SQL and the results, the
ORM will not show that there's any issue; if we access the ``addresses``
for the ``User`` we have, it will emit a lazy load for the collection and we
won't see that anything actually went wrong.

The solution to this problem is to always specify a deterministic sort order,
so that the main query always returns the same set of rows. This generally
means that you should :meth:`.Query.order_by` on a unique column on the table.
The primary key is a good choice for this::

    session.query(User).options(subqueryload(User.addresses)).order_by(User.id).first()

Note that the :func:`.joinedload` eager loader strategy does not suffer from
the same problem because only one query is ever issued, so the load query
cannot be different from the main query.  Similarly, the :func:`.selectinload`
eager loader strategy also does not have this issue as it links its collection
loads directly to primary key values just loaded.

.. seealso::

    :ref:`subqueryload_ordering`
