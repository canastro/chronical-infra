FROM prom/prometheus:latest

COPY ./railway/prometheus/prometheus.yml /etc/prometheus/prometheus.yml

EXPOSE 9090

# set the entrypoint command
USER root
ENTRYPOINT [ "/bin/prometheus" ]
CMD        [ "--config.file=/etc/prometheus/prometheus.yml", \
             "--storage.tsdb.path=/prometheus", \
             "--storage.tsdb.retention.time=30d", \
             "--web.console.libraries=/usr/share/prometheus/console_libraries", \
             "--web.console.templates=/usr/share/prometheus/consoles", \
             "--web.external-url=http://localhost:9090", \
             "--web.enable-otlp-receiver", \
             "--web.enable-remote-write-receiver", \
             "--enable-feature=exemplar-storage", \
             "--enable-feature=native-histograms", \
             "--log.level=debug"]