# 🖥️ Acesso Remoto ao Ubuntu Server via SSH

## 📋 Pré-requisitos
- Ubuntu Server instalado e funcionando
- Acesso administrativo (sudo)
- Acesso ao painel do roteador
- Cliente SSH no dispositivo remoto

## 🛠️ 1. Instalação e Configuração Inicial do SSH

### Instalar o OpenSSH Server
```bash
sudo apt update
sudo apt install openssh-server -y
```

### Verificar status e ativar o serviço
```bash
sudo systemctl status ssh
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Fazer backup da configuração original
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

## 🌐 2. Configuração de Rede

### Descobrir endereços IP
```bash
# IP interno da máquina
ip a
# ou
hostname -I

# IP externo/público
curl ifconfig.me
# ou
curl icanhazip.com
```

### Testar conexão local primeiro
```bash
# Teste dentro da rede local
ssh usuario@IP_INTERNO
```

## 🔧 3. Configuração do Port Forwarding no Roteador

**Acesse o painel do roteador** (geralmente `192.168.0.1` ou `192.168.1.1`):

| Campo | Valor |
|-------|-------|
| **Porta Externa** | `2222` |
| **Porta Interna** | `22` |
| **Protocolo** | `TCP` |
| **IP Destino** | `IP_INTERNO_DO_SERVIDOR` |

> ⚠️ **Importante**: Mantenha o SSH na porta 22 no servidor e mapeie para 2222 externamente. Isso evita conflitos.

## 🔒 4. Configurações de Segurança Essenciais

### Editar configuração do SSH
```bash
sudo nano /etc/ssh/sshd_config
```

### Aplicar configurações de segurança
```bash
# Desabilitar login root
PermitRootLogin no

# Limitar tentativas de login
MaxAuthTries 3
MaxSessions 2

# Timeout de sessão
ClientAliveInterval 300
ClientAliveCountMax 2

# Protocolo SSH v2 apenas
Protocol 2

# Desabilitar autenticação por senha vazia
PermitEmptyPasswords no

# Configurações adicionais de segurança
X11Forwarding no
AllowUsers SEU_USUARIO  # Substitua pelo seu usuário
```

### Reiniciar SSH após configuração
```bash
sudo systemctl restart ssh
sudo systemctl status ssh  # Verificar se reiniciou sem erros
```

## 🔑 5. Autenticação por Chaves SSH 

### No cliente (máquina que vai conectar)
```bash
# Gerar par de chaves
ssh-keygen -t rsa -b 4096 -C "seu_email@exemplo.com"

# Copiar chave pública para o servidor
ssh-copy-id -p 2222 usuario@SEU_IP_PUBLICO
```

### No servidor, após copiar a chave
```bash
# Editar configuração SSH
sudo nano /etc/ssh/sshd_config

# Desabilitar autenticação por senha
PasswordAuthentication no
PubkeyAuthentication yes

# Reiniciar SSH
sudo systemctl restart ssh
```

## 🛡️ 6. Proteção Adicional com Fail2Ban

### Instalar e configurar
```bash
sudo apt install fail2ban -y

# Criar configuração personalizada
sudo nano /etc/fail2ban/jail.local
```

### Conteúdo do jail.local
```ini
[DEFAULT]
bantime = 1800
findtime = 600
maxretry = 3
ignoreip = 127.0.0.1/8 192.168.0.0/16

[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
banaction = iptables-multiport
```

### Ativar fail2ban
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd  # Verificar status
```

## 🔥 7. Configuração do Firewall (UFW)

```bash
# Instalar UFW (se não estiver instalado)
sudo apt install ufw -y

# Configurar regras básicas
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH
sudo ufw allow ssh
sudo ufw allow 22/tcp

# Permitir outras portas se necessário (ex: HTTP, HTTPS)
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp

# Ativar firewall
sudo ufw enable
sudo ufw status verbose
```

## 🌐 8. DNS Dinâmico 

### Domínio: seu_dominio.com
 
Objetivo: acessar https://seu_dominio.com e ele redirecionar para o container FastAPI

 
Usar um proxy reverso
 Como o Docker roda normalmente em uma porta específica (8000, 8080 etc.), precisamos de um proxy reverso para expor a app no subdomínio:


Instale o Nginx no servidor se não tiver:
```bash 
sudo apt update
sudo apt install nginx -y
````
Configure um server block para o subdomínio:
```bash 
server {
    listen 80;
    server_name seu_dominio.com;

    location / {
        proxy_pass http://127.0.0.1:8000;  # Porta do container FastAPI
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Teste a configuração e reinicie o Nginx:
```bash
sudo nginx -t
sudo systemctl restart nginx
```



## Habilitar HTTPS


Via Nginx: use Certbot:
```bash 
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d seu_dominio.com
```


## 🔌 9. Como Conectar

### Dentro da rede local
```bash
ssh usuario@IP_INTERNO
```

### Fora da rede (Internet)
```bash
# Com IP público
ssh -p 2222 usuario@SEU_IP_PUBLICO

# Com DNS dinâmico
ssh -p 2222 usuario@seu_dominio.com

# Com arquivo de chave específica
ssh -p 2222 -i ~/.ssh/id_rsa usuario@seu_dominio.com
```

### Criar alias para facilitar (opcional)
```bash
# No cliente, editar ~/.ssh/config
nano ~/.ssh/config

# Adicionar:
Host meuservidor
    HostName seu_dominio.com
    Port 2222
    User usuario
    IdentityFile ~/.ssh/id_rsa

# Conectar simplesmente com: ssh meuservidor
```

## 🔍 10. Monitoramento e Manutenção

### Verificar logs de conexão
```bash
# Últimas tentativas de login
sudo tail -f /var/log/auth.log

# IPs banidos pelo fail2ban
sudo fail2ban-client status sshd

# Status do SSH
sudo systemctl status ssh
```

### Comandos úteis de diagnóstico
```bash
# Verificar portas em uso
sudo netstat -tlnp | grep :22

# Testar configuração SSH
sudo sshd -t

# Ver usuários conectados
w
who
```

## 📊 11. Backup das Configurações

```bash
# Criar backup das configurações importantes
sudo tar czf ~/backup_ssh_$(date +%Y%m%d).tar.gz \
  /etc/ssh/sshd_config \
  /etc/fail2ban/jail.local \
  ~/.ssh/
```

## ⚠️ 12. Troubleshooting Comum

### SSH não conecta de fora da rede
1. Verificar se port forwarding está correto no roteador
2. Confirmar se o IP público está correto: `curl ifconfig.me`
3. Testar se a porta está aberta: `telnet SEU_IP_PUBLICO 2222`

### Falha na autenticação
1. Verificar se o usuário existe: `id usuario`
2. Conferir permissões da pasta SSH: `ls -la ~/.ssh/`
3. Verificar logs: `sudo tail -f /var/log/auth.log`

### Firewall bloqueia conexão
```bash
# Verificar regras UFW
sudo ufw status numbered
# Remover regra se necessário: sudo ufw delete NUMERO
```

## 🎯 Resumo de Comandos Rápidos

```bash
# Conectar de fora da rede
ssh -p 2222 usuario@SEU_IP_PUBLICO

# Verificar status dos serviços
sudo systemctl status ssh
sudo systemctl status fail2ban
sudo ufw status

# Ver tentativas de login
sudo tail -20 /var/log/auth.log

# Verificar IPs banidos
sudo fail2ban-client status sshd
```

---

 

