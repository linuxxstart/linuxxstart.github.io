# Cluster Elastic com Swarm e pipeline PaloAlto

![elk+docker(1)](https://user-images.githubusercontent.com/55243431/72045179-41e07880-3294-11ea-98f9-1dde4b29f5d9.png)
Neste post vou subir um cluster elastic, com Elasticsearch, Kibana e o Logstash (ELK). A stack da elastic é uma solução de análise de dados em tempo real, além das ferramentas acima tem o beats, que é um agente de envio de dados para o Elasticsearch ou para o Logstach.

Tenho testado o elastic stack há algumas meses para obter métricas do HAproxy e do firewall PaloAlto (vou demonstrar um pipeline), quando iniciei o teste eu subi o [repo](https://github.com/linuxxstart/cluster-elastic) com docker-compose.

A intençao agora é tentar subir como service no cluster swarm, para escalar principalmente  os data nodes. Segue a [stack no swarm](https://github.com/linuxxstart/elastic-cluster-swarm) e aguardo sugestões para melhoria da solução.

### 1. Arquivo instances.yml
Vamos criar um arquivo que será usado na criação dos certificados do elastic, responsáveis pelo comunicação segura entre os nodes.
![instances_01](https://user-images.githubusercontent.com/55243431/72038636-e5279280-3280-11ea-8b83-fa654fbb9057.png)
Como podemos ver na imagem acima, definimos dois nomes de serviços e os ips.

### 2. Arquivo create-certs.yml
Esta é o arquivo responsável por criar os certificados com os nomes dos serviços que definimos no arquivo anterior e vai gerar um volume onde estes certificados serão armazenados.
![certs_01](https://user-images.githubusercontent.com/55243431/72039104-911dad80-3282-11ea-90ea-720f8053a72c.png)

```
docker-compose -f create-certs.yml run --rm create_certs
```
O comando acima vai gerar um container apenas para criar os certificados passando o arquivo instances.yml e gerando o volume elk_certs no docker. Este nome elk está definido do arquivo .env (COMPOSE_PROJECT_NAME=elk).

![certs_02](https://user-images.githubusercontent.com/55243431/72039361-90394b80-3283-11ea-9f8e-3a06ac73f7d8.png)

### 3. Arquivo docker-compose.yml

Agora vamos falar de algumas partes do arquivo de criação da stack de serviços.

### 3.1. Master node
![compose_01](https://user-images.githubusercontent.com/55243431/72039983-d1325f80-3285-11ea-9bb0-a94317b0ea93.png)

Definindo as configuração do arquivo elasticsearch.yml.

* **node.name:** Definindo o nome que do node elastic, estou usando o nome do service mais a task que corresponderá ao numero da réplica.

* **cluster.name:** Definir o nome do cluster.
* **node.master:** Será um master node.
* **discovery.seed_hosts:** Faz uma varredura na na porta 9300 pelos node que devem ser master. Usei a tasks.master.
* **cluster.initial_master_nodes:** Definir os node master. Ainda estudando uma solução para varedurada da rede, quando usei a tasks.master o cluster não subiu.
* **xpack.monitoring.collection.enabled:** Habilita o monitoramento do cluster.
* **xpack.license.self_generated.type:** Habilita a licença básica.
* **xpack.security.enabled:** Habilitando a comunicação entre os node de forma segura e com isso é habilitado a autenticação no kibana e permitindo perfis de acesso.
* **volumes:** Aqui defino um volume para a criação dos arquivo de configução e a montagem do volume dos certificados.

As configurações do sevice data node são parecidas, mudando apenas as nomeclaturos do service e do certificado.

### 3.2. Redes e Volumes
![compose_02](https://user-images.githubusercontent.com/55243431/72041540-9b43aa00-328a-11ea-9d76-0434a9686fa2.png)

Criando uma rede overlay para o cluster swarm, definindo o volume "elk_certs" como externo e os volumes "master" e "data" serão criados de acordo com o nome do serviço o número da réplica.
![volumes_01](https://user-images.githubusercontent.com/55243431/72041808-43f20980-328b-11ea-8dc5-5f73b5e8386b.png)

Agora basta fazer o deploy da stack que o cluster vai subir. :pray: :crossed_fingers:
```
docker stack deploy -c docker-compose.yml elk
```
### 4. Importar tamplate 
Vamos copiar e depois impontando index templates do PaloAlto no Elasticsearch.

``` 
docker cp traffic.json elk_master.1:/
docker cp threat.json elk_master.1:/
docker exec -ti elk_master.1 curl -XPUT -u elastic:Senha123 http://127.0.0.1:9200/_template/panos-traffic?pretty -H 'Content-Type: application/json' -d@/traffic.json
docker exec -ti elk_master.1 curl -XPUT -u elastic:Senha123 http://127.0.0.1:9200/_template/panos-threat?pretty -H 'Content-Type: application/json' -d@/threat.json
```
### 5. PaloAlo

Configuar o envio de syslog do firewall para o Logstash.
![paloalto_02](https://user-images.githubusercontent.com/55243431/72042354-127a3d80-328d-11ea-8ff6-979dea5075ee.png)
* porta UDP 5514
* format BSD
* facility LOG_USER

![paloalto_01](https://user-images.githubusercontent.com/55243431/72042338-02625e00-328d-11ea-8016-e0824633e2d1.png)

### 6. Kibana
Agora vamos acessar o Kibana e configurar os index patterns e importar os dashboard.
O usuário de acesso é elastic.
![kibana_01](https://user-images.githubusercontent.com/55243431/72042684-f4f9a380-328d-11ea-8237-51e6a10bfc83.png)

### 6.1 Index patterns

Clique em Management > index patterns e depois create
![index_patterns_01](https://user-images.githubusercontent.com/55243431/72042895-7fda9e00-328e-11ea-8174-9bb149aed636.png)

Agora vamos criar os index:
* panos-system-*
* panos-threat-*
* panos-config-*
* panos-traffic-*

![index_patterns_02](https://user-images.githubusercontent.com/55243431/72043082-00010380-328f-11ea-922d-0de4575882b9.png)

![index_patterns_03](https://user-images.githubusercontent.com/55243431/72043102-0c855c00-328f-11ea-8364-054ff7b7c5f0.png)

### 6.2 Importando dashboard
Clique em Management > Saved objects e depois em import
![danshboar_01](https://user-images.githubusercontent.com/55243431/72043371-c11f7d80-328f-11ea-959b-da65c4e23249.png)

### 7. Dashboard
![dashboard_02](https://user-images.githubusercontent.com/55243431/72043701-95e95e00-3290-11ea-80d7-3882b0b20efa.png)

![dashboard_03](https://user-images.githubusercontent.com/55243431/72043719-a3064d00-3290-11ea-8ae4-690b7894da4b.png)



### Fontes
* https://github.com/sm-biz/paloalto-elasticstack-viz
* https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster
* https://github.com/elastic/elasticsearch-docker/issues/91
* https://www.elastic.co/pt/blog/configuring-ssl-tls-and-https-to-secure-elasticsearch-kibana-beats-and-logstash
* https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-tls-docker.html



