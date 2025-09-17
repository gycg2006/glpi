## Implementação do GLPI com Docker em ambiente Ubuntu WSL

### Objetivo
Implementar uma instância local do GLPI (Gestão Livre de Parque de Informática) utilizando Docker e Docker Compose em uma máquina virtual Ubuntu Server através do WSL do Windows.

### Requisitos técnicos avaliados
- Conhecimentos em ambiente Linux (Ubuntu)
- Configuração do WSL (Windows Subsystem for Linux)
- Comandos e operações básicas em terminal Linux
- Instalação e configuração de pacotes
- Redes e protocolos
- Tecnologia de contêineres (Docker e Docker Compose)
- Solução de problemas

---

## Parte 1: Configuração do ambiente Ubuntu no WSL

### 1.1. Instalação do WSL e Ubuntu Server
1. Abra o PowerShell como administrador e execute:
   ```powershell
   wsl --install -d Ubuntu
   ```
   
2. Se você já tem o WSL instalado, mas precisa adicionar o Ubuntu:
   ```powershell
   wsl --install -d Ubuntu
   ```

3. Após a instalação, o Ubuntu iniciará automaticamente. Configure seu nome de usuário e senha quando solicitado.

**Dica**: Em caso de problemas com a instalação do WSL, consulte a [documentação oficial da Microsoft](https://learn.microsoft.com/pt-br/windows/wsl/install).

### 1.2. Atualização do sistema Ubuntu
```bash
sudo apt update
sudo apt upgrade -y
```

---

## Parte 2: Instalação do Docker e Docker Compose

### 2.1. Instalação do Docker
```bash
# Remover versões antigas se existirem
sudo apt remove docker docker-engine docker.io containerd runc

# Instalar pré-requisitos
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg

# Adicionar a chave GPG oficial do Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Adicionar o repositório do Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Atualizar o índice de pacotes
sudo apt update

# Instalar o Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

### 2.2. Configurar Docker sem sudo
```bash
# Adicionar seu usuário ao grupo docker
sudo usermod -aG docker $USER

# Aplicar as alterações de grupo (ou reinicie a sessão)
newgrp docker
```

### 2.3. Verificar a instalação do Docker
```bash
# Verificar se o Docker foi instalado corretamente
docker --version

# Executar uma imagem de teste
docker run hello-world
```

### 2.4. Instalação do Docker Compose
```bash
# Instalar Docker Compose
sudo apt install -y docker-compose-plugin

# Verificar a instalação
docker compose version
```

**Dica**: Se você encontrar problemas com a instalação do Docker, consulte a [documentação oficial do Docker](https://docs.docker.com/engine/install/ubuntu/).

---

## Parte 3: Criação do projeto GLPI com Docker Compose

### 3.1. Criar estrutura de diretórios
```bash
# Criar diretório para o projeto
mkdir -p ~/glpi-docker
cd ~/glpi-docker

# Criar diretório para volumes persistentes
mkdir -p volumes/glpi volumes/mysql volumes/phpmyadmin
```

### 3.2. Criar arquivo docker-compose.yml
Crie um arquivo chamado `docker-compose.yml` usando um editor como o nano:

```bash
nano docker-compose.yml
```

Adicione o seguinte conteúdo:

```yaml
version: '3'

services:
  mysql:
    image: mysql:5.7
    container_name: glpi-mysql
    volumes:
      - ./volumes/mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=glpi
      - MYSQL_DATABASE=glpi
      - MYSQL_USER=glpi
      - MYSQL_PASSWORD=glpi
    restart: always
    networks:
      - glpi-network

  glpi:
    image: diouxx/glpi
    container_name: glpi
    volumes:
      - ./volumes/glpi:/var/www/html/glpi
    environment:
      - TIMEZONE=America/Sao_Paulo
    ports:
      - "8080:80"
    depends_on:
      - mysql
    restart: always
    networks:
      - glpi-network

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: glpi-phpmyadmin
    environment:
      - PMA_HOST=mysql
      - PMA_PORT=3306
      - MYSQL_ROOT_PASSWORD=glpi
    ports:
      - "8081:80"
    depends_on:
      - mysql
    restart: always
    networks:
      - glpi-network

networks:
  glpi-network:
    driver: bridge
```

**Dica de redes**: Observe a configuração de redes no arquivo docker-compose. O uso de `networks` cria uma rede isolada para os contêineres se comunicarem. A opção `driver: bridge` permite que os contêineres na mesma rede se comuniquem entre si enquanto permanecem isolados de outros contêineres.

### 3.3. Iniciar os contêineres
```bash
# Iniciar os serviços em segundo plano
docker compose up -d
```

### 3.4. Verificar o status dos contêineres
```bash
# Verificar se todos os contêineres estão em execução
docker ps

# Verificar logs em caso de problemas
docker logs glpi
docker logs glpi-mysql
docker logs glpi-phpmyadmin
```

---

## Parte 4: Configuração do GLPI

### 4.1. Acesso ao GLPI
1. Abra um navegador e acesse: `http://localhost:8080/glpi`
2. Você será redirecionado para a página de instalação do GLPI.

### 4.2. Instalação do GLPI
1. Selecione o idioma e clique em "OK"
2. Aceite os termos de licença
3. Clique em "Instalar"
4. Na configuração de conexão com o banco de dados, use:
   - Servidor SQL: `mysql` (nome do serviço no docker-compose)
   - Usuário SQL: `glpi`
   - Senha SQL: `glpi`

5. Selecione o banco de dados `glpi`
6. Após a instalação, exclua o arquivo install/install.php conforme solicitado:
   ```bash
   docker exec glpi rm -rf /var/www/html/glpi/install/install.php
   ```

7. Acesse com as credenciais padrão:
   - Usuário: `glpi`
   - Senha: `glpi`

### 4.3. Acesso ao phpMyAdmin
1. Acesse `http://localhost:8081`
2. Use as credenciais:
   - Usuário: `root`
   - Senha: `glpi`

---

## Parte 5: Verificações técnicas e exploração

### 5.1. Verificação de rede
```bash
# Listar as redes Docker
docker network ls

# Inspecionar a rede glpi-network
docker network inspect glpi-network

# Verificar as portas em uso
sudo netstat -tulpn | grep -E '8080|8081'
```

### 5.2. Exploração de contêineres
```bash
# Acessar o shell do contêiner GLPI
docker exec -it glpi bash

# Verificar a configuração do Apache
cat /etc/apache2/sites-available/000-default.conf

# Verificar arquivos do GLPI
ls -la /var/www/html/glpi

# Verificar logs do Apache
tail -f /var/log/apache2/error.log
```

### 5.3. Comandos úteis para gerenciamento
```bash
# Parar todos os contêineres
docker compose down

# Reiniciar os contêineres
docker compose restart

# Verificar uso de recursos
docker stats
```

---

## Parte 6: Resolução de problemas comuns

### 6.1. GLPI não está acessível
- Verifique se todos os contêineres estão em execução: `docker ps`
- Verifique se as portas estão disponíveis: `sudo netstat -tulpn | grep -E '8080|8081'`
- Verifique os logs do contêiner GLPI: `docker logs glpi`
- Verifique se há regras de firewall bloqueando as portas: `sudo ufw status`

### 6.2. Problemas de conexão com o banco de dados
- Verifique se o contêiner MySQL está em execução: `docker ps | grep mysql`
- Verifique os logs do MySQL: `docker logs glpi-mysql`
- Tente conectar manualmente ao banco:
  ```bash
  docker exec -it glpi-mysql mysql -u glpi -pglpi -h localhost glpi
  ```

### 6.3. Problemas de permissão
- Verifique as permissões dos volumes:
  ```bash
  ls -la ~/glpi-docker/volumes/
  ```
- Ajuste permissões se necessário:
  ```bash
  sudo chown -R $USER:$USER ~/glpi-docker/volumes/
  ```

---
