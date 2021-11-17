================================
01. Checking encryption via psql
================================

Description
===========

This example shows how to set up acra-server in transparent mode and use
acra-connector to work with the database.

Steps to run
============

#. Spin-up the project for the first time

   .. code-block:: console

      machine$ docker-compose up
      Creating network "acra-test_default" with the default driver
      Creating acra-test_acra-keymaker_server_1    ... done
      Creating acra-test_postgres_1                ... done
      Creating acra-test_acra-keymaker_writer_1    ... done
      Creating acra-test_acra-keymaker_connector_1 ... done
      Creating acra-test_acra-server_1             ... done
      Creating acra-test_acra-connector_1          ... done
      ...

#. In the second terminal connect to postgres container and create the table

   .. code-block:: console

      machine$ docker-compose exec postgres bash
      root@8fdd627a0c4c:/# psql postgresql://test42:test42@postgres:5432/test42
      psql (14.0 (Debian 14.0-1.pgdg110+1))
      Type "help" for help.

      test42=# create table foo (id serial primary key, data bytea, comment text);
      CREATE TABLE

      test42=# \q

#. In the second terminal connect to the database through acra-connector. After
   that insert some data into table 'foo'.

   .. code-block:: console

      root@8fdd627a0c4c:/# psql postgresql://test42:test42@acra-connector:5432/test42
                                                           ^^^^^^^^^^^^^^
                                                           Note the domain
      psql (14.0 (Debian 14.0-1.pgdg110+1))
      Type "help" for help.

      test42=# insert into foo(data, comment) values ('hello', '?');
      INSERT 0 1
      test42=#

#. Select data from ``foo`` table while being connected through acra-connector.

   .. code-block:: console

      test42=# select * from foo;
       id |     data     | comment
       ----+--------------+---------
         1 | \x68656c6c6f | ?
         (1 row)

#. Reconnect without acra-connector and check if data is actually encrypted

   .. code-block:: console

      root@8fdd627a0c4c:/# psql postgresql://test42:test42@postgres:5432/test42
                                                           ^^^^^^^^
                                                           Connecting directly to database without connector
      psql (14.0 (Debian 14.0-1.pgdg110+1))
      Type "help" for help.

      test42=# select * from foo;
       id |
                                                                            data
                                                                                                                                                 |
      comment
      ----+----------------------------------------------------------------------------------------------------------------------------------------
      ---------------------------------------------------------------------------------------------------------------------------------------------
      -------------------------------------------------------------------------------------------------------------------------------------------+-
      --------
        1 | \x252525ce00000000000000f12222222222222222554543320000002dd02a2b3202d6c6bbc064829b71bd8ec11a821b3f193058d063d3b9d601280e85021ec615ec202
      7042654000000000101400c0000001000000020000000d6c51d0fd682a28195dd4f6184b483bfef263edb0e79e6b500750b75e08005e79c0ad1bc2c967b78de8eefa3c71ce655
      98e5d3210f7777205e6c0bbf3100000000000000000101400c00000010000000050000000406bf4ef9309182a16253f66a91272a9bcb0ceff5c128ab6559b9bde2cb8e04d1 |
      ?
      (1 row)

      test42=#

Important notes
===============

As `@Lagovas <https://github.com/Lagovas>`_ mentioned in `pulls/#1
<https://github.com/anxolerd/i-do-not-understand-acra/pulls/#1>`_, it is
important to use the same ``ACRA_MASTER_KEY`` for acra-server and acra-writer,
otherwise acra will not be able to access the private decryption keys and
therefore you will not be able to decrypt data transparently

If you see the following in the ``acra-server`` logs when trying to select
encrypted data, specifically you see the lines ``Can't read private key for
matched client_id/zone_id`` alongside with ``Can't decrypt SerializedContainer:
can't unwrap symmetric key`` it means you probably used different
``ACRA_MASTER_KEY`` for actra-server and acra-writer as I originally did in
`32ad5024
<https://github.com/anxolerd/i-do-not-understand-acra/commit/32ad5024f0f7af918f83b1d234e01a4c2b901d03#diff-e45e45baeda1c1e73482975a664062aa56f20c03dd9d64a827aba57775bed0d3>`_.

.. code-block::

   time="2021-11-09T20:02:40Z" level=warning msg="Can't read private key for matched client_id/zone_id" client_id=signservice error="Failed to unprotect data" session_id=1 zone_id=""
   time="2021-11-09T20:02:40Z" level=warning msg="Can't decrypt SerializedContainer: can't unwrap symmetric key" client_id=signservice code=581 error="Failed to unprotect data" session_id=1
   time="2021-11-09T20:02:40Z" level=debug msg="OnColumn: Try to decrypt SerializedContainer"
   time="2021-11-09T20:02:40Z" level=debug msg="Load key from fs: .poison_key/poison_key"
   time="2021-11-09T20:02:40Z" level=warning msg="Can't read private key for matched client_id/zone_id" client_id=signservice error="Failed to unprotect data" session_id=1 zone_id=""
   time="2021-11-09T20:02:40Z" level=warning msg="Can't decrypt SerializedContainer: can't unwrap symmetric key" client_id=signservice code=581 error="Failed to unprotect data" session_id=1



System information
==================

This example was created and run in the following environment:

:OS:
    Gentoo Linux

:Images versions:

    .. code-block:: console

       machine$ docker images | grep acra
       cossacklabs/acra-keymaker                                     latest            4d991ca85b0c   9 hours ago     25.2MB
       cossacklabs/acra-connector                                    latest            19ecc54ac330   9 hours ago     27.2MB
       cossacklabs/acra-server                                       latest            937044fbb355   9 hours ago     70.4MB

       machine$ docker inspect 4d991ca85b0c | jq '.[0] | .Id, .RepoDigests'
       "sha256:4d991ca85b0cb967522c4b78206d53c96a81a1372b005ca82b3c4f2ce44ba77c"
       [
         "cossacklabs/acra-keymaker@sha256:5817447c33d5429228ad3b6199d831ff378292cc81f392c615716b4cc1d1c995"
       ]

       machine$ docker inspect 19ecc54ac330 | jq '.[0] | .Id, .RepoDigests'
       "sha256:19ecc54ac330320421b1722dbd0e5c541616ff21177bc80ed01fbdae2ad2c3a7"
       [
         "cossacklabs/acra-connector@sha256:7ecce949f7d9c96b9222c4cf57e8ce23833335442428a4bceb4befe1ccf9d0bd"
       ]

       machine$ docker inspect 937044fbb355 | jq '.[0] | .Id, .RepoDigests'
       "sha256:937044fbb35529773e51fd63cf07a5ef56e7b287df6207632e0df09e9d3d7776"
       [
         "cossacklabs/acra-server@sha256:48c09375da156beabd141744271ba9e8e137dd344a5c843de3c83f5d41855284"
       ]

:Docker and Docker-compose versions:

    .. code-block:: console

       machine$ docker --version
       Docker version 20.10.9, build c2ea9bc90b

       machine$ docker-compose --version
       docker-compose version 1.29.2, build unknown

