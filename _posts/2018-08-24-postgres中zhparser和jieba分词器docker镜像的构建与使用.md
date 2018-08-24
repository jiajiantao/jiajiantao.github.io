基于SCWS、zhparser、jieba、rum的Postgres中文全文搜索镜像构建

构建镜像的Dockerfile文件为
===============================================


```
FROM postgres:10
ENV SCWS_VERSION 1.2.3
RUN     mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
        echo "deb http://mirrors.163.com/debian/ stretch main non-free contrib" >/etc/apt/sources.list && \
        echo "deb http://mirrors.163.com/debian/ stretch-updates main non-free contrib" >>/etc/apt/sources.list && \
        echo "deb http://mirrors.163.com/debian/ stretch-backports main non-free contrib" >>/etc/apt/sources.list && \
        echo "deb-src http://mirrors.163.com/debian/ stretch main non-free contrib" >>/etc/apt/sources.list && \
        echo "deb-src http://mirrors.163.com/debian/ stretch-updates main non-free contrib" >>/etc/apt/sources.list && \
        echo "deb-src http://mirrors.163.com/debian/ stretch-backports main non-free contrib" >>/etc/apt/sources.list && \
        echo "deb http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib" >>/etc/apt/sources.list && \
        echo "deb-src http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib" >>/etc/apt/sources.list


RUN     apt-get update && apt-get install -y --no-install-recommends ca-certificates \
        gcc g++ libc6-dev cmake make git wget unzip postgresql-server-dev-10 libpq-dev && rm -rf /var/lib/apt/lists/* && \
        mkdir -p /usr/lib/scws/rum && \
        wget -O /usr/lib/scws/scws-${SCWS_VERSION}.tar.bz2 "http://www.xunsearch.com/scws/down/scws-${SCWS_VERSION}.tar.bz2" && \
        wget -O /usr/lib/scws/zhparser.zip "https://github.com/amutu/zhparser/archive/master.zip" && \
        git clone https://github.com/postgrespro/rum /usr/lib/scws/rum && \
        git clone https://github.com/jaiminpan/pg_jieba /usr/lib/scws/pg_jieba && \
        chown -R postgres.postgres /usr/lib/scws/rum/ && \
        tar xjf /usr/lib/scws/scws-${SCWS_VERSION}.tar.bz2 -C /usr/lib/scws/ && \
        unzip /usr/lib/scws/zhparser.zip -d /usr/lib/scws/ && \
        cd /usr/lib/scws/scws-1.2.3 && \
        ./configure && \
        make install && \
        cd /usr/lib/scws/zhparser-master && \
        SCWS_HOME=/usr/local make && make install 

RUN     /sbin/ldconfig -v && \ 
        chown -R postgres.postgres /usr/lib/postgresql && \
        chown -R postgres.postgres /usr/share/postgresql && \
        chown -R postgres.postgres /usr/include/postgresql && \
        cd /usr/lib/scws/pg_jieba && \
        git submodule update --init --recursive && \
        mkdir /usr/lib/scws/pg_jieba/build && \
        cd /usr/lib/scws/pg_jieba/build &&\ 
        cmake -DPostgreSQL_TYPE_INCLUDE_DIR=/usr/include/postgresql/10/server .. && \
        make && make install  && \
        apt-get purge -y --auto-remove ca-certificates cmake wget unzip && \
        rm -rf /usr/lib/scws/scws* /usr/lib/scws/zhparser* /usr/lib/scws/pg_jieba
```
构建镜像
===============================================

```
docker build -t postgres-scws .
```
然后生成镜像`postgres-scws:latest`
```
[root@hadoop tmp]# docker images | grep postgres-scws
postgres-scws                                              latest              89cb6bdc8748        28 minutes ago      664MB
```

如何使用此镜像
===============================================
* 生成用户自定义配置文件
```
[root@hadoop tmp]# docker run -i --rm postgres-scws cat /usr/share/postgresql/postgresql.conf.sample > config/postgresql.conf
```
* 自定义配置文件，在文件末尾添加SCWS的zhparser用户自定义词典配置信息

```
[root@hadoop tmp]# echo -e "zhparser.extra_dicts = 'labeldic.xdb'\nzhparser.dict_in_memory = 'true'" >>  config/postgresql.conf 
[root@hadoop tmp]# tail config/postgresql.conf 
#include = 'special.conf'               # include file


#------------------------------------------------------------------------------
# CUSTOMIZED OPTIONS
#------------------------------------------------------------------------------

# Add settings for extensions here
zhparser.extra_dicts = 'labeldic.xdb'
zhparser.dict_in_memory = 'true'
```
* 数据库初始化shell脚本

```
[root@hadoop tmp]# cat sql/init.sh 
#!/bin/bash - 
#===============================================================================
#
#          FILE: init.sh 
# 
#         USAGE: ./init.sh  
# 
#   DESCRIPTION: 当镜像启动时，运行脚本生成用户自定义词典的xdb文件及安装postgres的
#                rum模块
#       CREATED: 08/21/2018 05:00:38 PM
#===============================================================================

set -o nounset                              # Treat unset variables as an error
scws-gen-dict -i  /usr/share/postgresql/10/tsearch_data/labeldic.txt -o /usr/share/postgresql/10/tsearch_data/labeldic.xdb -c utf-8 > /dev/null 2>&1
cd /usr/lib/scws/rum
make USE_PGXS=1
make USE_PGXS=1 install
make USE_PGXS=1 installcheck
```
```
[root@hadoop tmp]# cat sql/init.sql
create extension pg_jieba;
create extension zhparser; 
CREATE TEXT SEARCH CONFIGURATION testzhcfg (PARSER = zhparser);
ALTER TEXT SEARCH CONFIGURATION testzhcfg ADD MAPPING FOR n,v,a,i,e,l,t WITH simple
```
```
[root@hadoop tmp]# head -30 sql/tatt.sql        
--
-- PostgreSQL database dump
--

-- Dumped from database version 10.2 (Debian 10.2-1.pgdg90+1)
-- Dumped by pg_dump version 10.2 (Debian 10.2-1.pgdg90+1)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SET check_function_bodies = false;
SET client_min_messages = warning;
SET row_security = off;

SET search_path = public, pg_catalog;

SET default_tablespace = '';

SET default_with_oids = false;

--
-- Name: rmtj_advanced_search; Type: TABLE; Schema: public; Owner: tatt
--

CREATE TABLE rmtj_advanced_search (
    id numeric(32,0) NOT NULL,
    sub_case_type character varying(128),
    sub_case_type_code character varying(128),
```


* 目录结构信息为
```
[root@hadoop tmp]# tree
.
├── config
│   └── postgresql.conf
├── Dockerfile
├── labeldic.txt
└── sql
    └── init.sh

2 directories, 4 files
```
* 启动`postgres-scws:latest`实例
```
[root@hadoop tmp]# docker run --name postgresql -p 5432:5432 -v "$PWD/sql/":/docker-entrypoint-initdb.d/ -v "$PWD/config/postgresql.conf":/etc/postgresql/postgresql.conf -v "$PWD/labeldic.txt":/usr/share/postgresql/10/tsearch_data/labeldic.txt -e POSTGRES_PASSWORD=tatt -e POSTGRES_DB=tatt -e POSTGRES_USER=tatt -d postgres-scws:latest  -c config_file=/etc/postgresql/postgresql.conf 
faa15042fd503ef5f2fb4b6a399331cfa948d6d856bd38ad08da2b601759daeb
```
验证
```
[root@hadoop tmp]# docker-enter faa15042fd50
mesg: ttyname failed: Success
root@faa15042fd50:~# su postgres
$ psql -U tatt -d tatt
could not change directory to "/root": Permission denied
psql (10.2 (Debian 10.2-1.pgdg90+1))
Type "help" for help.

tatt=# create extension zhparser; 
CREATE EXTENSION
tatt=# create extension pg_jieba;
CREATE EXTENSION
tatt=# create extension rum;
CREATE EXTENSION
tatt=# CREATE TEXT SEARCH CONFIGURATION testzhcfg (PARSER = zhparser);
CREATE TEXT SEARCH CONFIGURATION
tatt=# ALTER TEXT SEARCH CONFIGURATION testzhcfg ADD MAPPING FOR n,v,a,i,e,l,t WITH simple;
ALTER TEXT SEARCH CONFIGURATION
tatt=# SELECT * FROM ts_parse('zhparser', '改变机动车离婚');
 tokid |      token      
-------+-----------------
   110 | 改变机动车
   110 | 离婚
(2 rows)

tatt=# select * from to_tsvector('jiebacfg', '小明硕士毕业于中国科学院计算所，后在日本京都大学深造');
                                              to_tsvector                                               
--------------------------------------------------------------------------------------------------------
 '中国科学院':5 '小明':1 '日本京都大学':10 '毕业':3 '深造':11 '硕士':2 '计算所':6
(1 row)

tatt=# select * from rum_ts_distance(to_tsvector('jiebacfg', '小明硕士毕业于中国科学院计算所，后在日本京都大学深造') , to_tsquery('计'));
 rum_ts_distance 
-----------------
         16.4493
(1 row)

tatt=# 
```
* 在docker-swarm中启动

docker stack compose文件为
```
[root@hadoop tmp]# cat pg-scws.yml 
version: "3.2"
networks:
    cluster:
services:
    pg:
        hostname: db
        image: postgres-scws:latest 
        environment:
            #database we want to use for application
            POSTGRES_PASSWORD: tatt
            POSTGRES_USER: tatt
            POSTGRES_DB: tatt
            
        ports:
            - 5432:5432
        volumes:
            - $PWD/sql/:/docker-entrypoint-initdb.d/
            - $PWD/labeldic.txt:/usr/share/postgresql/10/tsearch_data/labeldic.txt
            - $PWD/config/postgresql.conf:/etc/postgresql/postgresql.conf
            - /etc/localtime:/etc/localtime:ro
        command: postgres -c config_file=/etc/postgresql/postgresql.conf
        networks:
            cluster:
                aliases:
                    - pg
```
docker swarm中应用的启动
```
[root@hadoop tmp]# docker stack deploy -c pg-scws.yml pg
Creating network pg_cluster
Creating service pg_pg
[root@hadoop tmp]# docker stack ls 
NAME                SERVICES
pg                  1
[root@hadoop tmp]# docker ps 
CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS              PORTS               NAMES
fe4fb96ed2e3        postgres-scws:latest              "docker-entrypoint.s…"   3 minutes ago       Up 3 minutes        5432/tcp            pg_pg.1.jwq5raxk0webpaewxx0tln6w7
```


