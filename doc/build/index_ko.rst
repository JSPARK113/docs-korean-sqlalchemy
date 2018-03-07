:orphan:

.. _index_toplevel:

========================
SQLAlchemy 문서
========================

시작하기
===============

SQLAlchemy에 대한 간단한 설명.

:doc:`개요 <intro_ko>` |
:ref:`설치 가이드 <installation>` |
:doc:`자주 물어보는 질문 <faq/index_ko>` |
:doc:`1.1에서 마이그레이션 하기 <changelog/migration_12_ko>` |
:doc:`용어집 <glossary_ko>` |
:doc:`오류 메세지 <errors_ko>` |
:doc:`변경사항 목록 <changelog/index_ko>`

SQLAlchemy ORM
==============

여기에서는 ORM(Object Relational Mapper)을 소개한다.
자동으로 생성되는 고수준 SQL과 파이썬 객체의 자동 기록 기능을 사용하고 싶으면
이 섹션의 튜토리얼부터 진행한다.

* **가장 처음에 읽어야하는 문서:**
  :doc:`orm/tutorial_ko`

* **ORM 설정:**
  :doc:`Mapper Configuration <orm/mapper_config_ko>` |
  :doc:`Relationship Configuration <orm/relationships_ko>`

* **설정용 확장 프로그램:**
  :doc:`선언형(declarative) <orm/extensions/declarative/index_ko>` |
  :doc:`연관 프록시(Association Proxy) <orm/extensions/associationproxy_ko>` |
  :doc:`하이브리드 속성(Hybrid Attributes) <orm/extensions/hybrid_ko>` |
  :doc:`오토맵(Automap) <orm/extensions/automap_ko>` |
  :doc:`뮤터블 스칼라(Mutable Scalars) <orm/extensions/mutable_ko>` |
  :doc:`인덱서블(Indexable) <orm/extensions/indexable_ko>`

* **ORM 사용법:**
  :doc:`세션의 사용과 가이드라인 <orm/session_ko>` |
  :doc:`객체 로딩 <orm/loading_objects_ko>` |
  :doc:`쿼리 캐쉬 <orm/extensions/baked_ko>`

* **ORM 확장하기:**
  :doc:`ORM 이벤트와 내부 구조 <orm/extending_ko>`

* **기타:**
  :doc:`예제 소개 <orm/examples_ko>`

SQLAlchemy 코어
=====================

여기에서는 SQLAlchemy의 SQL 렌더링 엔진, DBAPI 통합, 트랜스잭션 통합, 스키마 서술 서비스에
대해 설명한다.
ORM의 도메인 중심의 사용 방식과 달리, SQL 표현식 언어는 스키마 중심의 사용 패러다임을 제공한다.

* **가장 처음에 읽어야하는 문서:**
  :doc:`core/tutorial_ko`

* **SQL 내장 기능:**
  :doc:`SQL 표현식 API <core/expression_api_ko>`

* **엔진, 컨넥션, 풀링:**
  :doc:`엔진 설정 <core/engines_ko>` |
  :doc:`컨넥션, 트랜잭션 <core/connections_ko>` |
  :doc:`컨넥션 풀링 <core/pooling_ko>`

* **스키마 정의:**
  :doc:`개요 <core/schema>` |
  :ref:`테이블과 컬럼 <metadata_describing_toplevel_ko>` |
  :ref:`데이터베이스 인트로스펙션(Reflection) <metadata_reflection_toplevel_ko>` |
  :ref:`인서트/업데이트 디폴트 <metadata_defaults_toplevel_ko>` |
  :ref:`제약조건과 인덱스 <metadata_constraints_toplevel_ko>` |
  :ref:`데이터 정의 언어(DDL: Data Definition Language) 사용법 <metadata_ddl_toplevel_ko>`

* **Datatypes:**
  :ref:`개요 <types_toplevel_ko>` |
  :ref:`커스텀 자료형 정의 <types_custom_ko>` |
  :ref:`API <types_api_ko>`

* **코어 기초:**
  :doc:`개요 <core/api_basics_ko>` |
  :doc:`런타임 인트로스펙션(Inspection) API <core/inspection_ko>` |
  :doc:`이벤트 시스템 <core/event_ko>` |
  :doc:`코어 이벤트 인터페이스 <core/events_ko>` |
  :doc:`커스텀 SQL 생성 <core/compiler_ko>` |


Dialect 문서
======================

**dialect**\ 는 SQLAlchemy에서 다양한 타입의 DBAPI 및 데이터베이스를 사용하기 위한 시스템이다.
이 섹션에서는 개별적인 dialect 및 이와 관련된 사용 패턴을 다루고 있다.

:doc:`Index of all Dialects <dialects/index_ko>`

