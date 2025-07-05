# Homelab do Zero

## 1. Preparação do Ambiente

1. **Hardware Necessário:**
   - 4 Raspberry Pi 4B com 8GB de RAM.
   - 4 cartões microSD de 250GB, classe 10.
   - Fonte de alimentação adequada para cada Raspberry Pi.
   - Switch de rede e cabos Ethernet.

2. **Configuração Inicial:**
   - Instale o sistema operacional Raspberry Pi OS Lite em cada cartão microSD.
   - Configure o acesso SSH em cada Raspberry Pi.
   - Conecte cada Raspberry Pi à rede local via Ethernet.

### 2. Configuração do Cluster K3s

1. **Instalação do K3s:**
   - No Raspberry Pi que será o master, execute:
     ```bash
     curl -sfL https://get.k3s.io | sh -
     ```
   - Anote o token gerado para adicionar nós ao cluster.

2. **Configuração dos Nós:**
   - Nos outros 3 Raspberry Pi, execute:
     ```bash
     curl -sfL https://get.k3s.io | K3S_URL=https://192.168.15.2:6443 K3S_TOKEN=<TOKEN> sh -
     ```
   - Substitua `<TOKEN>` pelo token anotado anteriormente.

3. **Verificação do Cluster:**
   - No master, verifique se todos os nós estão ativos:
     ```bash
     kubectl get nodes
     ```

### 3. Implantação dos Serviços

1. **Namespace para Rancher:**
   - Crie um namespace para o Rancher:
     ```bash
     kubectl create namespace rancher
     ```
2. **Instalação do Rancher no K3s:**
   - Adicione o repositório Helm do Rancher:
     ```bash
     helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
     helm repo update
     ```
   - Instale o Rancher no K3s:
     ```bash
     kubectl create namespace rancher
     kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.crds.yaml
     helm repo add jetstack https://charts.jetstack.io
     helm repo update
     helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.5.3 --kubeconfig /etc/rancher/k3s/k3s.yaml
     
     helm install rancher rancher-latest/rancher --namespace rancher --set hostname=rancher.local --set replicas=3 --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=webmaster@quati.com.br --set letsEncrypt.ingress.class=traefik --set letsEncrypt.environment=staging --kubeconfig /etc/rancher/k3s/k3s.yaml --wait
     ```
     - Aguarde até que os pods do Rancher estejam em execução:
     ```bash
     kubectl -n rancher rollout status deploy/rancher --kubeconfig /etc/rancher/k3s/k3s.yaml
     ```
     - Adicione uma entrada no seu arquivo `/etc/hosts` para acessar o Rancher via hostname:
     ```bash
     sudo nano /etc/hosts
     ```
     - Adicione a linha:
     ```bash
     192.168.15.2 rancher.local
     ```
     - Salve e feche o arquivo.
     - Agora você pode acessar o Rancher através do hostname `https://rancher.local`.
     
     Gere a senha de acesso administrador do Rancher:
     ```bash
     kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
     ```

3. **Namespace para Pi-hole:**
   - Crie um namespace para o Pi-hole:
     ```bash
     kubectl create namespace pihole
     ```
4. **Instalação do Pi-hole:**
   - Utilize um chart Helm para instalar o Pi-hole no namespace `pihole` com alta disponibilidade:
     ```bash
     helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
     helm repo update
     helm install pihole mojo2600/pihole --namespace pihole --set service.type=LoadBalancer --set service.externalTrafficPolicy=Local --set persistence.enabled=false --set replicaCount=4 --set service.loadBalancerIP=192.168.15.2 --set service.port=80 --set service.targetPort=80 --kubeconfig /etc/rancher/k3s/k3s.yaml
     ```

   - Alternativamente, utilize o seguinte manifesto YAML para instalar o Pi-hole no namespace `pihole` com alta disponibilidade:
     ```yaml
     apiVersion: v1
     kind: Namespace
     metadata:
       name: pihole

     ---
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: pihole
       namespace: pihole
     spec:
       replicas: 4
       selector:
         matchLabels:
           app: pihole
       template:
         metadata:
           labels:
             app: pihole
         spec:
           containers:
           - name: pihole
             image: pihole/pihole:latest
             ports:
             - containerPort: 80
             - containerPort: 53
             env:
             - name: TZ
               value: "America/Sao_Paulo"
             - name: WEBPASSWORD
               value: "yourpassword"
             volumeMounts:
             - name: pihole-config
               mountPath: /etc/pihole
             - name: dnsmasq-config
               mountPath: /etc/dnsmasq.d
           volumes:
           - name: pihole-config
             emptyDir: {}
           - name: dnsmasq-config
             emptyDir: {}

     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: pihole
       namespace: pihole
     spec:
       type: LoadBalancer
       ports:
       - port: 80
         targetPort: 80
       - port: 53
         targetPort: 53
         protocol: UDP
       selector:
         app: pihole
     ```

   - Adicione uma entrada no seu arquivo `/etc/hosts` para acessar o Pi-hole via hostname:
     ```bash
     sudo nano /etc/hosts
     ```

   - Adicione a linha:
     ```bash
     192.168.15.2 pihole.local
     ```

   - Salve e feche o arquivo.
   - Agora você pode acessar o Pi-hole através do hostname `http://pihole.local/admin`.
   - Verifique o nome do pod do Pi-hole com detalhes adicionais:
     ```bash
     k3s kubectl get pods -n pihole -o wide
     ```

   - Defina a senha inicial para o Pi-hole no K3s:
     ```bash
     k3s kubectl exec -it <nome-do-pod> -n pihole -- pihole -a -p <nova-senha>
     ```
   - Substitua `<nome-do-pod>` pelo nome do pod do Pi-hole e `<nova-senha>` pela senha desejada.
   - Se a senha não funcionar, redefina a senha diretamente no pod:
     ```bash
     k3s kubectl exec -it <nome-do-pod> -n pihole -- sudo pihole -a -p <nova-senha>
     ```
     
   - Verifique se a senha foi alterada com sucesso acessando a interface web do Pi-hole em `http://pihole.local/admin` e fazendo login com a nova senha.


### 4. Configuração de Rede

1. **Configuração de Saída para a Rede Local:**
   - Configure o IP do master para 192.168.15.2/24.
   - Certifique-se de que o tráfego de saída dos serviços Rancher e Pi-hole está roteado corretamente através do master.

### 5. Testes e Verificações

1. **Acesso ao Rancher:**
   - Acesse o Rancher via navegador em `https://rancher.local`.

2. **Verificação do Pi-hole:**
   - Acesse a interface do Pi-hole para verificar o bloqueio de anúncios.

3. **Monitoramento do Cluster:**
   - Utilize ferramentas como `kubectl` e `k9s` para monitorar o estado do cluster e dos pods.

### 6. Manutenção e Atualizações

1. **Atualizações de Sistema:**
   - Mantenha o sistema operacional e os pacotes atualizados em cada Raspberry Pi.

2. **Backup e Recuperação:**
   - Configure backups regulares dos dados críticos e das configurações do cluster.

Seguindo este roteiro, você terá um home lab funcional com K3s, Rancher e Pi-hole, cada um em seu namespace separado, e com saída para a rede local através do master.