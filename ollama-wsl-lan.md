# Configurando Ollama no WSL para Acesso na Rede Local

Este guia explica como configurar o Ollama, rodando dentro do WSL (Windows Subsystem for Linux), para que ele possa ser acessado por outras máquinas na sua rede local. Isso envolve configurar o Ollama para ouvir em todas as interfaces, configurar o encaminhamento de porta no Windows e ajustar o firewall do Windows.

**Pré-requisitos:**

*   WSL 2 instalado e configurado.
*   Ollama instalado no WSL (preferencialmente como um serviço systemd `ollama.service`).
*   Acesso de Administrador no Windows para configurar o encaminhamento de porta e o firewall.
*   Porta padrão do Ollama: `11434`.

**Passos:**

**1. Configurar Ollama para Ouvir em Todas as Interfaces (`0.0.0.0`)**

   Por padrão, Ollama ouve apenas em `localhost` (`127.0.0.1`), impedindo acessos externos. Precisamos mudar isso para `0.0.0.0`. A forma mais robusta é via `systemd`.

   *   Edite a unidade de serviço do Ollama usando `systemctl edit` (cria um arquivo de override seguro):

```
     # No terminal WSL

     sudo systemctl edit ollama.service
```
   *   No editor que abrir, insira o seguinte conteúdo:
```
     [Service]

     Environment="OLLAMA_HOST=0.0.0.0"

     # Se precisar permitir acesso de navegadores (CORS) de qualquer origem:
     
     # Environment="OLLAMA_ORIGINS=http://*"
```
     
     *Nota:* Ajuste `OLLAMA_ORIGINS` para URLs específicas se precisar de mais segurança ao acessar via navegador. Para acesso via API/CLI de backends, `OLLAMA_HOST` geralmente é suficiente.
   *   Salve e feche o editor.
   *   Recarregue a configuração do systemd e reinicie o Ollama:

```
     # No terminal WSL
     sudo systemctl daemon-reload
     sudo systemctl restart ollama.service

```
   *   **Alternativa (Não Persistente/Sem Systemd):** Se você inicia manualmente (`ollama serve`), defina a variável antes:

```
     # No terminal WSL (apenas para a sessão atual)
     export OLLAMA_HOST=0.0.0.0
     ollama serve

```

**2. Verificar se Ollama está Ouvindo Corretamente (Dentro do WSL)**

   *   Verifique em qual endereço e porta o Ollama está ouvindo:

```
# No terminal WSL

sudo ss -tulnp | grep ollama

# Ou, se o grep acima não funcionar, procure pela porta:

# sudo ss -tulnp | grep 11434
```
   *   A saída deve mostrar um processo `ollama` ouvindo (`LISTEN`) em `0.0.0.0:11434` ou `*:11434`. Se ainda mostrar `127.0.0.1:11434`, a configuração do `OLLAMA_HOST` não foi aplicada corretamente (revise o Passo 1).

**3. Obter Endereços IP**

   Você precisará do IP interno do WSL e do IP do seu computador Windows na rede local.

   *   **IP do WSL:** No terminal WSL:

```
     hostname -I

```
     Anote este IP (ex: `172.x.x.x`). **Lembre-se que ele pode mudar!**
   *   **IP do Windows:** Abra o Prompt de Comando ou PowerShell no *Windows* (não no WSL):

```
     ipconfig

```
     Procure o "Endereço IPv4" da sua conexão de rede ativa (Wi-Fi ou Ethernet, ex: `192.168.1.x`). Este é o IP que as outras máquinas usarão.

**4. Configurar Encaminhamento de Porta no Windows**

   Direcione o tráfego que chega na porta `11434` do Windows para a porta `11434` do IP do WSL.

   *   Abra o **PowerShell** ou **Prompt de Comando** como **Administrador** no Windows.
   *   Execute o comando `netsh`, substituindo `<IP_DO_WSL>` pelo IP obtido no passo anterior:

```
     netsh interface portproxy add v4tov4 listenport=11434 listenaddress=0.0.0.0 connectport=11434 connectaddress=<IP_DO_WSL>

```
   *   Para verificar as regras: `netsh interface portproxy show all`
   *   Para remover (se precisar atualizar o IP): `netsh interface portproxy delete v4tov4 listenport=11434 listenaddress=0.0.0.0`

**5. Configurar o Firewall do Windows**

   Permita conexões de entrada na porta `11434`.

   *   Abra o **PowerShell** como **Administrador** no Windows.
   *   Execute o comando:

```
     New-NetFirewallRule -Name "WSL Ollama (Port 11434)" -DisplayName "WSL Ollama (Port 11434)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 11434

```

**6. Conectar de Outra Máquina**

   Na outra máquina da sua rede local, configure seu cliente ou aplicação para usar a API do Ollama apontando para o endereço IP do seu computador Windows.

   *   **Exemplo com a CLI do Ollama (em outra máquina):**

```
     # Defina a variável de ambiente para apontar para o host Windows

 export OLLAMA_HOST=http://<IP_DO_WINDOWS_HOST>:11434

     # Agora use os comandos ollama normalmente
     ollama list

ollama run nome-do-modelo "Seu prompt aqui"

```
     Substitua `<IP_DO_WINDOWS_HOST>` pelo IP do Windows na rede local (passo 3).

   *   **Exemplo com `curl` (para testar a API):**

```

curl http://<IP_DO_WINDOWS_HOST>:11434/api/tags

```

## Problemas Comuns e Soluções

*   **Problema:** Conexão recusada (`Connection refused`) ou Timeout ao tentar conectar da outra máquina.
    *   **Causas Possíveis:**
        *   Serviço Ollama não está rodando no WSL (`sudo systemctl status ollama.service`).
        *   Ollama ainda está ouvindo apenas em `127.0.0.1` (verifique com `ss` no Passo 2; garanta que `OLLAMA_HOST=0.0.0.0` foi aplicado e o serviço reiniciado).
        *   Firewall do Windows bloqueando a porta `11434` (verifique a regra do Passo 5).
        *   Regra de encaminhamento `netsh` incorreta ou apontando para um IP do WSL antigo (verifique e atualize no Passo 4).
        *   IP do Host Windows incorreto sendo usado no cliente.

*   **Problema:** Acesso funciona via `curl` ou CLI, mas falha em um aplicativo web (JavaScript no navegador) com erros de CORS ou "Forbidden".
    *   **Causa:** O navegador bloqueia requisições cross-origin por padrão. O servidor Ollama precisa explicitamente permitir a origem do seu aplicativo web.
    *   **Solução:** Configure a variável `OLLAMA_ORIGINS` no serviço Ollama (Passo 1) para incluir a URL do seu aplicativo web, ou use um valor mais permissivo como `http://*` (menos seguro) durante o desenvolvimento. Lembre-se de rodar `sudo systemctl daemon-reload` e `sudo systemctl restart ollama.service` após editar a configuração.

*   **Problema:** A conexão funciona, mas para de funcionar após reiniciar o PC ou o WSL.
    *   **Causa Provável:** O endereço IP interno do WSL mudou.
    *   **Solução:**
        1.  Obtenha o novo IP do WSL (Passo 3: `hostname -I`).
        2.  No PowerShell do Windows (Admin), remova a regra antiga e adicione a nova com o IP atualizado (Passo 4):
            ```powershell
            netsh interface portproxy delete v4tov4 listenport=11434 listenaddress=0.0.0.0
            netsh interface portproxy add v4tov4 listenport=11434 listenaddress=0.0.0.0 connectport=11434 connectaddress=<NOVO_IP_DO_WSL>
            ```

**Considerações Importantes:**

*   **IP Dinâmico do WSL:** Esteja ciente da necessidade de atualizar a regra `netsh portproxy` quando o IP do WSL mudar.
*   **Segurança:** Expor Ollama na rede local permite que qualquer dispositivo na rede o acesse. Ollama não possui autenticação robusta por padrão. Use em redes confiáveis.
*   **Recursos:** Modelos de linguagem grandes consomem muita RAM e podem exigir GPU. Certifique-se de que sua máquina host Windows tem recursos suficientes alocados para o WSL.
