FROM ruby:2.2

ENV APP_HOME /app
ENV COMMENT_DATABASE_HOST comment_db
ENV COMMENT_DATABASE comments

RUN mkdir $APP_HOME

WORKDIR $APP_HOME

ADD Gemfile* $APP_HOME/
ADD . $APP_HOME

RUN apt-get update -qq && \
    apt-get install -y build-essential && \
    bundle install && \
    rm -rf /var/lib/art/lists/*

CMD ["puma"]
