###############
Getting Started
###############

Zocalo requires a suitable message broker to be configured. Currently it
supports either `ActiveMQ`_ or `RabbitMQ`_, however the use of RabbitMQ is
strongly recommended. Some features may not be available using ActiveMQ,
and support may be deprecated in future versions of Zocalo.

.. _ActiveMQ: https://activemq.apache.org/components/classic/
.. _RabbitMQ: https://www.rabbitmq.com/


.. code-block:: bash

    export ZOCALO_CONFIG=$PWD/contrib/site-configuration.yml

RabbitMQ
========

Use e.g. `docker`_ or `podman`_ to start a RabbitMQ server:

.. _docker: https://www.docker.com/
.. _podman: https://podman.io/

.. code-block:: bash

    docker run -d --hostname my-rabbit --name zocalo-rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

The container exposes the following ports:

.. list-table::
   :header-rows: 1

   * - Port
     - Description
   * - 5672
     - AMQP 0-9-1
   * - 15672
     - Management UI and HTTP API

Next, configure vhosts, exchanges, queues, bindings and policies using the
``zocalo.configure_rabbitmq`` command:

.. code-block:: bash

    zocalo.configure_rabbitmq contrib/rabbitmq-configuration.yaml

***************
Graylog
***************

This can be started easily with a docker-compose.yml. See https://docs.graylog.org/en/3.3/pages/installation/docker.html for full details.

.. code-block:: yaml

    version: '3'
    services:
    # MongoDB: https://hub.docker.com/_/mongo/
    mongo:
        image: mongo:4.2
        networks:
        - graylog
    # Elasticsearch: https://www.elastic.co/guide/en/elasticsearch/reference/6.x/docker.html
    elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.10.0
        environment:
        - http.host=0.0.0.0
        - transport.host=localhost
        - network.host=0.0.0.0
        - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
        ulimits:
        memlock:
            soft: -1
            hard: -1
        deploy:
        resources:
            limits:
            memory: 1g
        networks:
        - graylog
    # Graylog: https://hub.docker.com/r/graylog/graylog/
    graylog:
        image: graylog/graylog:4.0
        environment:
        - GRAYLOG_PASSWORD_SECRET=mysecret
        # Password: admin
        - GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
        - GRAYLOG_HTTP_EXTERNAL_URI=http://localhost:9000/
        networks:
        - graylog
        restart: always
        depends_on:
        - mongo
        - elasticsearch
        ports:
        # Graylog web interface and REST API
        - 9000:9000
        # Syslog TCP
        - 1514:1514
        # Syslog UDP
        - 1514:1514/udp
        # GELF TCP
        - 12201:12201
        # GELF UDP
        - 12201:12201/udp
    networks:
    graylog:
        driver: bridge


Then start with:

.. code-block:: bash

    docker-compose up

Graylog admin console should be available on http://localhost:9000
Port 12201 is available for python GELF logging. Configure an input in the graylog web console to enable receiving messages.

***************
Zocalo
***************

For developing create a new conda / virtual environment, clone zocalo, and install:

.. code-block:: bash

    conda create -n zocalo
    conda activate zocalo
    git clone https://github.com/DiamondLightSource/python-zocalo
    cd python-zocalo
    pip install -e .


For production, install with pip:

.. code-block:: bash

    pip install zocalo


***************
Configure
***************

Copy `contrib/site-configuration.yml`. At minimum `graylog` and `activemq` must be configured. Environments should be defined for `live` and `test`. Paths to recipes and drop files must also be specified. Messages are written to drop files if ActiveMQ is unavailable.

The config file to use is specified from the environment variable `ZOCALO_CONFIG`.

Sample recipes can be used:

.. code-block:: yaml

    storage:
      plugin: storage
      zocalo.recipe_directory: .../python-zocalo/examples/recipes

===============
JMX
===============

To make use of `zocalo.queue_monitor` and `zocalo.status_monitor` JMX needs to be configured. The JMX configuration points to the Jolokia REST API. When starting ActiveMQ the logs will tells you where the REST API is running

.. code-block:: bash

  INFO | ActiveMQ Jolokia REST API available at http://0.0.0.0:8161/api/jolokia/


So configuration should be 

.. code-block:: yaml

    port: 8161
    host: localhost
    base_url: api/jolokia


Username and password are the same as the web console and defined in `users.properties`

***************
Starting Up
***************

`-e test` will make use of the test environment

Start the dispatcher

.. code-block:: bash

    conda activate zocalo
    zocalo.service -s Dispatcher (-e test)


Start the process runner

.. code-block:: bash

    zocalo.service -s Runner (-e test)


Run the test recipe:

.. code-block:: bash

    zocalo.go -r example -s workingdir="$(pwd)" 1234 (-e test)

***********************
Dead Letter Queue (DLQ)
***********************

The dead letter queue is where rejected messages end up. One dlq is available per topic to easily identify where messages are being rejected. For details on dlq see https://activemq.apache.org/message-redelivery-and-dlq-handling

Messages can be purged using:

.. code-block:: bash

    zocalo.dlq_purge --output-directory=/path/to/dlq (-e test)

And re-injected with:

.. code-block:: bash

    zocalo.dlq_reinject dlq_file (-e test)
