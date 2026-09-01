Nuvem Pessoal com Nextcloud AIO, Samba e Tailscale

Projeto de nuvem pessoal baseado em Ubuntu Server, utilizando Nextcloud All-in-One (AIO) em Docker, integração com Samba e acesso remoto privado por Tailscale.

A proposta é oferecer uma experiência semelhante a um “Google Drive pessoal”, mantendo o compartilhamento SMB disponível na rede local e permitindo acesso remoto aos arquivos por navegador, aplicativo móvel e cliente Nextcloud.

Este README foi preparado para publicação pública. Todos os endereços, hostnames, usuários, domínios e identificadores reais foram substituídos por exemplos ou placeholders.

Objetivos

Manter o Samba disponível na rede local.

Acessar arquivos remotamente pelo Nextcloud.

Não expor SMB diretamente à Internet.

Utilizar Tailscale para acesso remoto privado.

Executar o Nextcloud isolado em Docker.

Preservar serviços já existentes no servidor, como Apache, Zabbix e Grafana.

Compartilhar uma pasta existente do Samba com o Nextcloud usando ACLs.

Arquitetura

                         INTERNET
                            │
                        Tailscale
                            │
              ┌─────────────┴─────────────┐
              │                           │
           Notebook                    Smartphone
           Windows                     Android/iOS
              │                           │
              └─────────────┬─────────────┘
                            │
                            ▼
                     Ubuntu Server
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       Apache            Grafana             Samba
      / Zabbix            :3000             :139/:445
          │
          │
          └──────── Serviços existentes

                            +
                            │
                            ▼
                       Docker Engine
                            │
                            ▼
                      Nextcloud AIO
                            │
                     127.0.0.1:11000
                            │
                            ▼
                      Tailscale Serve
                            │
                            ▼
               https://<host>.<tailnet>.ts.net

Componentes

Componente

Função

Ubuntu Server

Sistema operacional

Docker Engine

Execução dos containers

Nextcloud AIO

Nuvem pessoal

Samba

Compartilhamento SMB local

Tailscale

Rede privada para acesso remoto

Tailscale Serve

Proxy HTTPS privado para o Nextcloud

Apache

Mantido para serviços existentes

Zabbix

Monitoramento

Grafana

Dashboards

MariaDB

Banco de dados já existente no servidor

1. Verificações iniciais

Antes da instalação:

lsb_release -a
uname -a
ip addr
df -h
lsblk -f
free -h

Verifique portas já utilizadas:

sudo ss -lntup

Exemplo:

22     SSH
80     Apache/Zabbix
139    Samba
445    Samba
3000   Grafana
3306   MariaDB localhost
10050  Zabbix Agent
10051  Zabbix Server

Confirme que as portas utilizadas pelo AIO estão livres:

sudo ss -lntup | grep -E ':(8080|8443|11000)\b'

2. Preservando Apache e Zabbix

Se o Apache já fornece o frontend do Zabbix, não remova nem desabilite o serviço.

Verifique:

sudo apache2ctl -S

ls -lah /etc/apache2/conf-enabled/

E:

dpkg -l | grep -E 'zabbix.*frontend|zabbix.*apache|php'

O Nextcloud AIO será executado em uma porta interna separada, evitando conflito com a porta 80.

3. Instalação do Docker

Dependências

sudo apt update
sudo apt install ca-certificates curl -y

Chave GPG

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

Repositório

Exemplo para Ubuntu 24.04 (noble) em amd64:

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF

Atualize:

sudo apt update

Verifique:

apt-cache policy docker-ce

Instalação

sudo apt install \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin \
-y

Testes

sudo systemctl status docker --no-pager
sudo systemctl is-enabled docker
sudo docker --version
sudo docker compose version

Teste funcional:

sudo docker run hello-world

Resultado esperado:

Hello from Docker!

4. Pasta Samba

Exemplo de caminho:

/srv/samba/Compartilhado

Verifique:

ls -ld /srv/samba/Compartilhado

Evite:

chmod -R 777 /srv/samba/Compartilhado

e evite alterar recursivamente o proprietário da pasta para www-data.

A abordagem adotada é usar ACL.

5. ACL para Samba + Nextcloud

Verifique o usuário web:

getent passwd 33

Exemplo:

www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin

Conceda acesso:

sudo setfacl -R -m u:www-data:rwX /srv/samba/Compartilhado

Configure ACL padrão:

sudo find /srv/samba/Compartilhado -type d \
-exec setfacl -m \
d:u::rwx,d:u:www-data:rwx,d:g::rwx,d:m::rwx,d:o::--- {} +

Verifique:

getfacl /srv/samba/Compartilhado

Exemplo:

user::rwx
user:www-data:rwx
group::rwx
mask::rwx
other::---

default:user::rwx
default:user:www-data:rwx
default:group::rwx
default:mask::rwx
default:other::---

Teste de escrita:

sudo -u www-data touch /srv/samba/Compartilhado/.teste-nextcloud

Depois remova:

sudo rm /srv/samba/Compartilhado/.teste-nextcloud

6. Nextcloud AIO

O AIO será configurado para usar:

APACHE_PORT=11000
APACHE_IP_BINDING=127.0.0.1

e a pasta Samba será disponibilizada via:

NEXTCLOUD_MOUNT=/srv/samba/Compartilhado/

Criar o mastercontainer

sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=127.0.0.1 \
  --env APACHE_ADDITIONAL_NETWORK="" \
  --env SKIP_DOMAIN_VALIDATION=true \
  --env NEXTCLOUD_MOUNT="/srv/samba/Compartilhado/" \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  ghcr.io/nextcloud-releases/all-in-one:latest

Acesse o painel AIO pela LAN:

https://<IP_DO_SERVIDOR>:8080

Exemplo genérico:

https://192.168.1.100:8080

7. Tailscale no Ubuntu

Instale o Tailscale:

sudo mkdir -p --mode=0755 /usr/share/keyrings

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg \
| sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list \
| sudo tee /etc/apt/sources.list.d/tailscale.list

sudo apt update
sudo apt install tailscale -y

Autentique:

sudo tailscale up

Verifique:

tailscale status
tailscale ip -4

8. Tailscale Serve

O backend do Nextcloud fica restrito ao localhost:

127.0.0.1:11000

Publique-o somente dentro da tailnet:

sudo tailscale serve --bg http://127.0.0.1:11000

Verifique:

tailscale serve status

Exemplo:

https://<hostname>.<tailnet>.ts.net (tailnet only)
|-- / proxy http://127.0.0.1:11000

No AIO, configure apenas:

<hostname>.<tailnet>.ts.net

Sem:

https://

e sem / no final.

9. Validação

Backend:

curl -I http://127.0.0.1:11000

Um 302 Found é normal.

HTTPS via Tailscale:

curl -vkI https://<hostname>.<tailnet>.ts.net

Verifique a porta:

sudo ss -lntp | grep 11000

Esperado:

127.0.0.1:11000

Não:

0.0.0.0:11000

10. Variáveis do AIO

Confirme:

sudo docker inspect nextcloud-aio-mastercontainer \
--format '{{range .Config.Env}}{{println .}}{{end}}' \
| grep -E 'NEXTCLOUD_MOUNT|APACHE_PORT|APACHE_IP_BINDING|SKIP_DOMAIN'

Esperado:

APACHE_PORT=11000
APACHE_IP_BINDING=127.0.0.1
SKIP_DOMAIN_VALIDATION=true
NEXTCLOUD_MOUNT=/srv/samba/Compartilhado/

11. Containers opcionais

Para um servidor de recursos moderados que já executa outros serviços, uma configuração inicial simples é:

Recurso

Estado

Nextcloud Hub atual

Ativado

Office

Desativado

ClamAV

Desativado

Fulltext Search

Desativado

Imaginary

Ativado

Nextcloud Talk

Desativado

Talk Recording

Desativado

Docker Socket Proxy

Desativado

HaRP

Desativado

Whiteboard

Desativado

Fuso horário:

America/Sao_Paulo

12. Inicialização

Na interface AIO:

Baixe e inicie os contêineres

Acompanhe:

watch -n 3 'sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'

Exemplo de containers:

nextcloud-aio-mastercontainer
nextcloud-aio-apache
nextcloud-aio-nextcloud
nextcloud-aio-database
nextcloud-aio-redis
nextcloud-aio-notify-push
nextcloud-aio-imaginary

O estado esperado é:

healthy

13. External Storage Support

No Nextcloud:

Avatar
→ Aplicativos
→ External storage support
→ Ativar

Depois:

Avatar
→ Configurações de administração
→ Armazenamento externo

Cadastre:

Nome: Compartilhado
Tipo: Local
Autenticação: Nenhuma
Caminho: /srv/samba/Compartilhado

Opções sugeridas:

Somente leitura: desativado
Visualizações prévias: ativado
Compartilhamento: opcional
Mac NFD: desativado

14. Acesso pelo Windows

Instale o Tailscale e conecte o dispositivo à mesma tailnet.

Teste:

tailscale status

Resolve-DnsName <hostname>.<tailnet>.ts.net

Test-NetConnection <hostname>.<tailnet>.ts.net -Port 443

Depois acesse:

https://<hostname>.<tailnet>.ts.net

15. Acesso pelo celular

No Android ou iOS:

Instale o Tailscale.

Conecte à mesma tailnet.

Instale o aplicativo Nextcloud.

Informe:

https://<hostname>.<tailnet>.ts.net

Faça login.

Acesse:

Arquivos → Compartilhado

16. Fluxo final

Windows na LAN
     │
     │ SMB
     ▼
/srv/samba/Compartilhado
     ▲
     │
     │ External Storage
     ▼
Nextcloud AIO
     │
     ▼
127.0.0.1:11000
     │
     ▼
Tailscale Serve
     │
     ▼
https://<hostname>.<tailnet>.ts.net
     │
     ├── Windows remoto
     ├── Notebook
     ├── Android
     └── iPhone

17. Segurança

Não exponha SMB

Não publique:

137/UDP
138/UDP
139/TCP
445/TCP

Não exponha o backend do Nextcloud

Mantenha:

127.0.0.1:11000

Não publique informações reais no GitHub

Substitua por placeholders:

<IP_DO_SERVIDOR>
<HOSTNAME>
<TAILNET>
<USUARIO>
<DOMINIO_TAILSCALE>

Nunca publique:

senha;

passphrase do AIO;

auth key do Tailscale;

token;

chave SSH privada;

IP público real;

hostname real;

nome real da tailnet;

dumps de configuração contendo credenciais;

arquivos .env com segredos.

18. Backup

A pasta externa precisa de backup próprio.

Exemplo:

/srv/samba/Compartilhado
        │
        ├── backup local
        └── backup off-site

Considere a estratégia 3-2-1:

3 cópias
2 mídias diferentes
1 cópia externa

19. Troubleshooting

Docker não encontrado

docker: command not found

Instale o Docker Engine oficial.

Erro GPG

Exemplo:

NO_PUBKEY ...

Verifique:

gpg --show-keys /etc/apt/keyrings/docker.asc

Lock do APT/DPKG

Exemplo:

Could not get lock /var/lib/dpkg/lock-frontend

Identifique o processo:

ps -fp <PID>

Não remova manualmente os arquivos de lock enquanto existir um processo ativo.

Nextcloud não abre

Teste:

curl -I http://127.0.0.1:11000

tailscale serve status

curl -vkI https://<hostname>.<tailnet>.ts.net

Windows não resolve .ts.net

Confirme:

tailscale status

O dispositivo precisa estar conectado à tailnet.

Pasta Samba não aparece

Confira:

getfacl /srv/samba/Compartilhado

sudo docker inspect nextcloud-aio-mastercontainer \
--format '{{range .Config.Env}}{{println .}}{{end}}' \
| grep NEXTCLOUD_MOUNT

Teste dentro do container:

sudo docker exec --user www-data nextcloud-aio-nextcloud \
ls -lah /srv/samba/Compartilhado

20. Comandos úteis

Docker

sudo docker ps
sudo docker ps -a
sudo docker images

Logs

sudo docker logs --tail 100 nextcloud-aio-nextcloud

sudo docker logs --tail 100 nextcloud-aio-mastercontainer

Tailscale

tailscale status
tailscale serve status

Samba

sudo testparm -s
getfacl /srv/samba/Compartilhado

Resultado

Com essa arquitetura:

o Samba continua disponível na LAN;

o Nextcloud roda isolado em Docker;

os arquivos do Samba podem ser acessados pelo Nextcloud;

o acesso remoto ocorre por Tailscale;

SMB não fica exposto à Internet;

Windows, notebooks e celulares podem acessar os arquivos remotamente;

serviços existentes como Zabbix e Grafana podem permanecer no mesmo servidor.

Referências oficiais

Docker Engine: https://docs.docker.com/engine/install/ubuntu/

Nextcloud AIO: https://github.com/nextcloud/all-in-one

Nextcloud External Storage: https://docs.nextcloud.com/server/latest/admin_manual/configuration_files/external_storage_configuration_gui.html

Tailscale: https://tailscale.com/

Tailscale Serve: https://tailscale.com/kb/1242/tailscale-serve

Aviso

Este projeto foi desenvolvido para laboratório e uso pessoal.

Antes de utilizar em produção, revise:

firewall;

backup;

atualização dos containers;

autenticação multifator;

política de senhas;

logs;

hardening;

segmentação de rede;

disponibilidade e recuperação de desastre.
