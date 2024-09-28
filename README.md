# Guia de Implantação de Aplicativo Django no DigitalOcean

- [Pré-requisitos](#pré-requisitos)
- [Configuração Inicial do Servidor](#configuração-inicial-do-servidor)
- [Instalação de Dependências](#instalação-de-dependências)
- [Configuração do Banco de Dados PostgreSQL](#configuração-do-banco-de-dados-postgresql)
- [Configuração do Ambiente Virtual e do Projeto Django](#configuração-do-ambiente-virtual-e-do-projeto-django)
- [Configuração do Gunicorn](#configuração-do-gunicorn)
- [Configuração do Nginx](#configuração-do-nginx)
- [Configuração do Firewall](#configuração-do-firewall)
- [Configuração de SSL com Let's Encrypt (Opcional mas Recomendado)](#configuração-de-ssl-com-lets-encrypt-opcional-mas-recomendado)
- [Testando a Aplicação](#testando-a-aplicação)
- [Automatizando Tarefas com Supervisor (Opcional)](#automatizando-tarefas-com-supervisor-opcional)
- [Considerações Finais](#considerações-finais)

## 1. Pré-requisitos
Antes de começar, certifique-se de ter o seguinte:

Conta no DigitalOcean: Acesse sua conta ou crie uma nova em digitalocean.com.
Chave SSH configurada: Recomenda-se usar chaves SSH para autenticação segura. Guia de configuração de chaves SSH.
Aplicativo Django pronto para implantação: Certifique-se de que seu aplicativo está funcionando corretamente em ambiente de desenvolvimento.
Conhecimento básico de linha de comando e administração de sistemas Linux.

## 2. Configuração Inicial do Servidor
### 2.1. Criação do Droplet no DigitalOcean
Login no DigitalOcean e navegue até o painel de controle.
Crie um novo Droplet:
Selecione a imagem do SO: Escolha Ubuntu 22.04 LTS (ou a versão mais recente disponível).
Escolha o tamanho do Droplet: Selecione de acordo com as necessidades do seu aplicativo. Para aplicativos pequenos, um Droplet básico é suficiente.
Escolha a região: Selecione uma região próxima aos seus usuários.
Autenticação: Adicione sua chave SSH para acesso seguro.
Finalize a criação: Dê um nome ao seu Droplet e clique em Create Droplet.
### 2.2. Acessando o Servidor via SSH
No seu terminal local, execute:

```bash
ssh root@seu_endereco_ip
```
Substitua seu_endereco_ip pelo endereço IP do seu Droplet.

### 2.3. Atualizando o Servidor
Após acessar o servidor, atualize os pacotes existentes:

```bash
apt update && apt upgrade -y
```

### 2.4. Criando um Novo Usuário Sudo
Por segurança, é recomendado não usar o usuário root diretamente.

Crie um novo usuário:

```bash
adduser seu_usuario
```

Adicione o usuário ao grupo sudo:


```bash
usermod -aG sudo seu_usuario
```

Copie as chaves SSH para o novo usuário:


```bash
rsync --archive --chown=seu_usuario:seu_usuario ~/.ssh /home/seu_usuario
```

Saia do usuário root e acesse com o novo usuário:


```bash
exit
ssh seu_usuario@seu_endereco_ip
```
## 3. Instalação de Dependências
### 3.1. Instalando Python e Pip

Instale Python 3 e pip:


```bash
sudo apt install python3 python3-pip python3-venv -y
```

### 3.2. Instalando Git
Caso você vá clonar seu projeto de um repositório Git:


```bash
sudo apt install git -y
```

### 3.3. Instalando Nginx
Nginx será usado como servidor web reverso:

```bash
sudo apt install nginx -y
```

## 4. Configuração do Banco de Dados MySQL

### 4.1. Instalação do MySQL:

Primeiramente, instale o MySQL no servidor:

```bash
sudo apt-get update
sudo apt-get install mysql-server
```

### 4.2. Segurança do MySQL:

Após a instalação, execute o script de segurança do MySQL para remover algumas configurações padrão inseguras e proteger o acesso ao banco de dados:

```bash
sudo mysql_secure_installation
```

Siga as instruções no terminal para definir uma senha de root, remover usuários anônimos, desabilitar o login root remotamente, e remover o banco de dados de teste.

### 4.3. Criação de um Banco de Dados e Usuário MySQL:

Acesse o MySQL com a conta root:

```bash
sudo mysql -u root -p
```

### 4.4. Dentro do console MySQL, execute os seguintes comandos para criar um banco de dados e um usuário:

```sql
CREATE DATABASE room_reservation CHARACTER SET UTF8MB4 COLLATE utf8mb4_general_ci;
CREATE USER 'room_user'@'localhost' IDENTIFIED BY 'sua_senha_secreta';
GRANT ALL PRIVILEGES ON room_reservation.* TO 'room_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Substitua room_reservation, room_user, e sua_senha_secreta pelos nomes e senha de sua escolha.

### 4.5. Configuração do Django para usar o MySQL:

No arquivo settings.py do seu projeto Django, configure o banco de dados da seguinte forma:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'room_reservation',
        'USER': 'room_user',
        'PASSWORD': 'sua_senha_secreta',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```
Em caso de utilização de .env atualize nela os valores.

### 4.6. Instalação do MySQL Client:

Para que o Django possa se comunicar com o MySQL, instale o conector MySQL:

```bash
sudo apt-get install libmysqlclient-dev
```
```bash
sudo apt-get install python3-dev default-libmysqlclient-dev build-essential pkg-config
```

```bash
pip install mysqlclient
```

### 4.7. Aplicar Migrações:

Finalmente, aplique as migrações para configurar o banco de dados:

```bash
python manage.py migrate
```

## 5. Configuração do Ambiente Virtual e do Projeto Django
### 5.1. Configurando o Diretório do Projeto em /var/www/
Crie o diretório do projeto:

```bash
sudo mkdir -p /var/www/seu_projeto
```

Atribua permissões ao seu usuário:

```bash
sudo chown -R seu_usuario:seu_usuario /var/www/seu_projeto
```

### 5.2. Clonando seu Projeto
Navegue até o diretório do projeto:

```bash
cd /var/www/seu_projeto
```

Clone seu repositório Git:

```bash
git clone https://github.com/seu_usuario/seu_repositorio.git .
```

Ou copie seus arquivos manualmente para este diretório.

### 5.3. Criando e Ativando o Ambiente Virtual
Crie o ambiente virtual:

```bash
python3 -m venv venv
```

Ative o ambiente virtual:

```bash
source venv/bin/activate
```

### 5.4. Instalando as Dependências do Projeto
Atualize o pip:

```bash
pip install --upgrade pip
```

Instale as dependências:

```bash
pip install -r requirements.txt
```

Nota: Certifique-se de que seu arquivo requirements.txt esteja atualizado com todas as dependências necessárias.

### 5.5. Configurando as Variáveis de Ambiente
É uma boa prática manter informações sensíveis, como configurações de banco de dados e chaves secretas, em variáveis de ambiente.

Instale o python-dotenv (se ainda não estiver instalado):

```bash
pip install python-dotenv
```

Crie um arquivo .env no diretório do seu projeto:

```bash
nano /var/www/seu_projeto/.env
```

Adicione as seguintes configurações:

```bash
DEBUG=False
SECRET_KEY=sua_chave_secreta
ALLOWED_HOSTS=seu_endereco_ip ou seu_dominio
DB_NAME=nome_do_seu_banco
DB_USER=seu_usuario_banco
DB_PASSWORD=sua_senha_segura
DB_HOST=localhost
DB_PORT=5432
```

Configure seu settings.py para usar essas variáveis:

```python
# settings.py
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY')

DEBUG = os.getenv('DEBUG') == 'True'

ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS').split(',')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT'),
    }
}
```

### 5.6. Aplicando Migrações e Coletando Arquivos Estáticos
Aplique as migrações do banco de dados:

```bash
python manage.py migrate
```

Crie um usuário administrador (opcional):

```bash
python manage.py createsuperuser
```

Colete os arquivos estáticos:

```bash
python manage.py collectstatic
```

Nota: Quando solicitado, digite yes para confirmar.

## 6. Configuração do Gunicorn
O Gunicorn é um servidor WSGI Python que servirá sua aplicação Django.

### 6.1. Instalando o Gunicorn
Com o ambiente virtual ainda ativo:

```bash
pip install gunicorn
```

### 6.2. Criando um Socket Systemd para o Gunicorn
Crie e edite o arquivo de serviço do Gunicorn:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

Adicione o seguinte conteúdo:

```ini
[Unit]
Description=gunicorn daemon for Seu_Projeto
After=network.target

[Service]
User=Seu User
Group=www-data
WorkingDirectory=/var/www/Seu_Projeto/sistema
ExecStart=/var/www/Seu_Projeto/venv/bin/gunicorn  --workers 3 --bind unix:/var/www/Seu_Projeto/sistema/gunicorn.sock setup.wsgi:application
Restart=always

[Install]
WantedBy=multi-user.target
```

Substitua:

seu_usuario pelo nome do seu usuário.
seu_projeto pelo nome real do seu projeto.
Certifique-se de que o caminho para o arquivo wsgi.py esteja correto (seu_projeto.wsgi:application).


### 6.3. Iniciando e Habilitando o Serviço do Gunicorn
Recarregue o daemon do systemd:

```bash
sudo systemctl daemon-reload
```

Inicie o serviço do Gunicorn:

```bash
sudo systemctl start gunicorn
```

Habilite o serviço para iniciar automaticamente na inicialização:

```bash
sudo systemctl enable gunicorn
```

Verifique o status do serviço:

```bash
sudo systemctl status gunicorn
```

Saída esperada: O serviço deve estar ativo e em execução sem erros.

## 7. Configuração do Nginx
O Nginx atuará como um proxy reverso, encaminhando solicitações ao Gunicorn.

### 7.1. Configurando o Nginx
Crie um arquivo de configuração para o seu projeto:

```bash
sudo nano /etc/nginx/sites-available/seu_projeto
```

Adicione o seguinte conteúdo:

```nginx
server {
    listen 80;
    server_name 10.0.0.1;  # Substitua pelo seu domínio ou IP

    location / {
        proxy_pass http://unix:/var/www/RoomReservation/sistema/gunicorn.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/RoomReservation/sistema/static/;
    }

    location /media/ {
        alias /var/www/RoomReservation/sistema/media/;
    }
}
```

Substitua:

seu_dominio ou seu_endereco_ip pelo seu domínio real ou endereço IP.
Certifique-se de que os caminhos para static e media estejam corretos conforme a estrutura do seu projeto.
Ative a configuração criando um link simbólico:

```bash
sudo ln -s /etc/nginx/sites-available/seu_projeto /etc/nginx/sites-enabled
```

Teste a configuração do Nginx:

```bash
sudo nginx -t
```

Reinicie o Nginx:

```bash
sudo systemctl restart nginx
```

### 7.2. Configurando Permissões
Dê permissões apropriadas ao diretório do projeto:

```bash
sudo chown -R :www-data /var/www/seu_projeto
sudo chmod -R 775 /var/www/seu_projeto
```

Isso garante que o Nginx (executando como www-data) tenha acesso aos arquivos estáticos e de mídia.

## 8. Configuração do Firewall
Recomenda-se usar o ufw (Uncomplicated Firewall) para gerenciar o firewall.

### 8.1. Permitir Conexões Necessárias
Permita conexões SSH:

```bash
sudo ufw allow OpenSSH
```

Permita conexões HTTP:

```bash
sudo ufw allow 'Nginx Full'
```

Habilite o firewall:

```bash
sudo ufw enable
```

Verifique o status do firewall:

```bash
sudo ufw status
```

Saída esperada: Deverá mostrar que as conexões para OpenSSH e Nginx estão permitidas.

## 9. Configuração de SSL com Let's Encrypt (Opcional mas Recomendado)
Para garantir que as conexões ao seu site sejam seguras, você pode configurar certificados SSL gratuitos usando o Let's Encrypt.

### 9.1. Instalando o Certbot
Instale o Certbot e o plugin Nginx:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 9.2. Obtendo e Instalando o Certificado SSL
Execute o Certbot:

```bash
sudo certbot --nginx -d seu_dominio -d www.seu_dominio
```

Siga as instruções e forneça um e-mail para notificações. O Certbot irá configurar automaticamente o Nginx para usar SSL.

### 9.3. Renovação Automática
O Certbot configura automaticamente a renovação automática de certificados. Você pode testar a renovação com:

```bash
sudo certbot renew --dry-run
```

## 10. Testando a Aplicação
Após concluir todas as etapas, acesse seu domínio ou endereço IP no navegador:

Sem SSL: http://seu_dominio ou http://seu_endereco_ip
Com SSL: https://seu_dominio

Verifique se:

A página inicial do seu aplicativo carrega corretamente.
Os arquivos estáticos (CSS, JS, imagens) estão sendo carregados.
As funcionalidades principais do aplicativo estão funcionando.
O painel de administração do Django está acessível (se aplicável).


## 11. Automatizando Tarefas com Supervisor (Opcional)
Se você tiver tarefas em background ou processos longos, pode usar o Supervisor para gerenciá-los.

### 11.1. Instalando o Supervisor
```bash
sudo apt install supervisor -y
```

### 11.2. Configurando uma Tarefa no Supervisor
Crie um arquivo de configuração:

```bash
sudo nano /etc/supervisor/conf.d/seu_projeto.conf
```

Adicione o seguinte conteúdo:

```ini
[program:seu_projeto]
command=/var/www/seu_projeto/venv/bin/python /var/www/seu_projeto/manage.py sua_tarefa
directory=/var/www/seu_projeto
user=seu_usuario
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/seu_projeto.log
stderr_logfile=/var/log/supervisor/seu_projeto_err.log
```

Recarregue o Supervisor:

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

## 12. Considerações Finais
Segurança: Monitore regularmente seu servidor quanto a atualizações de segurança e mantenha seus pacotes atualizados.
Backups: Configure backups regulares de seu banco de dados e arquivos importantes.
Monitoramento: Considere implementar ferramentas de monitoramento para acompanhar o desempenho e a disponibilidade de sua aplicação.
Escalabilidade: Planeje como você irá escalar sua aplicação conforme o tráfego e as demandas aumentam.
Este guia fornece uma estrutura sólida para implantar seu aplicativo Django em um servidor DigitalOcean de forma eficiente e segura. Guarde este passo a passo para futuras referências e adapte conforme as necessidades específicas do seu projeto evoluírem.

Se você encontrar algum erro ou tiver dúvidas durante o processo, não hesite em consultar a documentação oficial ou buscar ajuda em comunidades de desenvolvedores.

Boa sorte com sua implantação!
