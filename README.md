# Wizard - Portainer e NGinx


Este guia explica como configurar automaticamente o **Portainer.IO** e o **NGINX Proxy Manager** em uma VPS com **Ubuntu**. Ele inclui dois métodos diferentes para garantir que a instalação funcione corretamente, além de informações sobre como usar o script `setup.sh`.

---

## **Pré-requisitos**

- VPS com **Ubuntu** (recomenda-se Ubuntu 20.04 ou superior).
- Acesso com permissões de super usuário (root).
- Conexão com a internet.

---

## **Método 1 (Execução Automática)**

Este método executa tudo em uma única linha de comando.

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y dos2unix && cd / && mkdir -p stacks && cd /stacks && curl -fsSL https://raw.githubusercontent.com/smile-dev-br/vps-wizard/refs/heads/main/setup.sh -o setup.sh && cat setup.sh && dos2unix setup.sh && chmod +x setup.sh && ./setup.sh
```

### **O que este comando faz**

1. **Atualiza o sistema**:
    - Atualiza os pacotes e instala o `dos2unix` (necessário para corrigir arquivos com formatação de linha do Windows).
2. **Cria a estrutura de diretórios**:
    - Cria a pasta `/stacks` onde o script será armazenado e executado.
3. **Baixa o script `setup.sh`**:
    - Faz o download do script hospedado no Pastebin e o salva como `setup.sh`.
4. **Valida e executa o script**:
    - Exibe o conteúdo do script com `cat setup.sh` para validação.
    - Converte o arquivo para formato Unix com `dos2unix`.
    - Torna o script executável e o executa.

---

## **Método 2 (Execução Manual)**

Se o método automático não funcionar, siga os passos abaixo para configurar manualmente:

### **Passo 1: Atualize o sistema**

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y dos2unix
```

### **Passo 2: Crie a pasta `stacks`**

```bash
cd /
mkdir -p stacks
cd /stacks
```

### **Passo 3: Baixe ou crie o arquivo `setup.sh`**

1. **Use SFTP ou o editor `nano`** para criar o arquivo `setup.sh`.
2. **Cole o conteúdo abaixo no arquivo `setup.sh`:**

```bash
#!/bin/bash

# Cores para output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

# Função de log
log() { echo -e "${GREEN}[+] $1${NC}"; }

# Função de aviso
warn() { echo -e "${YELLOW}[!] $1${NC}"; }

# Função de erro
error() {
  echo -e "${RED}[!] ERRO: $1${NC}"
  exit 1
}

# Verificar se é root
if [[ $EUID -ne 0 ]]; then
  error "Este script precisa ser rodado como root. Use: sudo ./setup.sh"
fi

# Instalar Docker se não estiver instalado
if ! command -v docker &> /dev/null; then
  warn "Docker não está instalado. Instalando Docker..."
  curl -fsSL https://get.docker.com | sudo bash || error "Falha ao instalar Docker."
  log "Docker instalado com sucesso."
fi

# Verificar se o Docker Compose está instalado
if ! command -v docker-compose &> /dev/null; then
  warn "Docker Compose não está instalado. Instalando..."
  curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose || error "Falha ao baixar Docker Compose."
  chmod +x /usr/local/bin/docker-compose || error "Falha ao tornar o Docker Compose executável."
  log "Docker Compose instalado com sucesso."
fi

# Inicializar Docker Swarm se não estiver iniciado
if ! docker info | grep -q "Swarm: active"; then
  warn "Docker Swarm não está iniciado. Inicializando..."
  docker swarm init || error "Falha ao inicializar Docker Swarm."
  log "Docker Swarm inicializado com sucesso."
else
  log "Docker Swarm já está ativo."
fi

# Variáveis
SETUP_DIR=$(realpath $(dirname "$0"))
PORTAINER_DIR="$SETUP_DIR/portainer"
NGINX_DIR="$SETUP_DIR/nginx-proxy-manager"

log "Configurando diretórios de dados em $PORTAINER_DIR e $NGINX_DIR..."

# Portas
PORTAINER_HTTP=9000
PORTAINER_HTTPS=9443
NGINX_HTTP=80
NGINX_HTTPS=443
NGINX_ADMIN=81

# Criar diretórios para volumes
log "Criando diretórios locais para volumes..."
mkdir -p "$PORTAINER_DIR/portainer_data" || error "Falha ao criar diretório $PORTAINER_DIR/portainer_data"
mkdir -p "$NGINX_DIR/nginx_data" || error "Falha ao criar diretório $NGINX_DIR/nginx_data"
mkdir -p "$NGINX_DIR/letsencrypt" || error "Falha ao criar diretório $NGINX_DIR/letsencrypt"

# Criar rede pública
log "Criando rede pública 'rede_publica'..."
docker network rm rede_publica &> /dev/null
docker network create --driver overlay --attachable rede_publica || error "Falha ao criar rede pública 'rede_publica'"

# Docker Compose Portainer
log "Gerando arquivo docker-compose.yml para Portainer..."
cat > "$PORTAINER_DIR/docker-compose.yml" << EOL
services:
  portainer:
    image: portainer/portainer-ce:2.21.4
    container_name: portainer
    restart: always
    networks:
      - rede_publica
    ports:
      - 8000:8000
      - $PORTAINER_HTTP:9000
      - $PORTAINER_HTTPS:9443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: $PORTAINER_DIR/portainer_data

networks:
  rede_publica:
    external: true
EOL

# Docker Compose Nginx Proxy Manager
log "Gerando arquivo docker-compose.yml para Nginx Proxy Manager..."
cat > "$NGINX_DIR/docker-compose.yml" << EOL
services:
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: always
    networks:
      - rede_publica
    ports:
      - $NGINX_HTTP:80
      - $NGINX_ADMIN:81
      - $NGINX_HTTPS:443
    volumes:
      - nginx_data:/data
      - letsencrypt:/etc/letsencrypt

volumes:
  nginx_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: $NGINX_DIR/nginx_data
  letsencrypt:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: $NGINX_DIR/letsencrypt

networks:
  rede_publica:
    external: true
EOL

# Subir serviços
log "Iniciando Portainer..."
docker-compose -f "$PORTAINER_DIR/docker-compose.yml" up -d || error "Falha ao iniciar Portainer"

log "Iniciando Nginx Proxy Manager..."
docker-compose -f "$NGINX_DIR/docker-compose.yml" up -d || error "Falha ao iniciar Nginx Proxy Manager"

# Verificar status dos containers
log "Verificando containers em execução..."
docker ps | grep -E "portainer|nginx-proxy-manager" || error "Nenhum container está em execução."

# IP do servidor
SERVER_IP=$(hostname -I | awk '{print $1}')

# Mensagem final
log "Redes e serviços configurados com sucesso! 🚀"
log "Portainer acessível em:"
log "  - HTTP:  http://$SERVER_IP:$PORTAINER_HTTP"
log "  - HTTPS: https://$SERVER_IP:$PORTAINER_HTTPS"
log "Nginx Proxy Manager acessível em:"
log "  - HTTP:  http://$SERVER_IP:$NGINX_ADMIN"
log "  "
log "Credenciais padrão do Nginx Proxy Manager:"
log "  Email:    admin@example.com"
log "  Password: changeme"
```

---

### **Passo 4: Converta o arquivo para formato Unix e execute**

```bash
dos2unix setup.sh
chmod +x setup.sh
./setup.sh
```

---

## **Resultados**

- **Portainer**:
    
    - HTTP: `http://<IP_DO_SERVIDOR>:9000`
    - HTTPS: `https://<IP_DO_SERVIDOR>:9443`
- **NGINX Proxy Manager**:
    
    - HTTP: `http://<IP_DO_SERVIDOR>:81`
    - Credenciais padrão:
        - **Email**: `admin@example.com`
        - **Password**: `changeme`

Com tudo configurado, você pode acessar e gerenciar seus serviços de maneira fácil e intuitiva! 🚀

---

### **Conclusão**

Esse guia cobre os passos essenciais para configurar Portainer.IO e NGINX Proxy Manager de forma rápida e eficiente. Escolha o método mais conveniente para o seu caso. 🚀
