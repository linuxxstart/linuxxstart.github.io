# HAproxy + Consul com Docker Swarm


Fala pessoal,

Hoje vou falar de um pequeno projeto que fiz para transferir uma aplicação que estava em vms para container.

Utilizamos o HAproxy como balanceador de carga para essas aplicações e para manter a compatibilidade com a arquitetura já em funcionamento eu busquei uma forma de integrar o balanceador com a solução de micros serviços do docker. Apartir da versão 1.8 do HAproxy (Estamos utilizando a 1.9) houve um integração com o serviço consul.

O [consul](https://www.consul.io/) é uma ferramenta de descoberta de micros serviços, entre outras funções. Ele será usado nesta solução como DNS, ele fornecerá os ips e portas dos serviços que criaremos no docker para o HAproxy.
![imagem](https://user-images.githubusercontent.com/55243431/71564806-631bba80-2a85-11ea-9e57-403aa0878ce5.png)

[Arquivos do projeto](https://github.com/linuxxstart/docker-haproxy-consul)


### 1. Configurações HAproxy
![haproxy-arquivos](https://user-images.githubusercontent.com/55243431/71564940-8c3d4a80-2a87-11ea-8c65-f042de4d7a3b.png)

A imagem acima nos mostrar os arquivos que utilizei. Criei duas aplições (ou sites) para os nossos micros serviços: meusite.com.br e web.local.br. Por isso criei os arquivos de certificados, uma página de manutenção e o arquivo de configuração.

```
frontend http
    bind *:80
	##----------Linha para redirecionar www para nome do host-------##
    http-request redirect prefix http://%[hdr(host),regsub(^www\.,,i)] code 301 if { hdr_beg(host) -i www. }
    ##--------------------------------------------------------------##
	http-request redirect scheme https if !{ ssl_fc }

frontend https
    bind *:443 ssl crt /usr/local/etc/haproxy/ssl/
    

	## -----------ACLs dominios dos sistemas-----------##
    acl meusite hdr_end(host) -i meusite.com.br
    acl web hdr_end(host) -i web.local.br
    ##-------------------------------------------------##	

	##-------------Pagina de manutencao----------------##
    acl all_ips src 0.0.0.0/0
    #acl rede_local src 192.168.0.0/24
	#use_backend manutencao if all_ips meusite !rede_local 
    ##-------------------------------------------------##

	##------------ACLs de redirecionamento-------------##
	acl raiz path -i /
	##-------------------------------------------------##


	##--------Redirecionamento de URLs-----------------##
    http-request redirect code 301 location https://meusite.com.br/teste1 if meusite raiz
    http-request redirect code 301 location https://web.local.br/site if web raiz
    ##-------------------------------------------------##

	##-------Redireciona o dominio para o sistema------##
    use_backend meusite if meusite
    use_backend web if web
    ##-------------------------------------------------##

resolvers consul
  nameserver consul consul:8600
  accepted_payload_size 8192
  hold valid 5s


backend meusite
    
	option httpchk GET /teste1/check.html
    http-check expect status 200
	acl meusite path_beg,url_dec -m beg -i /teste1 /test2 /teste3
    http-request deny if !meusite
    dynamic-cookie-key MYKEY
    cookie SRVID insert dynamic
    balance roundrobin
    server-template meusite 4 _meusite._tcp.service.consul resolvers consul resolve-opts allow-dup-ip resolve-prefer ipv4 check

backend web
    
	option httpchk GET /site/index.html
    http-check expect status 200
	acl meusite path_beg,url_dec -m beg -i /site
    http-request deny if !meusite
    dynamic-cookie-key MYKEY
    cookie SRVID insert dynamic
    balance roundrobin
    server-template web 4 _web._tcp.service.consul resolvers consul resolve-opts allow-dup-ip resolve-prefer ipv4 check

backend manutencao
	mode http
	errorfile  503 /usr/local/etc/haproxy/errors/manutencao.html
```
Coloquei algumas configurações acima que queria destacar.

### 1.1. frontend http
* redireciono o acesso www para o nome do host. 
* regra para redirecionar todo acesso http para https (frontend https).

### 1.2. frontend https
Onde estão as configurações do serviços ou sites que iremos usar. 

* definir uma pasta para os certificados dos múltiplos domínios.
* acl com domínios das aplicações.
* acl para uma página de manutenção das aplicações.
* acl com o path raiz (/) do site.
* acl redirecionar urls, onde defino uma urls para caso for do domínio “a” no path raiz.
* acl que redireciona para o backend da aplicação.
 
### 1.3. consul
* definindo o servidor consul como dns de consulta para as aplicações.

### 1.4. backend
* definir uma página de checagem para verificar se a aplicação está no ar.
* acl para negar o acesso aos subdiretórios que não existem na aplicação.
* acl para persistência de sessão dinâmica.
* acl para o tipo de balanceamento de carga.
* onde definimos o nome da aplicação, o número instâncias ou replicas de container , a entrada dela no consul e a opção de checagem.  

### 1.5. backend manutenção 
* em caso de erro 503 a página de manutenção será exibida.

### 2. Subindo a stack

Criar uma rede no swarm e depois subir a stack
```
docker network create --driver overlay --attachable rede_proxy
docker stack deploy -c docker-compose.yml nome_da_stack
```
* os serviços mapeados pelo consul.

![consul_01](https://user-images.githubusercontent.com/55243431/71587958-8171cc80-2afe-11ea-8d0b-461747dcdf76.png)

![consul_02](https://user-images.githubusercontent.com/55243431/71587976-90587f00-2afe-11ea-8b08-23672e648bfd.png)

* a página de status do HAproxy com os containers mapeados pelo consul up e a quantidades possíveis que definimos do arquivo de conf.

![haproy_01](https://user-images.githubusercontent.com/55243431/71588017-b5e58880-2afe-11ea-91e0-b4b14ca274fa.png)



* e aqui temos nossas aplicações sendo acessadas.


![web_01](https://user-images.githubusercontent.com/55243431/71588045-cdbd0c80-2afe-11ea-8cb5-b8dfb66adfe5.png)

![meusite_01](https://user-images.githubusercontent.com/55243431/71588058-da416500-2afe-11ea-9581-f8110dc3e5c6.png)

