# SOC infrastructure

sysctl -w vm.max_map_count=262144
$ docker-compose -f generate-indexer-certs.yml run --rm generator
docker-compose up -d --build

ports to allow:
```
514/udp
1514/tcp
1515/tcp
55000/tcp
80/tcp
443/tcp
666/tcp (ssh)
```