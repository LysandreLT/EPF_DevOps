FROM postgres:14.1-alpine

ENV POSTGRES_DB=db
ENV POSTGRES_USER=usr
ENV POSTGRES_PASSWORD=pwd

COPY CreateScheme.sql /docker-entrypoint-initdb.d
COPY InsertData.sql /docker-entrypoint-initdb.d