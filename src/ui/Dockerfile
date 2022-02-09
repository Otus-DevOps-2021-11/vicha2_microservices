FROM ubuntu:16.04

ENV APP_HOME /app
ENV POST_SERVICE_HOST post
ENV POST_SERVICE_PORT 5000
ENV COMMENT_SERVICE_HOST comment
ENV COMMENT_SERVICE_PORT 9292

RUN mkdir $APP_HOME

WORKDIR $APP_HOME

ADD Gemfile* $APP_HOME/
ADD . $APP_HOME

RUN apt-get update && \
    apt-get install -y ruby-full ruby-dev build-essential && \
    gem install bundler --no-ri --no-rdoc && \
    bundle install && \
    rm -rf /var/lib/art/lists/*

CMD ["puma"]
