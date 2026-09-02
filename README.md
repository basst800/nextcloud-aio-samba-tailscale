# Nuvem Pessoal com Nextcloud AIO, Samba e Tailscale

Projeto de **nuvem pessoal** baseado em Ubuntu Server, utilizando **Nextcloud All-in-One (AIO)** em Docker, integração com **Samba** e acesso remoto privado por **Tailscale**.

A proposta é oferecer uma experiência semelhante a um “Google Drive pessoal”, mantendo o compartilhamento SMB disponível na rede local e permitindo acesso remoto aos arquivos por navegador, aplicativo móvel e cliente Nextcloud.

> Este README foi preparado para publicação pública. Todos os endereços, hostnames, usuários, domínios e identificadores reais foram substituídos por exemplos ou placeholders.

---

## Objetivos

- Manter o Samba disponível na rede local.
- Acessar arquivos remotamente pelo Nextcloud.
- Não expor SMB diretamente à Internet.
- Utilizar Tailscale para acesso remoto privado.
- Executar o Nextcloud isolado em Docker.
- Preservar serviços já existentes no servidor, como Apache, Zabbix e Grafana.
- Compartilhar uma pasta existente do Samba com o Nextcloud usando ACLs.

---

## Arquitetura

```text
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
```

---

## Componentes

| Componente | Função |
|---|---|
| Ubuntu Server | Sistema operacional |
| Docker Engine | Execução dos containers |
| Nextcloud AIO | Nuvem pessoal |
| Samba | Compartilhamento SMB local |
| Tailscale | Rede privada para acesso remoto |
| Tailscale Serve | Proxy HTTPS privado para o Nextcloud |
| Apache | Mantido para serviços existentes |
| Zabbix | Monitoramento |
| Grafana | Dashboards |
| MariaDB | Banco de dados já existente no servidor |

---

# 1. Verificações iniciais

Antes da instalação:

```bash
lsb_release -a
uname -a
ip addr
df -h
lsblk -f
free -h
```

Verifique portas já utilizadas:

```bash
sudo ss -lntup
```

Exemplo:

```text
22     SSH
80     Apache/Zabbix
139    Samba
445    Samba
3000   Grafana
3306   MariaDB localhost
10050  Zabbix Agent
10051  Zabbix Server
```

Confirme que as portas utilizadas pelo AIO estão livres:

```bash
sudo ss -lntup | grep -E ':(8080|8443|11000)\b'
```

---

# 2. Preservando Apache e Zabbix

Se o Apache já fornece o frontend do Zabbix, não remova nem desabilite o serviço.

Verifique:

```bash
sudo apache2ctl -S
```

```bash
ls -lah /etc/apache2/conf-enabled/
```

E:

```bash
dpkg -l | grep -E 'zabbix.*frontend|zabbix.*apache|php'
```

O Nextcloud AIO será executado em uma porta interna separada, evitando conflito com a porta 80.

---

# 3. Instalação do Docker

## Dependências

```bash
sudo apt update
sudo apt install ca-certificates curl -y
```

## Chave GPG

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## Repositório

Exemplo para Ubuntu 24.04 (`noble`) em `amd64`:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Atualize:

```bash
sudo apt update
```

Verifique:

```bash
apt-cache policy docker-ce
```

## Instalação

```bash
sudo apt install \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin \
-y
```

## Testes

```bash
sudo systemctl status docker --no-pager
sudo systemctl is-enabled docker
sudo docker --version
sudo docker compose version
```

Teste funcional:

```bash
sudo docker run hello-world
```

Resultado esperado:

```text
Hello from Docker!
```

---

# 4. Pasta Samba

Exemplo de caminho:

```text
/srv/samba/Compartilhado
```

Verifique:

```bash
ls -ld /srv/samba/Compartilhado
```

Evite:

```bash
chmod -R 777 /srv/samba/Compartilhado
```

e evite alterar recursivamente o proprietário da pasta para `www-data`.

A abordagem adotada é usar ACL.

---

# 5. ACL para Samba + Nextcloud

Verifique o usuário web:

```bash
getent passwd 33
```

Exemplo:

```text
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

Conceda acesso:

```bash
sudo setfacl -R -m u:www-data:rwX /srv/samba/Compartilhado
```

Configure ACL padrão:

```bash
sudo find /srv/samba/Compartilhado -type d \
-exec setfacl -m \
d:u::rwx,d:u:www-data:rwx,d:g::rwx,d:m::rwx,d:o::--- {} +
```

Verifique:

```bash
getfacl /srv/samba/Compartilhado
```

Exemplo:

```text
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
```

Teste de escrita:

```bash
sudo -u www-data touch /srv/samba/Compartilhado/.teste-nextcloud
```

Depois remova:

```bash
sudo rm /srv/samba/Compartilhado/.teste-nextcloud
```

---

# 6. Nextcloud AIO

O AIO será configurado para usar:

```text
APACHE_PORT=11000
APACHE_IP_BINDING=127.0.0.1
```

e a pasta Samba será disponibilizada via:

```text
NEXTCLOUD_MOUNT=/srv/samba/Compartilhado/
```

## Criar o mastercontainer

```bash
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
```

Acesse o painel AIO pela LAN:

```text
https://<IP_DO_SERVIDOR>:8080
```

Exemplo genérico:

```text
https://192.168.1.100:8080
```

---

# 7. Tailscale no Ubuntu

Instale o Tailscale:

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
```

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg \
| sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
```

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list \
| sudo tee /etc/apt/sources.list.d/tailscale.list
```

```bash
sudo apt update
sudo apt install tailscale -y
```

Autentique:

```bash
sudo tailscale up
```

Verifique:

```bash
tailscale status
tailscale ip -4
```

---

# 8. Tailscale Serve

O backend do Nextcloud fica restrito ao localhost:

```text
127.0.0.1:11000
```

Publique-o somente dentro da tailnet:

```bash
sudo tailscale serve --bg http://127.0.0.1:11000
```

Verifique:

```bash
tailscale serve status
```

Exemplo:

```text
https://<hostname>.<tailnet>.ts.net (tailnet only)
|-- / proxy http://127.0.0.1:11000
```

No AIO, configure apenas:

```text
<hostname>.<tailnet>.ts.net
```

Sem:

```text
https://
```

e sem `/` no final.

---

# 9. Validação

Backend:

```bash
curl -I http://127.0.0.1:11000
```

Um `302 Found` é normal.

HTTPS via Tailscale:

```bash
curl -vkI https://<hostname>.<tailnet>.ts.net
```

Verifique a porta:

```bash
sudo ss -lntp | grep 11000
```

Esperado:

```text
127.0.0.1:11000
```

Não:

```text
0.0.0.0:11000
```

---

# 10. Variáveis do AIO

Confirme:

```bash
sudo docker inspect nextcloud-aio-mastercontainer \
--format '{{range .Config.Env}}{{println .}}{{end}}' \
| grep -E 'NEXTCLOUD_MOUNT|APACHE_PORT|APACHE_IP_BINDING|SKIP_DOMAIN'
```

Esperado:

```text
APACHE_PORT=11000
APACHE_IP_BINDING=127.0.0.1
SKIP_DOMAIN_VALIDATION=true
NEXTCLOUD_MOUNT=/srv/samba/Compartilhado/
```

---

# 11. Containers opcionais

Para um servidor de recursos moderados que já executa outros serviços, uma configuração inicial simples é:

| Recurso | Estado |
|---|---|
| Nextcloud Hub atual | Ativado |
| Office | Desativado |
| ClamAV | Desativado |
| Fulltext Search | Desativado |
| Imaginary | Ativado |
| Nextcloud Talk | Desativado |
| Talk Recording | Desativado |
| Docker Socket Proxy | Desativado |
| HaRP | Desativado |
| Whiteboard | Desativado |

Fuso horário:

```text
America/Sao_Paulo
```

---

# 12. Inicialização

Na interface AIO:

```text
Baixe e inicie os contêineres
```

Acompanhe:

```bash
watch -n 3 'sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
```

Exemplo de containers:

```text
nextcloud-aio-mastercontainer
nextcloud-aio-apache
nextcloud-aio-nextcloud
nextcloud-aio-database
nextcloud-aio-redis
nextcloud-aio-notify-push
nextcloud-aio-imaginary
```

O estado esperado é:

```text
healthy
```

---

# 13. External Storage Support

No Nextcloud:

```text
Avatar
→ Aplicativos
→ External storage support
→ Ativar
```

Depois:

```text
Avatar
→ Configurações de administração
→ Armazenamento externo
```

Cadastre:

```text
Nome: Compartilhado
Tipo: Local
Autenticação: Nenhuma
Caminho: /srv/samba/Compartilhado
```

Opções sugeridas:

```text
Somente leitura: desativado
Visualizações prévias: ativado
Compartilhamento: opcional
Mac NFD: desativado
```

---

# 14. Acesso pelo Windows

Instale o Tailscale e conecte o dispositivo à mesma tailnet.

Teste:

```powershell
tailscale status
```

```powershell
Resolve-DnsName <hostname>.<tailnet>.ts.net
```

```powershell
Test-NetConnection <hostname>.<tailnet>.ts.net -Port 443
```

Depois acesse:

```text
https://<hostname>.<tailnet>.ts.net
```

---

# 15. Acesso pelo celular

No Android ou iOS:

1. Instale o Tailscale.
2. Conecte à mesma tailnet.
3. Instale o aplicativo Nextcloud.
4. Informe:
   ```text
   https://<hostname>.<tailnet>.ts.net
   ```
5. Faça login.
6. Acesse:
   ```text
   Arquivos → Compartilhado
   ```

---

# 16. Fluxo final

```text
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
```

---

# 17. Segurança

## Não exponha SMB

Não publique:

```text
137/UDP
138/UDP
139/TCP
445/TCP
```

## Não exponha o backend do Nextcloud

Mantenha:

```text
127.0.0.1:11000
```

## Não publique informações reais no GitHub

Substitua por placeholders:

```text
<IP_DO_SERVIDOR>
<HOSTNAME>
<TAILNET>
<USUARIO>
<DOMINIO_TAILSCALE>
```

Nunca publique:

- senha;
- passphrase do AIO;
- auth key do Tailscale;
- token;
- chave SSH privada;
- IP público real;
- hostname real;
- nome real da tailnet;
- dumps de configuração contendo credenciais;
- arquivos `.env` com segredos.

---

# 18. Backup

A pasta externa precisa de backup próprio.

Exemplo:

```text
/srv/samba/Compartilhado
        │
        ├── backup local
        └── backup off-site
```

Considere a estratégia 3-2-1:

```text
3 cópias
2 mídias diferentes
1 cópia externa
```

---

# 19. Troubleshooting

## Docker não encontrado

```text
docker: command not found
```

Instale o Docker Engine oficial.

---

## Erro GPG

Exemplo:

```text
NO_PUBKEY ...
```

Verifique:

```bash
gpg --show-keys /etc/apt/keyrings/docker.asc
```

---

## Lock do APT/DPKG

Exemplo:

```text
Could not get lock /var/lib/dpkg/lock-frontend
```

Identifique o processo:

```bash
ps -fp <PID>
```

Não remova manualmente os arquivos de lock enquanto existir um processo ativo.

---

## Nextcloud não abre

Teste:

```bash
curl -I http://127.0.0.1:11000
```

```bash
tailscale serve status
```

```bash
curl -vkI https://<hostname>.<tailnet>.ts.net
```

---

## Windows não resolve `.ts.net`

Confirme:

```powershell
tailscale status
```

O dispositivo precisa estar conectado à tailnet.

---

## Pasta Samba não aparece

Confira:

```bash
getfacl /srv/samba/Compartilhado
```

```bash
sudo docker inspect nextcloud-aio-mastercontainer \
--format '{{range .Config.Env}}{{println .}}{{end}}' \
| grep NEXTCLOUD_MOUNT
```

Teste dentro do container:

```bash
sudo docker exec --user www-data nextcloud-aio-nextcloud \
ls -lah /srv/samba/Compartilhado
```

---

# 20. Comandos úteis

## Docker

```bash
sudo docker ps
sudo docker ps -a
sudo docker images
```

## Logs

```bash
sudo docker logs --tail 100 nextcloud-aio-nextcloud
```

```bash
sudo docker logs --tail 100 nextcloud-aio-mastercontainer
```

## Tailscale

```bash
tailscale status
tailscale serve status
```

## Samba

```bash
sudo testparm -s
getfacl /srv/samba/Compartilhado
```

---

# Resultado

Com essa arquitetura:

- o Samba continua disponível na LAN;
- o Nextcloud roda isolado em Docker;
- os arquivos do Samba podem ser acessados pelo Nextcloud;
- o acesso remoto ocorre por Tailscale;
- SMB não fica exposto à Internet;
- Windows, notebooks e celulares podem acessar os arquivos remotamente;
- serviços existentes como Zabbix e Grafana podem permanecer no mesmo servidor.

---

## Referências oficiais

- Docker Engine: https://docs.docker.com/engine/install/ubuntu/
- Nextcloud AIO: https://github.com/nextcloud/all-in-one
- Nextcloud External Storage: https://docs.nextcloud.com/server/latest/admin_manual/configuration_files/external_storage_configuration_gui.html
- Tailscale: https://tailscale.com/
- Tailscale Serve: https://tailscale.com/kb/1242/tailscale-serve

---

## Aviso

Este projeto foi desenvolvido para laboratório e uso pessoal.

Antes de utilizar em produção, revise:

- firewall;
- backup;
- atualização dos containers;
- autenticação multifator;
- política de senhas;
- logs;
- hardening;
- segmentação de rede;
- disponibilidade e recuperação de desastre.
