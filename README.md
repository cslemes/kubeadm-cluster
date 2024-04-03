---
title: ' "Instalando Kubernetes do Zero"'
date: '"2024-03-03T13:39:33.756Z"'
author: Cristiano Lemes
coverImage: '"https://media.dev.to/cdn-cgi/image/width=1000,height=420,fit=cover,gravity=auto,format=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fuczlklfhkyf9irwr5tnd.jpg"'
---
### Introdução

O Kubernetes emergiu como a principal plataforma de orquestração de containers, permitindo que organizações gerenciem e dimensionem aplicativos de maneira eficiente em ambientes de produção e desenvolvimento. Com sua arquitetura distribuída e recursos de automação robustos, o Kubernetes simplifica a implantação, o gerenciamento e a escalabilidade de aplicativos em containers.

Neste artigo, vamos explorar o processo de instalação do Kubernetes a partir do zero, utilizando o Kubeadm. O Kubeadm é uma ferramenta de linha de comando que facilita a configuração de clusters Kubernetes, oferecendo um método simplificado para inicializar e configurar um ambiente Kubernetes.
Estaremos fazendo toda instalando manualmente para fins didáticos, mas para esse tipo de instalação pode ser usado um gerenciador de configuração como Ansible para automatizar esse processo, que pode ser feito tanto em um servidor baremetal quanto em uma VM.


***Arquitetura do kubernetes***
<img src="https://kubernetes.io/images/docs/components-of-kubernetes.svg" alt="Componentes Kubernetes"/> 


Componentes do Controlplane:

- kube-apiserver: É o componente central do Kubernetes que expõe a API do Kubernetes. Todas as operações do cluster, como criação, atualização e exclusão de recursos, são realizadas através desta API.
- etcd: Um banco de dados chave-valor distribuído usado para armazenar o estado do cluster Kubernetes, incluindo configurações, metadados e informações sobre os nós e os pods.
- kube-scheduler: Responsável por agendar pods em execução nos nós do cluster. Ele considera requisitos de recursos, afinidades e restrições ao tomar decisões sobre onde os pods devem ser executados.
- kube-controller-manager: É responsável pela execução dos controladores do Kubernetes. Os controladores monitoram o estado do cluster e fazem ajustes para garantir que o estado desejado seja mantido. Exemplos de controladores incluem o controlador de replicação, o controlador de endpoints e o controlador de serviço.

Componentes do Node:

- kubelet: Agente que executa nos nós do cluster e é responsável por garantir que os containers estejam em execução em um nó. Ele se comunica com o kube-apiserver para receber instruções sobre quais pods devem ser executados e garante que os containers nos pods estejam saudáveis.
- kube-proxy: É um proxy de rede que executa no nó e mantém as regras de encaminhamento de rede. Ele gerencia o tráfego de rede para os pods no nó, permitindo que os mesmos se comuniquem entre si e com recursos externos.
- Container runtime: Um container runtime é uma parte essencial do ecossistema de contêineres. Ele é responsável por executar e gerenciar os containers. Ele provê isolamento de recursos, como cpu, memoria, redes e volumes, e faz a gestão do ciclo de vida do container. Ele é o componente reposnavel pela comunicação entre o container e o kernel.

***Tipos implantação do Kubernetes***

O método mais comum atualmente é utilizar uma distribuição de Kubernetes gerenciada, oferecida por provedores como Amazon, Google e Microsoft. No entanto, existem três abordagens principais para usar o Kubernetes.

Plataformas Gerenciadas: No Kubernetes gerenciado, você não terá controle sobre o control plane, que é o nó responsável por gerenciar o cluster Kubernetes. Isso simplifica a manutenção do cluster, porém limita algumas personalizações. Por exemplo, você não poderá atualizar a versão do Kubernetes por conta própria; essa tarefa fica a cargo do provedor, e você só terá acesso às versões que eles validaram.

Distribuiçoes Kubernetes: Existem distribuições que vêm empacotadas, que auxiliam desde da instalação da máquina, virtual ou baremetal, algumas tendo inclusive sistema operacional customizado para o kubernetes como é o caso do RKE, os instaladores vão fazer toda parte de instalação do cluster e configurar ferramentas auxiliares como helm e stacks de monitoramento, entregado o cluster pronto para uso no final.

Implantação Manual: Ao criar o cluster a partir do zero e instalar manualmente cada componente, você tem controle total sobre o cluster, podendo personalizá-lo e utilizar qualquer versão do Kubernetes. No entanto, isso implica em mais etapas de manutenção e uma maior responsabilidade sobre o cluster.

Kubernetes Gerenciado:
  - EKS Amazon Elastic Kubernetes Service
  - AKS Azure Kubernetes Service
  - GKE Google Kuberneste Engine 
  - DOKS DigitalOcean Kubernetes

Distribuições Kubernetes:
  - RKE Rancher Kubernetes Engine (OpenSource)
  - RedHat OpenShift (Comercial, mas tem a versão aberta chamada OKD)    
  - VMware Tanzu (Comercial)

Implantação Manual:
  - kubeadm: O que usarei neste artigo
  - kops: Ferramenta para criar o cluster de forma automatizada 
  - kubepray: Ferramenta que usa o ansioble para provisionar o cluster

***Configurações mínimas para o cluster:***

- SO Linux
- CPU: 2
- Memória: 2GB
- Conexão de rede entre todas as máquinas no cluster
- Swap desabilitado.
- Acesso ssh aos servidores

***Portas necessárias:***

Libere essas portas no firewall para comunicação entre os nós.

| Protocolo | Direção | Intervalo de Portas | Propósito                              | Utilizado por          |
|-----------|---------|---------------------|----------------------------------------|-------------------------|
| TCP       | Entrada | 6443                | Servidor da API do Kubernetes          | Todos                   |
| TCP       | Entrada | 2379-2380           | API servidor-cliente do etcd           | kube-apiserver, etcd    |
| TCP       | Entrada | 10250               | API do kubelet                         | kubeadm, Camada de gerenciamento |
| TCP       | Entrada | 10259               | kube-scheduler                         | kubeadm                 |
| TCP       | Entrada | 10257               | kube-controller-manager                | kubeadm                 |
| TCP       | Entrada |	10250	              | API do Kubelet	                       | kubelet, Camada de gerenciamento |
| TCP	      | Entrada |	30000-32767	        | Serviços NodePort	                     | Todos                   |
| TCP       | Entrada | 6783                | Weave Pod Network                      | Todos                   |
| UDP       | Entrada | 6783-6784           | Weave Pod Network                      | Todos                   |

***Requisitos de máquinas para o cluster.***

Para seguir este tutorial pode ser usado VM locais, ou máquinas de um provedor de cloud, estaremos usando três máquinas, mas dá para testar em apenas uma, lembrando que o controlplane por padrão não roda container de aplicações, apenas de serviços do control-plane, mas configuração que pode ser alterada facilmente usando *taints*.

***Configurando os nós do Cluster.***

Acessando cada nó do cluster, vamos iniciar instalando e configurando os pré-requisitos para rodar o kubernetes, estarei utilizando o Sistema Operacional Ubuntu 22.04.

1- Deslique o SWAP, Kubelet não funciona com SWAP Ativa, e remova sua entrada do arquivo */etc/fstab/*.
```bash
sudo swap off -a
```
2- Habilite os modulos do kernel necessários para o funcinamento do cluster, para isso vamos criar o arquivo k8s.conf em */etc/modules-load.d/*, depois use o *modprobe* para carregar os modulos sem ser necessario dar um reboot.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
3- Configurando parametros de rede, vamos criar o arquivo k8s.conf agora na pasta do *systcl* *etc/sysctl.d/* para que o linux possa visualizar o tráfego de redes, depois use o *sysctl* para aplicar as mudanças sem ser necessário dar um reboot.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

***Intalando binarios do kubernetes***

1- Atualize o repositorio *apt* e instale os pacotes basicos para baixar do repositorio oficial do kubernetes:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

2- Faça download da chave pública do Google cloud:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```

3- Adicione o repositório apt do Kubernetes:
```bash
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4- Atualize o apt e instale o kubelet, o kubeadm e o kubectl. Use o *hold* para fixar as versões para evitar problemas em atualizações automaticas.

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

***Instalando um container runtime, iremos utilizar o containerd.***

1- Instalando requisitos e chaves GPG do Docker

```bash	
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
2- Adicione o repositório a lista do *apt* e atualize os índices.

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

3- Instale o containerd

```bash
sudo apt-get install containerd.io
```

4- Gere arquivo de configuraça padrão do Containerd.

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml

```
5- Configure o systemd como cgroup driver, use o comando *sed* para alterar o arquivo gerado no passo anterior.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

```

6- Confirme que o serviço containerd está habilitado e reinicie-o para aplicar a alteração do arquivo.

```bash
sudo systemctl enable containerd
sudo systemctl restart containerd
```

***Criando o Cluster***

Essas etapas devem ser feitas no nó que funcionará como controlplane.

1- Inicie o controlplane usando o ``` kubeadm ```.

```bash	
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=<ip-da-maquina>
```

Há parametros opcionais que podem ser usados com o kubeadm, como *--apiserver-advertise-address* onde você espifica qual ip do nó vai ser usado pela api do controlplane, util se tiver mais de uma interface de rede configurado na maquina. Você consegue obter facilmente usando o comando *ip a*, tendo um ip publico e outro privado escolha o privado.

Output do kubeadm init

``` bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Ele vai imprimir na tela o comando para adicionar os nós do cluster, se você perder o token, você pode executar no servidor control-plane.

```bash
kubeadm token list
```

2- Crie o arquivo de configuração do kubernetes para gerenciar o cluster, apos executar o init o kubeadm gera as credenciais para o cluster em  ``` /etc/kubernetes/admin.conf ```, é necessarios por o conteudo deste arquivo  

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Agora já pode verificar o status do cluster

```bash	
kubectl get nodes

NAME                         STATUS     ROLES           AGE     VERSION
ubuntu-s-2vcpu-2gb-sfo3-02   NotReady   control-plane   5m30s   v1.29.2
```

3- Adicionando outros nós no cluster. Agora que o cluster está up, execute o comando ``` kubeadm join ``` com as especificaçãos obtidas no output da criação do cluster

```bash
  kubeadm join <control-plane-host>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Verificando novamente os nós, agora temos todos listados, mas ainda está com status NotReady, o kubernetes não possui nativamente gerencimanto de rede, por isso devemos instalar um plugin para suprir essa necessidade.

```bash	
kubectl get nodes

NAME                         STATUS     ROLES           AGE     VERSION
ubuntu-s-2vcpu-2gb-sfo3-01   NotReady   <none>          8s      v1.29.2
ubuntu-s-2vcpu-2gb-sfo3-02   NotReady   control-plane   5m30s   v1.29.2
ubuntu-s-2vcpu-2gb-sfo3-03   NotReady   <none>          7s      v1.29.2
```

4- Adicionando Plugin de Redes.

O kuberneste possui uma interface padronizada para que os plugins possam se integrar facilmente a ele, chamado CNI *container network interface*, com ele podemos escolher entre varias opções de plugins que se adequem a nossa necessidade.
Alguns dos principais plugins CNI usados em ambientes de containers, especialmente em clusters Kubernetes, incluem:

- Calico: Um plugin de rede de código aberto que oferece funcionalidades avançadas de rede, incluindo políticas de rede baseadas em identidade e suporte a BGP (Border Gateway Protocol) para escalabilidade e interoperabilidade.
- Flannel: Um plugin de rede simples e leve que cria uma rede sobreposta (overlay network) para conectar os containers em um cluster. Ele é popular por sua simplicidade e escalabilidade.
- Weave: Outro plugin de rede de sobreposição que cria uma rede virtual privada (VPN) entre os nós do cluster. Weave oferece suporte a funcionalidades como criptografia de ponta a ponta e descoberta automática de serviços.
- Cilium: Um plugin de rede e segurança que combina roteamento baseado em BPF (Berkeley Packet Filter) com política de segurança de camada 7. Ele fornece recursos avançados de segurança e observabilidade para containers e microsserviços.
- Kube-router: Um plugin de rede que integra o roteamento baseado em BGP diretamente no Kubernetes, permitindo o balanceamento de carga de entrada e saída do cluster.


Para esse tutorial estarei utilizando o weave, por ser simples, para ambiente produtivo avalie as demais opções, no inicio desse ano a Weave Works encerrou suas operações.

Instale o plugin usando o yaml.

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

---
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created

```

***Verificando o cluster e efetuando testes.***

1- Após a instalação do cluster os Status muda para Ready

```
kubectl get nodes
NAME                         STATUS   ROLES           AGE   VERSION
ubuntu-s-2vcpu-2gb-sfo3-01   Ready    <none>          33m   v1.29.2
ubuntu-s-2vcpu-2gb-sfo3-02   Ready    control-plane   43m   v1.29.2
ubuntu-s-2vcpu-2gb-sfo3-01   Ready    <none>          34m   v1.29.2
```

2- Checando os pods de sistema.

```bash
kubectl get pods -n kube-system
NAME                                                 READY   STATUS    RESTARTS        AGE
coredns-76f75df574-bsn9k                             1/1     Running   0               47m
coredns-76f75df574-mfpt2                             1/1     Running   0               47m
etcd-ubuntu-s-2vcpu-2gb-sfo3-02                      1/1     Running   0               48m
kube-apiserver-ubuntu-s-2vcpu-2gb-sfo3-02            1/1     Running   0               47m
kube-controller-manager-ubuntu-s-2vcpu-2gb-sfo3-02   1/1     Running   0               48m
kube-proxy-4vfvf                                     1/1     Running   0               47m
kube-proxy-4vfvf                                     1/1     Running   0               47m
kube-proxy-t67tm                                     1/1     Running   0               38m
kube-scheduler-ubuntu-s-2vcpu-2gb-sfo3-02            1/1     Running   0               47m
weave-net-g9m6f                                      2/2     Running   1 (5m20s ago)   5m28s
weave-net-rvt2v                                      2/2     Running   1 (5m21s ago)   5m28s
weave-net-g9m6f                                      2/2     Running   1 (5m22s ago)   5m28s
```

3- Criando um container para testar o cluster.

```
kubectl create deployment nginx-web --image nginx --replicas 3
```

4- Checando a criação dos container todos devem estar Running
```bash
kubectl get pods

NAME                         READY   STATUS    RESTARTS   AGE
nginx-web-5b757f798d-d9g2s   1/1     Running   0          24s
nginx-web-5b757f798d-pj57v   1/1     Running   0          24s
nginx-web-5b757f798d-z5bfx   1/1     Running   0          24s
```

***Conclusão***
Neste artigo, exploramos o processo de instalação do Kubernetes a partir do zero, utilizando o Kubeadm como ferramenta principal. O Kubernetes emergiu como a principal plataforma de orquestração de containers, oferecendo uma arquitetura distribuída e recursos de automação robustos para gerenciar e escalar aplicativos em ambientes de produção e desenvolvimento.

Ao longo do artigo, cobrimos os seguintes pontos:

- Componentes do Controlplane e dos Nodes do Kubernetes, destacando suas funções e importância dentro do ecossistema do Kubernetes.
- Exploramos os diferentes tipos de implantação do Kubernetes, desde plataformas gerenciadas até implantações manuais, destacando as vantagens e considerações de cada abordagem.
- Especificamos as configurações mínimas e as portas necessárias para configurar um cluster Kubernetes.
- Detalhamos o processo de configuração dos nós do cluster, incluindo a desativação do SWAP, a instalação de binários do Kubernetes e a configuração do Container Runtime.
- Demonstrações passo a passo para criar um cluster Kubernetes usando Kubeadm, desde a inicialização do controlplane até a adição de nós adicionais.
- Finalmente, instalamos e configuramos um plugin de rede, essencial para que os pods possam se comunicar entre si e com recursos externos.

Em resumo, o Kubernetes oferece uma base sólida para implantar, gerenciar e escalar aplicativos em containers de maneira eficiente e escalável. Com o conhecimento adquirido neste artigo, os administradores de sistemas e desenvolvedores estão equipados para iniciar e gerenciar clusters Kubernetes, seja para ambientes de desenvolvimento, teste ou produção.



### Referências:

- [Implantar um cluster usando Terraform e AKS]("https://dev.to/cslemes/criando-um-cluster-aks-4f76").
- [Componentes do Kubernetes]("https://kubernetes.io/docs/concepts/overview/components/")
- [Amazon EKS]("https://aws.amazon.com/pt/eks/")
- [Azure AKS]("https://azure.microsoft.com/pt-br/products/kubernetes-service")
- [Google GCP]("https://cloud.google.com/kubernetes-engine?hl=pt-BR")
- [Digital Ocean DOKS]("https://digitalocean.com/products/kubernetes")
- [Instalando o Kubeadm](https://kubernetes.io/pt-br/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Criando um cluster com Kubeadm]("https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/")
- [Portas e Protocolos](https://kubernetes.io/pt-br/docs/reference/ports-and-protocols/)
- [kOps]("https://kops.sigs.k8s.io/"")
- [Kubspray](https://kubespray.io/)
- [Docker]("https://docs.docker.com/engine/install/ubuntu/")
- [Containerd]("https://containerd.io")
- [Network Plugins]("https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/")
- [Weave GitHub]("https://github.com/weaveworks/weave")
- [Encerramento Weave Works]("https://thenewstack.io/end-of-an-era-weaveworks-closes-shop-amid-cloud-native-turbulence/")
[Cover Image](https://www.freepik.com/free-vector/building-construction-workers-isometric-banner_3887737.htm#fromView=search&page=1&position=3&uuid=7258b4b7-e524-4ac7-b10d-839b9ce544b5)



