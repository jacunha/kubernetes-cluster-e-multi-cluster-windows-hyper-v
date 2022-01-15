# Como criar um cluster ou multi-cluster Kubernetes localmente, em uma máquina Windows, utilizando Hyper-V e Minikube?
Criei este guia com a intenção de, ao mesmo tempo que continuo desenvolvendo meus conhecimentos, também contribuo com o conhecimento que venho absorvendo de outros gigantes, que vem compartilhando muito conteúdo bom e de maneira gratuita.

Registro aqui meus agradecimentos para:
 
- **Jeferson Fernando** [![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/badtux_.svg?style=social&label=@badtux_)](https://twitter.com/badtux_)
 
- **Marcelo Andrade** [![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/marcelo_devsres.svg?style=social&label=@marcelo_devsres)](https://twitter.com/marcelo_devsres/)
 
- **Caio Delgado** [![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/marcelo_devsres.svg?style=social&label=@caiodelgadonew)](https://twitter.com/caiodelgadonew/)
 
Para quem esta procurando conteúdo em **DevOps**, **SRE** e **IaC**, estas são as pessoas que você deve acompanhar de perto pelas redes.
 
---
## Antes de começar
O foco aqui é criar um ambiente para estudar Kubernetes e, se se você tem apenas um computador com Windows, este guia é para você, agora se você já possui um sistema operacional Linux, recomendo que você treine o Kubernetes diretamente no Linux e descarte este guia ou, se quiser conhecer como as coisas funcionam no Windows será um prazer compartilhar.
 
Dito isto, neste guia usaremos o mínimo de programas externos, apenas o **minikube** e o **kubectl**, todo o resto será feito nativamente no sistema operacional Windows.
 
> Todas as ações serão executadas via **PowerShell**, como boa parte dos comando requerem elevação das permissões, execute o terminal como a opção _**Executar como administrador**_.
 
---
## Preparando o Sistema Operacional Host (Windows)
 
Cada nó kubernetes utilizará uma máquina virtual que será provisionada utilizando o virtualizador de hardware nativo do Windows, o **Hyper-V**, que não vem habilitado por padrão.
 
### Habilitando o Hyper-V via PowerShell
> Após a execução deste comando, o computador será reiniciado.
 
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```
 
---
## Instalando minicube e kubectl
Para facilitar a gestão dos binários ```minikube``` e ```kubectl```, colocaremos os dois binários em uma única pasta e adicionaremos a localização desta pasta à variável de ambiente **Path**, para que os binários possam ser executados de qualquer local no sistema apenas pelo seu nome.
 
### Criando pasta ```minikube``` em ```C:```
```powershell
New-Item -Path 'c:\' -Name 'minikube' -ItemType Directory -Force
```
 
### Adicionando ```c:\minikube``` a variável de ambiente "Path"
```powershell
$oldPath = [Environment]::GetEnvironmentVariable('Path', [EnvironmentVariableTarget]::Machine)
 
if ($oldPath.Split(';') -inotcontains 'C:\minikube'){ `
  [Environment]::SetEnvironmentVariable('Path', $('{0};C:\minikube' -f $oldPath),
  [EnvironmentVariableTarget]::Machine) `
}
```
 
O primeiro comando, armazena o conteúdo atual da variável de ambiente "**Path**", para usar no segundo comando, que verifica se o item que queremos adicionar já existe e, caso o item não exista, ```c:\minikube``` será adicionado a variável de ambiente "**Path**"
 
### Baixando os binários minikube e kubectl
Para utilizarmos o minikube e kubectl, iremos baixar os dois binários diretamente em ```c:\minikube```.
 
### Baixando a última versão do minikube
```powershell
Invoke-WebRequest -OutFile 'c:\minikube\minikube.exe' -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' -UseBasicParsing
```
 
Quando provisionei meu primeiro nó kubernetes, percebi que não foi baixado a ultima versão estável do kubernetes, para garantir que usaremos a última versão estável do kubernetes, iremos carregar, na variável ```$kubeVersion```, a última versão estável do kubernetes.
 
### Consultando a última versão estável do kubernetes
```powershell
$kubeVersion = (Invoke-WebRequest 'https://dl.k8s.io/release/stable.txt').Content
Write-Output $kubeVersion
```
 
### Baixando o kubectl
> Aqui utilizaremos a variável ```$kubeVersion```, que carrega a versão atual do kubernetes para garantir que ambos kubernetes e kubectl estejam na ultima versão estável, caso você tenha fechado o seu terminal, você deve executar o comando anterior novamente.
```powershell
Invoke-WebRequest -OutFile 'c:\minikube\kubectl.exe' -Uri "https://dl.k8s.io/release/$kubeVersion/bin/windows/amd64/kubectl.exe" -UseBasicParsing
```
 
Neste ponto, você pode abrir um novo terminal e testar tanto o comando ```minikube``` como o ```kubectl```.
 
---
## Criando um cluster ou multi-cluster kubernetes
Aqui, criarei um cluster de nó único, que pode ser utilizado para estudos mesmo em uma máquina com poucos recursos e mais dois clusters kubernetes (lab-k8s-dev e lab-k8s-prod) idênticos para trabalhar em multi-cluster.
 
Aqui há um ponto que difere uma implantação do Kubernetes no Hyper-V em relação ao VirtualBox, no Hyper-V, precisaremos criar um _**virtual switch** para fazer as conexões externas com o nó, ou nós do cluster Kubernetes.
 
Para isto, iremos associar este _**virtual switch**_ a uma interface física de rede da nossa máquina host (nosso computador).
 
### Obtendo o nome da interface física de rede
Seguindo o critério de "executar todas as ações via terminal", precisamos identificar qual interface física de rede usaremos, identificando o seu ID para usar na criação do _**virtual switch**_.
 
```powershell
(Get-NetAdapter).Name
```
Exemplo de saída:
```
Ethernet
Conexão de Rede Bluetooth
Wi-Fi
```
Neste guia, usarei a interface de ID "**Wi-Fi**"):
 
> O sistema irá criar uma bridge entre o **virtual switch** e a interface física escolhida mas, por algum motivo (bug) que não sei explicar, após a criação desta bridge, a banda da minha interface física Wi-Fi ficou limitada a 8 Mbp/s.
Se isto ocorrer com você, apenas desative e ative a bridge ("Ponte de Rede", no Windows) e este bug será corrigido.
 
```powershell
(Get-NetAdapter).Name
```
Saída:
```
Ethernet
Ponte de Rede
Conexão de Rede Bluetooth
vEthernet (Minikube Switch)
Wi-Fi
```
```powershell
Disable-NetAdapter -Name 'Ponte de Rede' -Confirm:$false
Start-Sleep -Seconds 5
Enable-NetAdapter -Name 'Ponte de Rede' -Confirm:$false
```
### Criando o Virtual Switch no Hyper-V
```powershell
New-VMSwitch -name 'Minikube Switch' -NetAdapterName 'Wi-Fi' -AllowManagementOs $true -Notes 'Minikube: switch de comunicação com o Kubernetes.'
```
Exemplo de saída:
```
Name            SwitchType NetAdapterInterfaceDescription
----            ---------- ------------------------------
Minikube Switch External   Intel(R) Wi-Fi 6E AX210 160MHz
```
 
### Criando um cluster Kubernetes de nó único
 
O **Minikube** diferencia um cluster através do parâmetro ```--profile```, se este parâmetro for omitido, ele utiliza o default e sempre irá sobrepor a configuração anterior, caso você trabalhe com mais de um cluster, mesmo que de nó único, recomendo utilizar o parâmetro ```--profile <nome-do-cluster>```, isto preservará as configuração do seu cluster e permitirá que o kubectl interaja com com o seu cluster apenas trocando o contexto.
 
### Criando cluster de nó unico
Em um novo terminal, carregue a variável ```$kubeVersion``` novamente.
```powershell
$kubeVersion = (Invoke-WebRequest 'https://dl.k8s.io/release/stable.txt').Content
Write-Output $kubeVersion
```
 
### Criando o cluster
```powershell
minikube start --bootstrapper=kubeadm --driver=hyperv --hyperv-virtual-switch 'Minikube Switch' --kubernetes-version=$kubeVersion --memory=4096 --cpus=2 --profile=lab-k8s-01
```
Se você ver a saída abaixo, sucesso!!!
```
Done! kubectl is now configured to use "lab-k8s-01" cluster and "default" namespace by default
```
> Note que neste momento, há dois namespaces disponíveis, **lab-k8s-01** e **default**, lembrando que default não foi provisionado.
 
- Caso o comando retorne o erro ```No Major.Minor.Patch elements found```, é porque a variável ```$kubeVersion``` não foi carregada.
- O parâmetro ```--bootstrapper```, ```--memory``` e ```--cpus``` podem ser omitidos, eu prefiro usá-los para limitar os recursos do cluster.
 
 
### Criando mais de um cluster com mais de um nó cada (multi-cluster)
No exemplo abaixo, criarei o cluster lab-k8s-prod e o lab-k8s-dev, cada um com 3 nós.
 
### Cluster **lab-k8s-prod**
```powershell
minikube start --bootstrapper=kubeadm --driver=hyperv --hyperv-virtual-switch 'Minikube Switch' --kubernetes-version=$kubeVersion --nodes=3 --memory=4096 --cpus=2 --profile=lab-k8s-prod
```
 
### Cluster **lab-k8s-dev**
```powershell
minikube start --bootstrapper=kubeadm --driver=hyperv --hyperv-virtual-switch 'Minikube Switch' --kubernetes-version=$kubeVersion --nodes=3 --memory=4096 --cpus=2 --profile=lab-k8s-dev
```
 
---
## Verificando e testando o ambiente
 
### Verificando as máquinas virtuais criadas
```
PS C:\> Get-VM
 
Name             State   CPUUsage(%) MemoryAssigned(M) Uptime           Status               Version
----             -----   ----------- ----------------- ------           ------               -------
lab-k8s-01       Running 1           4096              00:13:26.2690000 Operando normalmente 10.0
lab-k8s-dev      Running 1           4096              00:02:39.3880000 Operando normalmente 10.0
lab-k8s-dev-m02  Running 1           4096              00:01:01.4760000 Operando normalmente 10.0
lab-k8s-dev-m03  Running 10          4096              00:00:10.3750000 Operando normalmente 10.0
lab-k8s-prod     Running 0           4096              00:06:39.9310000 Operando normalmente 10.0
lab-k8s-prod-m02 Running 0           4096              00:05:02.0120000 Operando normalmente 10.0
lab-k8s-prod-m03 Running 0           4096              00:04:12.0960000 Operando normalmente 10.0
 
PS C:\>
```
 
### Verificando os clusters provisionados pelo **Minikube**
```powershell
minikube profile list
```
Saída
```
PS C:\> minikube profile list
|--------------|-----------|---------|----------------|------|---------|---------|-------|
|   Profile    | VM Driver | Runtime |       IP       | Port | Version | Status  | Nodes |
|--------------|-----------|---------|----------------|------|---------|---------|-------|
| lab-k8s-01   | hyperv    | docker  | 192.168.31.248 | 8443 | v1.23.1 | Running |     1 |
| lab-k8s-dev  | hyperv    | docker  | 192.168.31.252 | 8443 | v1.23.1 | Running |     3 |
| lab-k8s-prod | hyperv    | docker  | 192.168.31.249 | 8443 | v1.23.1 | Running |     3 |
|--------------|-----------|---------|----------------|------|---------|---------|-------|
PS C:\>
```
 
### Verificando os clusters kubernetes disponíveis no seu ambiente
```powershell
kubectl config get-contexts
```
 
Repare que o último cluster é o cluster corrente no **kubectl**
```
PS C:\> kubectl config get-contexts
CURRENT   NAME           CLUSTER        AUTHINFO       NAMESPACE
          lab-k8s-01     lab-k8s-01     lab-k8s-01     default
*         lab-k8s-dev    lab-k8s-dev    lab-k8s-dev    default
          lab-k8s-prod   lab-k8s-prod   lab-k8s-prod   default
PS C:\>
 
```
 
### Trocando para o cluster lab-k8s-prod e listando todos os pods
```powershell
kubectl config use-context lab-k8s-prod
```
 
Verificando o contexto selecionado pelo **kubectl**:
```
PS C:\> kubectl config get-contexts
CURRENT   NAME           CLUSTER        AUTHINFO       NAMESPACE
          lab-k8s-01     lab-k8s-01     lab-k8s-01     default
          lab-k8s-dev    lab-k8s-dev    lab-k8s-dev    default
*         lab-k8s-prod   lab-k8s-prod   lab-k8s-prod   default
PS C:\>
```
 
Listando todos os **pods** em todos os **namespaces** em ***lab-k8s-prod**
```
PS C:\> kubectl get pods --all-namespaces
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   coredns-64897985d-9nwhj                1/1     Running   0          24m
kube-system   etcd-lab-k8s-prod                      1/1     Running   0          24m
kube-system   kindnet-4pprf                          1/1     Running   0          23m
kube-system   kindnet-bdpm2                          1/1     Running   0          24m
kube-system   kindnet-dgr7h                          1/1     Running   0          22m
kube-system   kube-apiserver-lab-k8s-prod            1/1     Running   0          24m
kube-system   kube-controller-manager-lab-k8s-prod   1/1     Running   0          24m
kube-system   kube-proxy-jkkc6                       1/1     Running   0          22m
kube-system   kube-proxy-lkxz8                       1/1     Running   0          24m
kube-system   kube-proxy-wz4p7                       1/1     Running   0          23m
kube-system   kube-scheduler-lab-k8s-prod            1/1     Running   0          24m
kube-system   storage-provisioner                    1/1     Running   0          24m
PS C:\>
```
 
### Desligando um cluster específico
```powershell
minikube stop --profile=lab-k8s-prod
```
 
### Desligando todos os clusters
```powershell
minikube stop --all
``` 

---
## Considerações
Creio que, para quem possui apenas uma máquina com o sistema operacional Windows, este guia vai ajudar nos estudos e testes com Kubernetes.
 
---
## Referências
 
Todos os comandos que utilizei neste guia, foram consultados nos sites das documentações oficiais das tecnologias empregadas aqui.
 
- [Documentação PowerShell](https://docs.microsoft.com/en-us/powershell/) 
 
- [Documentação Minikube](https://minikube.sigs.k8s.io/docs/)
 
- [Documentação Kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)

---
