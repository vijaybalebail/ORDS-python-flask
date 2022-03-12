FROM oracleslim19ic

RUN yum -y install nodejs node-oracledb-node14 && \
    rm -rf /var/cache/yum/*

ENV NODE_PATH=/usr/lib/node_modules/

CMD ["/bin/node", "-v"]
