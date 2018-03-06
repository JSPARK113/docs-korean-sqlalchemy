:orphan:

.. _index_toplevel:

========================
SQLAlchemy 문서
========================

시작하기
===============

간단한 설명.

:doc:`개요 <intro_ko>` |
:ref:`설치 가이드 <installation_ko>` |
:doc:`자주 물어보는 질문 <faq/index_ko>` |
:doc:`1.1에서 마이그레이션 하기 <changelog/migration_12>` |
:doc:`용어집 <glossary_ko>` |
:doc:`오류 메세지 <errors_ko>` |
:doc:`체인지로그 목록 <changelog/index_ko>`

SQLAlchemy ORM
==============

여기에서는 Object Relational Mapper를 소개하고 자세하게 설명한다.
자동으로 생성되는 상위 수준 SQL과 파이썬 객체의 자동 기록 기능을 사용하고 싶으면
이 섹션의 튜토리얼부터 진행한다.

* **Read this first:**
  :doc:`orm/tutorial`

* **ORM Configuration:**
  :doc:`Mapper Configuration <orm/mapper_config>` |
  :doc:`Relationship Configuration <orm/relationships>`

* **Configuration Extensions:**
  :doc:`Declarative Extension <orm/extensions/declarative/index>` |
  :doc:`Association Proxy <orm/extensions/associationproxy>` |
  :doc:`Hybrid Attributes <orm/extensions/hybrid>` |
  :doc:`Automap <orm/extensions/automap>` |
  :doc:`Mutable Scalars <orm/extensions/mutable>` |
  :doc:`Indexable <orm/extensions/indexable>`

* **ORM Usage:**
  :doc:`Session Usage and Guidelines <orm/session>` |
  :doc:`Loading Objects <orm/loading_objects>` |
  :doc:`Cached Query Extension <orm/extensions/baked>`

* **Extending the ORM:**
  :doc:`ORM Events and Internals <orm/extending>`

* **Other:**
  :doc:`Introduction to Examples <orm/examples>`

SQLAlchemy 코어
=====================

여기에서는  SQLAlchemy의 SQL 렌더링 엔진, DBAPI 통합, 트랜스잭션 통합, 스키마 서술 서비스에
대해 설명한다.
ORM의 도메인 중심 사용 모드와 달리, SQL 표현 언어는
스키마 중심 사용 패러다임을 제공한다.

* **Read this first:**
  :doc:`core/tutorial`

* **All the Built In SQL:**
  :doc:`SQL Expression API <core/expression_api>`

* **Engines, Connections, Pools:**
  :doc:`Engine Configuration <core/engines>` |
  :doc:`Connections, Transactions <core/connections>` |
  :doc:`Connection Pooling <core/pooling>`

* **Schema Definition:**
  :doc:`Overview <core/schema>` |
  :ref:`Tables and Columns <metadata_describing_toplevel>` |
  :ref:`Database Introspection (Reflection) <metadata_reflection_toplevel>` |
  :ref:`Insert/Update Defaults <metadata_defaults_toplevel>` |
  :ref:`Constraints and Indexes <metadata_constraints_toplevel>` |
  :ref:`Using Data Definition Language (DDL) <metadata_ddl_toplevel>`

* **Datatypes:**
  :ref:`Overview <types_toplevel>` |
  :ref:`Building Custom Types <types_custom>` |
  :ref:`API <types_api>`

* **Core Basics:**
  :doc:`Overview <core/api_basics>` |
  :doc:`Runtime Inspection API <core/inspection>` |
  :doc:`Event System <core/event>` |
  :doc:`Core Event Interfaces <core/events>` |
  :doc:`Creating Custom SQL Constructs <core/compiler>` |


Dialect 문서
======================

**dialect**\ 는 SQLAlchemy에서 다양한 타입의 DBAPI 및 데이터베이스와 커뮤니케이션하기 위해
사용되는 시스템이다.
이 섹션은 관련 내용, 옵션, 개별적인 dialect와 관련된 사용 패턴을 다루고 있다.

:doc:`Index of all Dialects <dialects/index>`

