FROM ubuntu:14.04

RUN locale-gen en_US.UTF-8
RUN dpkg-reconfigure locales
ENV LANG en_US.UTF-8
ENV LC_ALL en_US.UTF-8

RUN apt-get update; apt-get -y upgrade; apt-get clean

RUN apt-get install -y git curl wget tar make; apt-get clean

RUN apt-get install -y sqlite3 libsqlite3-dev; apt-get clean

RUN apt-get install -y mysql-client libmysqlclient-dev; apt-get clean

RUN apt-get install -y postgresql-9.3 postgresql-client-9.3 libpq-dev; apt-get clean

# chruby
RUN mkdir /tmp/chruby && \
    cd /tmp && \
    curl https://codeload.github.com/postmodern/chruby/tar.gz/v0.3.9 | tar -xz && \
    cd /tmp/chruby-0.3.9 && \
    sudo ./scripts/setup.sh && \
    rm -rf /tmp/chruby

# ruby-install
RUN mkdir /tmp/ruby-install && \
    cd /tmp && \
    curl https://codeload.github.com/postmodern/ruby-install/tar.gz/v0.5.0 | tar -xz && \
    cd /tmp/ruby-install-0.5.0 && \
    sudo make install && \
    rm -rf /tmp/ruby-install

#default ruby
RUN ruby-install ruby 2.1.2 && \
    cp /etc/profile.d/chruby.sh /etc/profile.d/chruby-with-ruby-2.1.2.sh && \
    echo "chruby 2.1.2" >> /etc/profile.d/chruby-with-ruby-2.1.2.sh

#Bundler
RUN ["/bin/bash", "-l", "-c", "gem install bundler --no-ri --no-rdoc"]

#BOSH CLI
RUN ["/bin/bash", "-l", "-c", "gem install bosh_cli --no-ri --no-rdoc"]
