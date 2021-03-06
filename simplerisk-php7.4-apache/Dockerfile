# Using dedicated PHP image with version 7.4 and Apache
FROM php:7.4-apache 

# Maintained by SimpleRisk
LABEL maintainer="Simplerisk <support@simplerisk.com>"

WORKDIR /var/www

# Installing apt dependencies
RUN apt-get update && \
    apt-get -y dist-upgrade && \
    apt-get -y install libldap2-dev \
                       libcap2-bin \
                       libonig-dev \
                       libcurl4-gnutls-dev \
                       supervisor \
                       mariadb-client
# Configure all PHP extensions 
RUN docker-php-ext-configure ldap --with-libdir=lib/x86_64-linux-gnu && \
    docker-php-ext-install ldap \
                           mbstring \
                           json \
                           mysqli \
                           pdo_mysql \
                           curl
# Setting up setcap for port mapping without root and removing packages 
RUN setcap CAP_NET_BIND_SERVICE=+eip /usr/sbin/apache2 && \
    apt-get -y remove libcap2-bin && \
    apt-get -y autoremove && \
    apt-get -y purge && \
    rm -rf /var/lib/apt/lists/*

# Copying all files
COPY ./app_setup/supervisord.conf /etc/supervisord.conf
COPY ./app_setup/foreground.sh /etc/apache2/foreground.sh
COPY ./app_setup/envvars /etc/apache2/envvars
COPY ./app_setup/000-default.conf /etc/apache2/sites-enabled/000-default.conf
COPY ./app_setup/default-ssl.conf /etc/apache2/sites-enabled/default-ssl.conf
COPY ./app_setup/entrypoint.sh /entrypoint.sh

# Start supervisor 
RUN service supervisor restart

# Configure Apache
RUN sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 5M/g' /usr/local/etc/php/php.ini-production
# Create SSL Certificates for Apache SSL
RUN echo $(< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-32}) > /tmp/pass_openssl.txt
RUN mkdir -p /etc/apache2/ssl/ssl.crt /etc/apache2/ssl/ssl.key
RUN openssl genrsa -des3 -passout pass:/tmp/pass_openssl.txt -out /etc/apache2/ssl/ssl.key/simplerisk.pass.key
RUN openssl rsa -passin pass:/tmp/pass_openssl.txt -in /etc/apache2/ssl/ssl.key/simplerisk.pass.key -out /etc/apache2/ssl/ssl.key/simplerisk.key
RUN rm /etc/apache2/ssl/ssl.key/simplerisk.pass.key /tmp/pass_openssl.txt
RUN openssl req -new -key /etc/apache2/ssl/ssl.key/simplerisk.key -out  /etc/apache2/ssl/ssl.crt/simplerisk.csr -subj "/CN=simplerisk"
RUN openssl x509 -req -days 365 -in /etc/apache2/ssl/ssl.crt/simplerisk.csr -signkey /etc/apache2/ssl/ssl.key/simplerisk.key -out /etc/apache2/ssl/ssl.crt/simplerisk.crt
# Activate Apache modules
RUN a2enmod rewrite ssl
RUN a2enconf security
RUN sed -i 's/SSLProtocol all -SSLv3/SSLProtocol TLSv1.2/g' /etc/apache2/mods-enabled/ssl.conf
RUN sed -i 's/#SSLHonorCipherOrder on/SSLHonorCipherOrder on/g' /etc/apache2/mods-enabled/ssl.conf
RUN sed -i 's/ServerTokens OS/ServerTokens Prod/g' /etc/apache2/conf-enabled/security.conf
RUN sed -i 's/#ServerSignature On/ServerSignature Off/g' /etc/apache2/conf-enabled/security.conf

# Download and extract SimpleRisk, plus saving version for database reference
RUN rm -rf /var/www/html && \
    VERSION=`curl -sL https://updates.simplerisk.com/Current_Version.xml | grep -oP '<appversion>(.*)</appversion>' | cut -d '>' -f 2 | cut -d '<' -f 1` && \
    curl -sL https://github.com/simplerisk/bundles/raw/master/simplerisk-$VERSION.tgz | tar xz -C /var/www && \
    echo $VERSION > /tmp/version

# Creating Simplerisk user on www-data group and setting up ownerships
RUN useradd -G www-data simplerisk
RUN chown -R simplerisk:www-data /var/www/simplerisk /etc/apache2 /var/run/ /var/log/apache2 && \
    chmod -R 770 /var/www/simplerisk /etc/apache2 /var/run/ /var/log/apache2 && \
    chmod 755 /entrypoint.sh /etc/apache2/foreground.sh

# Data to save
VOLUME /var/log/apache2
VOLUME /etc/apache2/ssl
VOLUME /var/www/simplerisk

# Using simplerisk user from here 
USER simplerisk 

# Setting up entrypoint
ENTRYPOINT [ "/entrypoint.sh" ]

# Ports to expose
EXPOSE 80
EXPOSE 443

# Start Apache 
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
