# PostgreSQL container for test databases from FMI master databases
# Adapted from official PostgreSQL Dockerfile:
# https://github.com/docker-library/postgres/
#
# https://hub.docker.com/_/centos/

FROM centos:7
MAINTAINER "fmi"

# Ability to override PGDATA from command line or docker-compose.yml
ENV PGDATA=/var/lib/pgsql/data/9.5

# Explicitly set user/group IDs
RUN groupadd -r postgres && useradd -r -g postgres postgres

# Inject excludes into YUM config files.
# https://wiki.postgresql.org/wiki/YUM_Installation
RUN sed -i '/\[base\]/,/gpgkey=/{/gpgkey=/s/$/\'$'\n''exclude = postgres*/;}' /etc/yum.repos.d/CentOS-Base.repo && \
    sed -i '/\[updates\]/,/gpgkey=/{/gpgkey=/s/$/\'$'\n''exclude = postgres*/;}' /etc/yum.repos.d/CentOS-Base.repo

# Install newest PostgreSQL major version repo.
# http://yum.postgresql.org/repopackages.php
# Note: EPEL is required by postgis and also supplies pv.

RUN yum -y update 
RUN yum -y localinstall https://download.postgresql.org/pub/repos/yum/9.5/redhat/rhel-7-x86_64/pgdg-centos95-9.5-2.noarch.rpm
RUN yum -y install postgresql95 postgresql95-server postgresql95-contrib postgresql95-libs postgis2_95 pv && \
    yum clean all && rm -rf /tmp/*
RUN yum -y install lsof

# The database dumps
COPY fminames-test.sql.gz /tmp
COPY fminames-features-test.dmp.gz /tmp
COPY fminames-altadditional-test.dmp.gz /tmp
COPY fminames-additional-test.dmp.gz /tmp
COPY fminames-kwadditional-test.dmp.gz /tmp
COPY fminames-kwhgnadditional-test.dmp.gz /tmp
COPY gis-test.sql.gz /tmp
COPY avi-test.sql.gz /tmp
COPY auth-test.sql.gz /tmp
COPY icemap-test.sql.gz /tmp
COPY icemap-copy.tar.gz /tmp

RUN tar xvf /tmp/icemap-copy.tar.gz -C /tmp

RUN mkdir -p ${PGDATA}

RUN mkdir -p /etc/provision /var/run/postgresql && \
    chown -R postgres ${PGDATA} /var/run/postgresql /tmp /etc/provision

COPY etc/* /etc/provision/

USER postgres

# The first command (only needed once) is to initialize the database in PGDATA.
# -E UTF-8 is needed here because the fminames schema is also UTF-8.
RUN /usr/pgsql-9.5/bin/initdb -E 'UTF-8' -D ${PGDATA}

COPY etc/postgresql.conf ${PGDATA}

# Start the server and create database users.
# -w is needed because otherwise createuser would run before the database is up.
# The server is stopped gracefully in the last step to avoid an error in the next
# layer, where the server is required to be running again.
RUN /usr/pgsql-9.5/bin/pg_ctl -w -s -D ${PGDATA} start && \
    createdb -E 'UTF-8' fminames && \
    createdb -e -E UTF8 gis && \
    createdb -e -E UTF8 avi && \
    createdb -E UTF8 authentication && \
    createdb -e -E UTF8 icemap2storage_ro && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER admin LOGIN SUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'admin_pw';" && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER fminames_user WITH PASSWORD 'fminames_pw';" && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER gis_admin NOLOGIN SUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION;" && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER gis_user WITH PASSWORD 'gis_pw';" && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER avi_user WITH PASSWORD 'avi_pw';" && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER auth_user WITH PASSWORD 'auth_pw';" && \
    /usr/pgsql-9.5/bin/psql -c "CREATE USER iceservice WITH PASSWORD 'iceservice_pw';" && \
    /usr/pgsql-9.5/bin/psql avi admin -c "CREATE SCHEMA avidb_bs_ro;" && \
    /usr/pgsql-9.5/bin/psql avi admin -c "CREATE EXTENSION postgis SCHEMA avidb_bs_ro;" && \
    /usr/pgsql-9.5/bin/psql icemap2storage_ro admin -c "CREATE SCHEMA icemap;" && \
    /usr/pgsql-9.5/bin/psql icemap2storage_ro admin -c "CREATE SCHEMA icemap_static;" && \
    /usr/pgsql-9.5/bin/psql icemap2storage_ro admin -c "CREATE EXTENSION postgis SCHEMA public;" && \
#   /usr/pgsql-9.5/bin/psql -c "ALTER USER admin NOLOGIN;" && \
    /usr/pgsql-9.5/bin/pg_ctl -w -s -D ${PGDATA} stop

# Populate database server from dumps. You might want to add
# --set ON_ERROR_STOP=on as an argument to psql, but the current fminames SQL
# dump will not load in that case.
RUN cd /tmp/ && pv -f fminames-test.sql.gz | gunzip > fminames.sql && \
    /usr/pgsql-9.5/bin/pg_ctl -w -s -D ${PGDATA} start && \
    pv -f /tmp/fminames.sql | /usr/pgsql-9.5/bin/psql fminames && \
    /usr/pgsql-9.5/bin/psql fminames -c "GRANT INSERT ON public.geonames TO fminames_user;" && \
    rm /tmp/fminames.sql && \
    pv -f /tmp/fminames-altadditional-test.dmp.gz | gunzip > fminames-altadditional.dmp && \
    pv -f /tmp/fminames-altadditional.dmp | /usr/pgsql-9.5/bin/psql fminames -c "\\copy public.alternate_geonames (id, source, geonames_id, language, name, preferred, short, colloquial, historic, priority, locked, last_modified) FROM STDIN" && \
    rm /tmp/fminames-altadditional.dmp && \
    pv -f /tmp/fminames-additional-test.dmp.gz | gunzip > fminames-additional.dmp && \
    pv -f /tmp/fminames-additional.dmp | /usr/pgsql-9.5/bin/psql fminames -c "\\copy public.geonames (id, name, ansiname, lat, lon, class, features_code, countries_iso2, cc2, admin1, admin2, admin3, admin4, population, elevation, dem, timezone, modified, municipalities_id, priority, locked, the_geom, the_geog, last_modified, landcover) FROM STDIN" && \
    rm /tmp/fminames-additional.dmp && \
    pv -f /tmp/fminames-kwadditional-test.dmp.gz | gunzip > fminames-kwadditional.dmp && \
    pv -f /tmp/fminames-kwadditional.dmp | /usr/pgsql-9.5/bin/psql fminames -c "\\copy public.keywords (keyword, comment, languages, autocomplete) FROM STDIN" && \
    rm /tmp/fminames-kwadditional.dmp && \
    pv -f /tmp/fminames-kwhgnadditional-test.dmp.gz | gunzip > fminames-kwhgnadditional.dmp && \
    pv -f /tmp/fminames-kwhgnadditional.dmp | /usr/pgsql-9.5/bin/psql fminames -c "\\copy public.keywords_has_geonames (keyword, geonames_id, comment, name, last_modified) FROM STDIN" && \
    rm /tmp/fminames-kwhgnadditional.dmp && \
    /usr/pgsql-9.5/bin/psql fminames -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO fminames_user;" && \
    pv -f gis-test.sql.gz | gunzip | grep -v "OWNER TO partio;" > gis.sql && \
    pv -f /tmp/gis.sql | /usr/pgsql-9.5/bin/psql gis && \
    /usr/pgsql-9.5/bin/psql gis admin -c "SELECT UpdateGeometrySRID('natural_earth', 'admin_0_countries', 'the_geom', 4326);" && \
    /usr/pgsql-9.5/bin/psql gis admin -c "GRANT USAGE ON SCHEMA public TO gis_user;" && \
    rm /tmp/gis.sql && \
    /usr/pgsql-9.5/bin/psql gis -c "GRANT SELECT ON ALL TABLES IN SCHEMA fminames,mml,natural_earth,public TO gis_user;" && \
    pv -f avi-test.sql.gz | gunzip | egrep -v "CREATE SCHEMA avidb_bs_ro" > avi.sql && \
    sed -i 's/public\.geometry/geometry/g' avi.sql && \
    sed -i 's/ TO avidb_r./ TO admin/g' avi.sql && \
    sed -i 's/ FROM avidb_r./ FROM admin/g' avi.sql && \
    sed -i 's/ TO avidb_bs_ro/ TO avi_user/g' avi.sql && \
    sed -i 's/ TO avidb_iwxxm/ TO admin/g' avi.sql && \
    pv -f /tmp/avi.sql | /usr/pgsql-9.5/bin/psql avi && \
    rm /tmp/avi.sql && \
    /usr/pgsql-9.5/bin/psql avi -c "ALTER SCHEMA avidb_bs_ro RENAME TO avi_user;" && \
    /usr/pgsql-9.5/bin/psql avi -c "GRANT USAGE ON SCHEMA avi_user TO avi_user;" && \
    /usr/pgsql-9.5/bin/psql -c "ALTER ROLE avi_user SET search_path TO avi_user,public;" && \
    /usr/pgsql-9.5/bin/psql avi -c "GRANT SELECT ON ALL TABLES IN SCHEMA avi_user,public TO avi_user;" && \
    /usr/pgsql-9.5/bin/psql avi -c "DROP FUNCTION public.avidb_messages_part_trig_func();" && \
    pv -f auth-test.sql.gz | gunzip > auth.sql && \
    sed -i 's/weatherproof_../admin/g' auth.sql && \
    pv -f /tmp/auth.sql | /usr/pgsql-9.5/bin/psql authentication && \
    rm /tmp/auth.sql && \
    /usr/pgsql-9.5/bin/psql authentication -c "GRANT USAGE ON SCHEMA authengine_test TO auth_user;" && \
    /usr/pgsql-9.5/bin/psql authentication -c "GRANT SELECT ON ALL TABLES IN SCHEMA authengine_test TO auth_user;" && \
    pv -f /tmp/icemap-test.sql.gz | gunzip > icemap.sql && \
    pv -f /tmp/icemap.sql | /usr/pgsql-9.5/bin/psql icemap2storage_ro admin && \
    /usr/pgsql-9.5/bin/psql icemap2storage_ro admin -a -f /tmp/icemap-copy.sql && \
    rm /tmp/icemap.sql && \
    /usr/pgsql-9.5/bin/psql icemap2storage_ro -c "GRANT USAGE ON SCHEMA icemap,icemap_static TO iceservice;" && \
    /usr/pgsql-9.5/bin/psql icemap2storage_ro -c "GRANT SELECT ON ALL TABLES IN SCHEMA icemap,icemap_static TO iceservice;" && \
    /usr/pgsql-9.5/bin/pg_ctl -w -s -D ${PGDATA} stop

RUN chmod 0700 ${PGDATA}

# By default PostgreSQL listens on TCP port 5432.
EXPOSE 5432

#Launch the daemon
ENTRYPOINT ["/usr/pgsql-9.5/bin/postgres", "-D", "/var/lib/pgsql/data/9.5"]
