FROM grafana/tempo:latest

COPY ./infra/railway/tempo/tempo.yaml /etc/tempo.yaml

EXPOSE 3200 14268 9095 4317 4318 9411

CMD ["-config.file=/etc/tempo.yaml"]