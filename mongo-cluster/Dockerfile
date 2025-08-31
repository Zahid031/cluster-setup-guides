FROM mongo:8.0.0
COPY mongodb-keyfile /etc/mongodb-keyfile

RUN chown mongodb:mongodb /etc/mongodb-keyfile
RUN chmod 400 /etc/mongodb-keyfile
