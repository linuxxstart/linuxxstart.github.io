# Deploy de vms no vCenter e cluster swarm com ansible

![image](https://user-images.githubusercontent.com/55243431/71477680-0476e800-27ca-11ea-83eb-348437581d93.png)

Fala pessoal, apesar de não gostar de escrever e muito menos me vi tendo um blog, mas depois de ser incentivado pelo [Rafael Gomes](https://gomex.me/) a “botar a cara” e produzir para comunidade, lá vamos nós.

Como primeiro post do blog, acredito que será um grande aprendizado, mas também espero que o mesmo possa ajudar com o pequeno conteúdo técnico que estou aprendendo.

Vou fazer uma playbook do ansible para criação de máquinas virtuais no vcenter. Além da criação das vms, adicionei a instalação do cluster swarm. Vejo que esse conteúdo do vcenter é pouco explorado atualmente, hoje vemos muitos conteúdos de ansible para clouds. Mas, assim como eu trabalho em um empresa que utiliza o vmware, acredito que tem outro colegas na mesma situação.

### Requisitos necessário

### 1. Instalando ansible

Primeiramente, vamos precisar ter o ansible instalado. Vou demostrar a instalação do ansible no Debian, para outros sistemas, recomendo que siga a documentação oficial.

Adicione a linha abaixo no arquivo /etc/apt/sources.list:
```
deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main
```
Agora execute os comando:
```
$ sudo apt-key adv — keyserver keyserver.ubuntu.com — recv-keys 93C4A3FD7BB9C367
$ sudo apt update
$ sudo apt install ansible
```
### 2. Instale o módulo PyVmomi para conexão API do vcenter
```
sudo apt install python-pip
sudo pip install pyvmomi
```

### Criando Playbook, roles e vars

### 1. Criando variáveis

Estou criando um arquivo com as variáveis necessárias para acesso ao vcenter e do acesso ao vmware_tools da vms após as mesma serem criada.
```
$ vim group_vars/all

ansible_connection: vmware_tools
ansible_vmware_host: vcenter.teste.br
datacenter_name: "TESTE"
cluster_name: "CLUSTER_TESTE"
ansible_vmware_user: usuario@teste.br
ansible_vmware_password: senha
ansible_vmware_validate_certs: no
ansible_vmware_guest_path: datacenter_name/vm/{{ inventory_hostname }}
ansible_vmware_tools_user: root
ansible_vmware_tools_password: senha
docker_swarm_manager_ip: "192.168.0.1"
docker_swarm_manager_port: "2377"
```

### 2. Inventário de vms

Vamos criar um arquivo onde estarão os grupos “manager” e “worker” do docker. Também terá os nomes e ips que serão configurados nas vms assim que elas forem criadas e o datastore que onde serão alocadas.
```
$ vim vms_deploy[docker_swarm_manager]
ansible-ubuntu01 guest_custom_ip=’192.168.0.1' vsphere_datastore=’datastore01’
ansible-ubuntu02 guest_custom_ip=’192.168.0.2' vsphere_datastore=’datastore01’
[docker_swarm_worker]
ansible-ubuntu03 guest_custom_ip=’192.168.0.3' vsphere_datastore=’datastore01’
``` 

### 3. Roles

Vamos criar as pastas roles para separar cada ação que vamos fazer
```
$ mkdir -p roles/vms/tasks
$ mkdir -p roles/manager/tasks
$ mkdir -p roles/worker/tasks
```

### 4. Criando vms

Aqui vamos criar as vms no vcenter, trocar o hostname e instalar o docker.
```
$ vim roles/vms/tasks/main.yml

- name: "Criando vms"
    vmware_guest:
      hostname: "{{ ansible_vmware_host }}"
      username: "{{ ansible_vmware_user }}"
      password: "{{ ansible_vmware_password }}"
      validate_certs: False
      name: "{{ inventory_hostname }}"
      template: template-ubuntu18.04
      datacenter: "{{ datacenter_name }}"
      folder: /{{ datacenter_name }}/vm
      cluster: "{{ cluster_name }}"
      datastore: "{{ vsphere_datastore }}"
      hardware:
        memory_mb: 4096
        num_cpus: 6
        num_cpu_cores_per_socket: 3
        hotadd_cpu: True
        hotadd_memory: True
      networks:
      - name: nome_rede
        ip: "{{ guest_custom_ip }}"
        netmask: 255.255.255.0
        gateway: 192.168.0.1
        type: static
        dns_servers: 
          - 192.168.0.1
          - 192.168.0.2
        dns_suffix:
          - teste.br
      customization:
        domain: teste.br
        dns_servers:
          - 192.168.0.1
          - 192.168.0.2
        dns_suffix:
          - teste.br
      state: poweredon
      wait_for_ip_address: yes
    delegate_to: localhost
    
  - name: "Trocando o hostname"
    vmware_vm_shell:
      hostname: "{{ ansible_vmware_host }}"
      username: "{{ ansible_vmware_user }}"
      password: "{{ ansible_vmware_password }}"
      validate_certs: False
      datacenter: "{{ datacenter_name }}"
      folder: "/{{datacenter_name}}/vm"
      vm_id: "{{ inventory_hostname }}"
      vm_username: "{{ ansible_vmware_tools_user }}"
      vm_password: "{{ ansible_vmware_tools_password }}"
      vm_shell: "/usr/bin/hostnamectl"
      vm_shell_args: "set-hostname {{ inventory_hostname }} > /tmp/$$.txt 2>&1"
      wait_for_process: True
    delegate_to: localhost- name: "Instalando docker"
    vmware_vm_shell:
      hostname: "{{ ansible_vmware_host }}"
      username: "{{ ansible_vmware_user }}"
      password: "{{ ansible_vmware_password }}"
      validate_certs: False
      datacenter: "{{ datacenter_name }}"
      folder: "/{{datacenter_name}}/vm"
      vm_id: "{{ inventory_hostname }}"
      vm_username: "{{ ansible_vmware_tools_user }}"
      vm_password: "{{ ansible_vmware_tools_password }}"
      vm_shell: "/usr/bin/curl"
      vm_shell_args: "-fsSL https://get.docker.com | sh > /tmp/$$.txt 2>&1"
      wait_for_process: True
    delegate_to: localhost
```

### 5. Habilitando o Swarm

Aqui nós vamos configurar o swarm no primeiro hosts e depois habilitá-lo nos demais hosts do grupo manager
```
$ vim roles/manager/tasks/main.yml

- hosts: all
  gather_facts: False
  tasks:
  - name: Verifica se o Docker Swarm está habilitado
    shell: docker info
    changed_when: False
    register: docker_info  - name: Iniciar o swarm
    shell: docker swarm init --advertise-addr {{ docker_swarm_manager_ip }}:{{ docker_swarm_manager_port }}
    when: "docker_info.stdout.find('Swarm: active') == -1 and inventory_hostname == groups['docker_swarm_manager'][0]"  - name: Armazena o token de manager
    shell: docker swarm join-token -q manager
    changed_when: False
    register: docker_manager_token
    delegate_to: "{{ groups['docker_swarm_manager'] | first }}"
    run_once: true
    when: "docker_info.stdout.find('Swarm: active')"  - name: Adiciona os outros swarms Managers no cluster.
    shell: docker swarm join --token "{{ docker_manager_token.stdout }}" {{ docker_swarm_manager_ip}}:{{ docker_swarm_manager_port }}
    changed_when: False
    when: "docker_info.stdout.find('Swarm: active') == -1
    and docker_info.stdout.find('Swarm: pending') == -1
    and 'docker_swarm_manager' in group_names
    and inventory_hostname != groups['docker_swarm_manager'][0]"
```

### 6. Habilitando o worker

Agora vamos incluir os hosts do grupo worker no swarm.
```
$ vim roles/worker/tasks/main.yml
 
- name: Verifica se o Docker Swarm está habilitado.
    shell: docker info
    changed_when: False
    register: docker_info- name: Pega o token do worker.
    shell: docker swarm join-token -q worker
    changed_when: False
    register: docker_worker_token
    delegate_to: "{{ groups['docker_swarm_manager'] | first }}"
    run_once: true
    when: "docker_info.stdout.find('Swarm: active') == -1"- name: Adiciona o servidor de Worker no cluster.
    shell: docker swarm join --token "{{ docker_worker_token.stdout }}" {{ docker_swarm_manager_ip}}:{{ docker_swarm_manager_port }}
    changed_when: False
    when: "docker_info.stdout.find('Swarm: active') == -1
           and docker_info.stdout.find('Swarm: pending') == -1"
```

### 7. Agora, vamos criar o arquivo que vai chamar as roles
```
$ vim main.yml- name: Cirando vms

    hosts: all
    gather_facts: no
    roles:
      - vms  - name: Configurando Managers
    hosts: docker_swarm_manager
    roles:
      - manager
  - name: Configurando Workers
    hosts: docker_swarm_manager:docker_swarm_worker
    roles:
      - worker
```

### 8. Executando o playbook

Antes de executar o playbook, vamos criptografar nosso arquivo com usuário e senha do vcenter. Digite a senha que será necessária na execução do playbook.
```
$ ansible-vault encrypt group_vars/vmware-vars
```
Agora podemos rodar o nosso playbook
```
$ ansible-playbook -i vms_deploy main.yml --ask-vault-pass -K
```
![1*M3vcPPfyoeaSmx3xgU-jlw](https://user-images.githubusercontent.com/55243431/71477958-d5617600-27cb-11ea-8a19-22eea75414f9.gif)

https://github.com/linuxxstart/ansible-vmware-deploy

### Fontes

Tirei grande parte da criação do cluster swarm do mundodocker, do post do -

[Cristhian Bicca](https://www.mundodocker.com.br/author/cristhian/) , tive que adaptar algumas coisas referente ao vcenter, mas no final deu certo.

https://docs.ansible.com/ansible/latest/modules/vmware_guest_module.html

https://docs.ansible.com/ansible/latest/modules/vmware_vm_shell_module.html

https://docs.ansible.com/ansible/latest/plugins/connection/vmware_tools.html

https://www.mundodocker.com.br/cluster-de-docker-swarm-com-ansible/
)
