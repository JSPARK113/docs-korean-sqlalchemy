.. _session_toplevel:

=================
세션 사용법
=================

.. module:: sqlalchemy.orm.session

:func:`.orm.mapper` 함수와 :mod:`~sqlalchemy.ext.declarative` 익스텐션은
ORM을 설정하기 위한 가장 기본 도구이지만
일단 설정이 끝나면 데이터 조작을 위한 가장 기본 인터페이스는 :class:`.Session`\ 이 된다.

.. toctree::
    :maxdepth: 2

    session_basics
    session_state_management
    cascades
    session_transaction
    persistence_techniques
    contextual
    session_events
    session_api


