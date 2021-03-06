FROM mcr.microsoft.com/oryx/build:20191216.3 as main

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update \
  && apt-get install -y libreadline-dev bzip2 build-essential libssl-dev zlib1g-dev libpq-dev libsqlite3-dev \
  curl patch gawk g++ gcc git make libc6-dev patch libreadline6-dev libyaml-dev sqlite3 autoconf \
  libgdbm-dev libncurses5-dev automake libtool bison pkg-config libffi-dev bison libxslt-dev \
  libxml2-dev --no-install-recommends wget default-libmysqlclient-dev



RUN apt-get install -y libssl1.0-dev

# Install dependencies
RUN apt-get update \
  && apt-get install -y openssh-client --no-install-recommends \
  && apt-get install -y vim tree --no-install-recommends \
  && apt-get install -y tcptraceroute \
# Install Squashfs tools for KuduLite build
  && apt-get install -y squashfs-tools \
  && wget -O /usr/bin/tcpping http://www.vdberg.org/~richard/tcpping \
  && chmod 755 /usr/bin/tcpping \
  && apt-get install -y clamav \
  && freshclam

# Enable SSH for Kudu Console
RUN apt-get install -y ssh \
   && sed -i '/^#Port* /s/^#//' /etc/ssh/sshd_config \
   && sed -i '/^#PermitRootLogin* /s/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config \
   && sed -i '/^#PrintLastLog* /s/^#PrintLastLog yes/PrintLastLog no/' /etc/ssh/sshd_config \
   && chmod -R 0644 /etc/update-motd.d/ \
   && echo "root:Docker!" | chpasswd

COPY webssh.zip ssh /tmp/

# Install Kudu
RUN mkdir -p /opt/Kudu/local \
  && chmod 755 /opt/Kudu/local \
  && apt-get update \
  && apt-get  install -y unzip \
# Install pm2 and pm2-logrotate
  && mkdir -p /home/LogFiles \
# Install webssh
  && mkdir /opt/webssh \
  && unzip /tmp/webssh.zip -d /opt/webssh \
# Replace ssh with wrapper script for CIFS mount permissions workaround
  && mv /usr/bin/ssh /usr/bin/ssh.original \
  && mv /tmp/ssh /usr/bin/ssh \
  && chown root:root /usr/bin/ssh \
  && chmod 755 /usr/bin/ssh \
  && chmod -R 777 /home \
  && rm -rf /tmp/*

ENV DOTNET_RUNNING_IN_CONTAINER=true

# Enable correct mode for dotnet watch (only mode supported in a container)
ENV DOTNET_USE_POLLING_FILE_WATCHER=true

# Skip extraction of XML docs - generally not useful within an image/container - helps performance
ENV NUGET_XMLDOC_MODE=skip

RUN dotnet tool install -g --version 2.2 dotnet-aspnet-codegenerator
ENV PATH=$PATH:/root/.dotnet/tools

RUN echo September Release.....
#Install Kudu
RUN cd /tmp \
    && git clone https://github.com/purva-vasudeo/KuduLite.git \
    && cd ./KuduLite \
    && git checkout user/puvasude/staticsite \
    && git log --format="%H" -n 1 > /kudu_commit.log \
    && cd ./Kudu.Services.Web \
    && benv dotnet=2.2 dotnet publish -c Release -o /opt/Kudu \
    && chmod 777 /opt/Kudu/Kudu.Services.Web.dll \
    && rm -rf /tmp/* \
    && chmod a+rw /var/nuget \
    && find /var/nuget -type d -exec chmod 777 {} \;

COPY startup.sh /

RUN chmod 777 /startup.sh

RUN benv node=9 npm=6 npm install -g kudusync
RUN benv node=9 npm=6 npm install pm2@latest -g

RUN ln -s /opt/nodejs/9/lib/node_modules/npm/bin/npm-cli.js /usr/bin/npm-cli.js
ENV PATH=$PATH:/opt/nodejs/9/bin

EXPOSE 8181

# Copy custom scripts
RUN mkdir /custom_scripts
COPY custom_scripts/* /custom_scripts/
RUN chmod 755 /custom_scripts/*

COPY deploy.bash.ruby.template /opt/Kudu/node_modules/kuduscript/lib/templates/deploy.bash.ruby.template
COPY deploy.bash.ruby.template /opt/Kudu/KuduConsole/node_modules/kuduscript/lib/templates/deploy.bash.ruby.template

# Patch, remove for future builds
RUN apt-get install -y libssl1.0-dev

ENTRYPOINT [ "/startup.sh" ]
CMD [ "1002", "kudu_group", "1001", "kudu_user", "localsite" ]
