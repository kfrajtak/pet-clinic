FROM fluent/fluentd:v1.9.2-1.0

COPY conf/fluent.conf /fluentd/etc/

USER root
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-document"]
USER fluent
