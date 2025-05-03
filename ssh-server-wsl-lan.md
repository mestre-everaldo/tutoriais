
# Configurando Servidor SSH no WSL para Acesso na Rede Local

Este guia detalha como instalar e configurar um servidor OpenSSH dentro do ambiente WSL (Windows Subsystem for Linux), permitindo o acesso via SSH a partir de outras máquinas na sua rede local. Abordaremos a configuração da porta, o encaminhamento no Windows, o firewall e um problema comum relacionado à ativação por socket do systemd.

**Pré-requisitos:**

*   WSL 2 instalado e configurado.
*   Uma distribuição Linux baseada em Debian/Ubuntu (ex: Ubuntu) rodando no WSL 2 (comandos `apt` e `systemd` serão usados). Adapte para outras distribuições se necessário.
*   Acesso de Administrador no Windows para configurar o encaminhamento de porta e o firewall.

**Passos:**

**1. Instalar o Servidor OpenSSH no WSL**

   Abra seu terminal WSL e execute:
   ```
   sudo apt update
   sudo apt install openssh-server -y
   ```

**2. Configurar o Servidor SSH (`sshd_config`)**

   É recomendado alterar a porta padrão (22) para evitar conflitos e por segurança. Usaremos a porta `2222` como exemplo.

   *   Edite o arquivo de configuração:
     ```
     sudo nano /etc/ssh/sshd_config
     ```
   *   Localize a linha `#Port 22` ou `Port 22`. Remova o `#` (descomente) e altere o número da porta:
     ```
     Port 2222
     ```
   *   Verifique se a autenticação por senha está habilitada (para o primeiro acesso; considere usar chaves SSH depois):
     Procure por `PasswordAuthentication`. Garanta que esteja descomentada e definida como `yes`:
     ```
     PasswordAuthentication yes
     ```
   *   Salve o arquivo (Ctrl+O, Enter no `nano`) e saia (Ctrl+X).
   *   Teste a sintaxe da configuração para garantir que não há erros:
     ```
     sudo sshd -t
     ```
     Se nenhum erro for exibido, a sintaxe está correta.

**3. Desabilitar a Ativação por Socket (Importante!)**

   Muitas distribuições modernas usam `systemd` com *ativação por socket* para o SSH (`ssh.socket`). Isso faz com que o `systemd` ouça na porta definida no *socket* (geralmente a padrão 22), ignorando a configuração `Port` no `sshd_config`. Para garantir que o `sshd_config` seja respeitado:

   *   Pare e desabilite o serviço de socket:

```

sudo systemctl stop ssh.socket

sudo systemctl disable ssh.socket

```

**4. Iniciar e Habilitar o Serviço SSH (`ssh.service`)**

   Agora que o socket está desabilitado, podemos iniciar o serviço principal, que lerá a configuração do `sshd_config`.

   *   Reinicie (ou inicie) o serviço SSH:
     ```
     sudo systemctl restart ssh.service
     ```
   *   (Opcional) Habilite o serviço para iniciar automaticamente com o WSL:
     ```
     sudo systemctl enable ssh.service
     ```

**5. Verificar o Status e a Porta de Escuta (Dentro do WSL)**

   *   Verifique se o serviço está ativo e rodando:
     ```
     sudo systemctl status ssh.service
     ```
     Procure por `Active: active (running)`.
   *   Verifique em qual porta o `sshd` está ouvindo:
     ```
     sudo ss -tlpn | grep sshd
     ```
     A saída deve mostrar o processo `sshd` ouvindo (`LISTEN`) em `0.0.0.0:2222` ou `*:2222` (e talvez `:::2222` para IPv6). Se ainda mostrar a porta 22, revise os passos 2 e 3.

**6. Obter Endereços IP**

   Você precisará do IP interno do WSL e do IP do seu computador Windows na rede local.

   *   **IP do WSL:** No terminal WSL:
     ```
     hostname -I
     ```
     Anote este IP (ex: `172.x.x.x`). **Ele pode mudar após reinicializações!**
   *   **IP do Windows:** Abra o Prompt de Comando ou PowerShell no *Windows* (não no WSL):
     ```
     ipconfig
     ```
     Procure o "Endereço IPv4" da sua conexão de rede ativa (Wi-Fi ou Ethernet, ex: `192.168.1.x`).

**7. Configurar Encaminhamento de Porta no Windows**

   Direcione o tráfego que chega na porta `2222` do Windows para a porta `2222` do IP do WSL.

   *   Abra o **PowerShell** ou **Prompt de Comando** como **Administrador** no Windows.
   *   Execute o comando `netsh`, substituindo `<IP_DO_WSL>` pelo IP obtido no passo anterior:
     ```
     netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=2222 connectaddress=<IP_DO_WSL>
     ```
   *   Para verificar as regras: `netsh interface portproxy show all`
   *   Para remover (se precisar atualizar o IP): `netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=0.0.0.0`

**8. Configurar o Firewall do Windows**

   Permita conexões de entrada na porta `2222`.

   *   Abra o **PowerShell** como **Administrador** no Windows.
   *   Execute o comando:
     ```
     New-NetFirewallRule -Name "WSL SSH (Port 2222)" -DisplayName "WSL SSH (Port 2222)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 2222
     ```

**9. Conectar de Outra Máquina**

   Na outra máquina da sua rede local, use um cliente SSH:

   ```
   ssh <seu_usuario_wsl>@<IP_DO_WINDOWS_HOST> -p 2222
   ```

   *   Substitua `<seu_usuario_wsl>` pelo seu nome de usuário no WSL.
   *   Substitua `<IP_DO_WINDOWS_HOST>` pelo IP do seu PC Windows na rede local (passo 6).

   Na primeira conexão, você será solicitado a aceitar a chave do host. Depois, digite a senha do seu usuário WSL.

## Problemas Comuns e Soluções

*   **Problema:** O servidor SSH continua ouvindo na porta 22, mesmo após editar `/etc/ssh/sshd_config` e reiniciar o serviço (`sudo systemctl restart ssh`).
    *   **Causa Provável:** A ativação por socket (`ssh.socket`) está ativa e tem precedência.
    *   **Inspeção:** Verifique o status do serviço SSH: `sudo systemctl status ssh.service`. Se a linha `Loaded:` mencionar `ssh.socket` ou se houver uma linha como `TriggeredBy: ● ssh.socket`, a ativação por socket está em uso. Verifique também se o socket está ativo: `sudo systemctl status ssh.socket`.
    *   **Solução:** Pare e desabilite permanentemente o socket como descrito no **Passo 3**:
        ```
        sudo systemctl stop ssh.socket
        sudo systemctl disable ssh.socket
        sudo systemctl restart ssh.service # Para garantir que o serviço agora leia o sshd_config
        sudo ss -tlpn | grep sshd          # Verificar se está ouvindo na porta correta (2222)
        ```

*   **Problema:** Conexão recusada (`Connection refused`).
    *   **Causas Possíveis:**
        *   Serviço SSH não está rodando no WSL (`sudo systemctl status ssh.service`).
        *   Firewall do Windows bloqueando a porta `2222` (verifique a regra criada no Passo 8).
        *   Regra de encaminhamento `netsh` não existe ou está incorreta.
        *   Você está tentando conectar no IP errado (use o IP do Windows, não o do WSL).
        *   Firewall *dentro* do WSL (como `ufw`) bloqueando a porta `2222`. Verifique com `sudo ufw status` e permita com `sudo ufw allow 2222/tcp` se necessário.

*   **Problema:** A conexão funciona, mas para de funcionar após reiniciar o PC ou o WSL (`Connection reset by peer`, `kex_exchange_identification`, ou timeout).
    *   **Causa Provável:** O endereço IP interno do WSL mudou.
    *   **Solução:**
        1.  Obtenha o novo IP do WSL (Passo 6: `hostname -I`).
        2.  No PowerShell do Windows (Admin), remova a regra antiga e adicione a nova com o IP atualizado (Passo 7):
            ```
            netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=0.0.0.0
            netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=2222 connectaddress=<NOVO_IP_DO_WSL>
            ```

*   **Problema:** `Permission denied (publickey,password)`.
    *   **Causas Possíveis:**
        *   Senha incorreta para o usuário WSL.
        *   A diretiva `PasswordAuthentication` está definida como `no` em `/etc/ssh/sshd_config` e você não configurou autenticação por chave SSH.

**Considerações Importantes:**

*   **IP Dinâmico do WSL:** A necessidade de atualizar a regra `netsh portproxy` pode ser frequente. Existem scripts para automatizar isso, ou pode-se investigar o uso de `connectaddress=localhost` na regra `netsh` (funciona em versões mais recentes do Windows/WSL).
*   **Segurança:** Autenticação por senha pela rede é menos segura. Após configurar o acesso inicial, considere fortemente configurar a autenticação baseada em chaves SSH e desabilitar a autenticação por senha (`PasswordAuthentication no` no `sshd_config`).
*   **Firewall:** Certifique-se de que não há outros firewalls (antivírus, etc.) no Windows bloqueando a porta. O perfil de rede do Windows (Público/Privado) também pode influenciar.
