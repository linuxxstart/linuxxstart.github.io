# Packer Ansible Vmware Kubernetes


Fala ai galera, tudo tranquilo?

No ano passado (como se fosse há muito tempo atrás rs), no dia 19 de dezembro de 2019 o canal [LIUNXtips](https://www.youtube.com/channel/UCJnKVGmXRXrH49Tvrx5X0Sw) #vai, falou sobre o packer da hashicorp. Eu não conhecia essa ferramenta sensacional que cria imagens de qualquer tipo e que vai me ajudar muito no meu ambiente VMware para gerar os meus templates.

Sem mais delongas, vou mostrar o código que eu criar para gerar templates de ubuntu server com docker e kubernetes instalado.


### 1. Instalando packer

Tive um erro com a versão mais recente do packer e tivesse que usar a 1.4.5 para rodar sem erros. Existem várias maneiras de se instalar o packer segue a [documentação](https://www.packer.io/downloads.html).

```
$ wget https://releases.hashicorp.com/packer/1.4.5/packer_1.4.5_linux_amd64.zip
$ unzip Downloads/packer_1.4.5_linux_amd64.zip && sudo mv packer /usr/local/bin/
$ packer version
Packer v1.4.5
```

Agora vamos instalar o plugin para criação remota de imagens de máquinas virtuais no VMware vSphere
```
$ wget https://github.com/jetbrains-infra/packer-builder-vsphere/releases/download/v2.3/packer-builder-vsphere-iso.linux
$ chmod +x Downloads/packer-builder-vsphere-iso.linux && sudo mv Downloads/packer-builder-vsphere-iso.linux /usr/local/bin/
```

### 2. Criando o código para buildar a imagem

Repositório [packer-vmware](https://github.com/linuxxstart/packer-vmware)

Temos o arquivos:
*  **ubuntu-18.04-kubernetes.json** - Temos definidos a conexção com VMware e as especificações de hardware para criação do template.
* **preseed.cfg** - Arquivo onde definimos as estratégias de formatação do disco, instalação de pacotes e usuários e senhas. 
* **kubernetes.sh** - Script para adicionar epositórios, atualizar o sistema e instalar os pacotes.


Vamos validar o código para ver se o plugin instalado foi reconhecido e depois vamos buildar a imagem.
```
$ packer validate ubuntu-18.04-kubernetes.json
$ packer build ubuntu-18.04-kubernetes.json
```
![packer_build_01](https://user-images.githubusercontent.com/55243431/72988405-f74e1880-3dca-11ea-9011-210a8bd2f1a1.png)


![packer_build_02](https://user-images.githubusercontent.com/55243431/72988448-06cd6180-3dcb-11ea-8e39-e485e7c14a35.png)


### 3. Plus ansible e cluster kubernetes

Editei meu código de ansible que criava um cluster docker swarm para fazer subir as máquinas virtuais no VMware com cluster kubernetes. 

Então depois de criar o template ubuntu-kubernetes, vamos executar o ansible para criar 3 máquinas virtuais com um cluster kubernetes. 

Repositório [Cluster Swarm](https://github.com/linuxxstart/ansible-vmware-deploy).

Repositório [Cluster Kubernetes](https://github.com/linuxxstart/ansible-cluster-kubernetes).

```
$ ansible-playbook -i vms_deploy main.yml --ask-vault-pass -K
```
![ansible_kube_01](https://user-images.githubusercontent.com/55243431/72989717-92e08880-3dcd-11ea-8828-355616292276.png)

![ansible_kube_02](https://user-images.githubusercontent.com/55243431/72989823-c91e0800-3dcd-11ea-85c9-5a4aa33f709e.png)


![ansible_kube_03](https://user-images.githubusercontent.com/55243431/72992301-4ba8c680-3dd2-11ea-8998-81c862b19889.png)








# Fontes
https://cloud.vmware.com/community/2019/11/12/infrastructure-code-hashicorp-packer-vmware-vmware-cloud-aws/
https://computingforgeeks.com/how-to-install-and-use-packer/

https://www.thehumblelab.com/automating-ubuntu-18-packer/#_