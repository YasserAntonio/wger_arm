#
# A wger installation under apache with WSGI
#
# Note: you MUST build this image from the project's root!
# docker build -f extras/docker/apache/Dockerfile --tag wger/apache .
#
# Please consult the documentation for usage
# docker build --tag wger/apache .
# docker run -ti --name wger.apache --publish 8000:80 wger/apache
#
# To stop the container:
# sudo docker container stop wger.apache
#
# To start developing again
# sudo docker container start --attach wger.apache
#

##########
# Builder
##########
FROM yasserantonio/wger_base:armv7 as builder

RUN apt-get update \
  && apt-get install --no-install-recommends -y \
      build-essential \
      python3.8-dev \
      python3-pip \
      python3-wheel \
      wget \
      git

RUN apt-get install  --no-install-recommends -y libtiff-dev libjpeg8-dev zlib1g-dev libfreetype6-dev liblcms2-dev libwebp-dev xli libpq-dev  # solve pillow build fail and psycopg2-binary
RUN wget http://ftp.us.debian.org/debian/pool/main/libs/libseccomp/libseccomp2_2.5.1-1_armhf.deb && dpkg -i libseccomp2_2.5.1-1_armhf.deb 
#RUN rm -rf /var/lib/apt/lists/*

# Set up the application
RUN git clone https://github.com/YasserAntonio/wger_arm.git 
RUN pip3 wheel --no-cache-dir --wheel-dir /usr/src/app/wheels -r /wger_arm/requirements.txt
RUN pip3 wheel --no-cache-dir --wheel-dir /usr/src/app/wheels -r /wger_arm/requirements_prod.txt


########
# Final
########
FROM yasserantonio/wger_base:armv7 
LABEL maintainer="Roland Geider <roland@geider.net>"

ARG DOCKER_DIR=wger_arm
EXPOSE 80


# Install dependencies
RUN apt-get update \
  && apt-get install --no-install-recommends -y \
    apache2 \
    cron \
    git\
    libapache2-mod-wsgi-py3\
    libjpeg-turbo8
 # && rm -rf /var/lib/apt/lists/*

# Configure apache
RUN git clone https://github.com/YasserAntonio/wger_arm.git 
RUN mv /wger_arm/* /home/wger
RUN mv /home/wger/extras/docker/apache/wger.conf /etc/apache2/sites-available/
RUN a2dissite 000-default.conf \
  && a2enmod headers \
  && a2ensite wger \
  && echo "ServerName localhost" >> /etc/apache2/conf-available/fqdn.conf \
  && a2enconf fqdn \
  && usermod -G wger www-data

# Configure cron
RUN mv /home/wger/extras/docker/apache/crontab /etc/cron.d/wger
RUN mv /home/wger/extras/docker/apache/venvwrapper /home/wger/venvwrapper
RUN mv /home/wger/extras/docker/apache/entrypoint.sh /home/wger/entrypoint.sh

RUN chmod 0644 /etc/cron.d/wger \
  && chmod 777 /home/wger /home/wger/entrypoint.sh \
  && touch /var/log/cron.log

COPY --from=builder /usr/src/app/wheels /wheels
COPY --chown=wger:www-data . /home/wger/src

# Set up the application
RUN ln -s /wger_arm/wger/static/CACHE /var/www

RUN rm -rf /home/wger && git clone https://github.com/wger-project/wger.git /home/wger
RUN npm config set strict-ssl false
RUN yarn config set "strict-ssl" false -g
WORKDIR /home/wger
RUN apt-get install libjpeg-turbo8
RUN python3 -m venv /home/wger/venv
RUN . /home/wger/venv/bin/activate \
    && pip install --upgrade pip \
    && pip install wheel \
    && pip install --no-cache /wheels/* \
    && python setup.py develop \
    && wger create-settings --database-path /home/wger/db/database.sqlite \
    && wger bootstrap


# Change permissions of some files and folders so the apache process
# can access them.
#RUN mkdir /home/wger/CACHE /home/wger/static/CACHE /home/wger/media \
#    && ln -s /home/wger/static/CACHE /home/wger/CACHE \
#    && chmod g+w /home/wger/static/CACHE \
RUN sed -i "/^MEDIA_ROOT/c\MEDIA_ROOT='\/home\/wger\/media'" settings.py \
    && echo STATIC_ROOT=\'/home/wger/static\' >> settings.py
#USER root
RUN apt-get remove build-essential -y \
    && apt autoremove -y \
    && chown :www-data -R /home/wger/db \
    && chmod -R 777 /home/wger \
    && chmod g+w /home/wger/db /home/wger/db/database.sqlite
ENTRYPOINT ["/home/wger/entrypoint.sh"]
