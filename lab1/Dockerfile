FROM postgres:15

RUN apt-get update && \
    apt-get install -y \
        netcat-openbsd \
        python3-pip \
        curl \
        python3-psycopg2 \
        python3-venv \
        iputils-ping && \
    rm -rf /var/lib/apt/lists/*

RUN python3 -m venv /opt/patroni-venv && \
    /opt/patroni-venv/bin/pip install --upgrade pip && \
    /opt/patroni-venv/bin/pip install patroni[zookeeper] psycopg2-binary

COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml

ENV PATH="/opt/patroni-venv/bin:$PATH"

USER postgres