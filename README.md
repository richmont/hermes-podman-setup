# hermes-podman-setup

## Pré-requisitos do host

A máquina host precisa ter o Podman e o `podman-compose` instalados. O container
é executado em modo rootless pelo usuário dedicado `hermes`; os comandos de
gerenciamento devem ser executados nesse contexto, por exemplo com
`sudo -u hermes podman ...`.

## Inicialização pelo systemd

O serviço `hermes-compose` inicia o Compose no boot e o encerra ao parar o
serviço. Ele roda como o usuário e grupo `hermes`, com `/etc/hermes` como
diretório de trabalho:

```ini
[Unit]
Description=Hermes Agent - Podman Compose Rootless
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes

User=hermes
Group=hermes

Environment=HOME=/home/hermes
Environment=XDG_RUNTIME_DIR=/run/user/1001
Environment=PATH=/usr/local/bin:/usr/bin:/bin

WorkingDirectory=/etc/hermes
ExecStart=/usr/bin/podman-compose --in-pod=false -f /etc/hermes/compose.yaml up -d
ExecStop=/usr/bin/podman-compose --in-pod=false -f /etc/hermes/compose.yaml down

TimeoutStartSec=180
TimeoutStopSec=90

[Install]
WantedBy=multi-user.target
```

O arquivo da unidade fica em `/etc/systemd/system/hermes-compose.service`.
`XDG_RUNTIME_DIR` aponta para `/run/user/1001` neste host; esse valor depende do
UID atribuído ao usuário `hermes` no sistema.

## Reconstruir a imagem e recriar o container

Após alterar o `Containerfile` — por exemplo, para adicionar ferramentas como
`agy` e `codex` — reconstrua a imagem e recrie o container como usuário
`hermes`:

```bash
sudo -u hermes podman build -t localhost/hermes-opencode:2 -f /etc/hermes/Containerfile /etc/hermes
sudo -u hermes podman-compose --in-pod=false -f /etc/hermes/compose.yaml up -d --force-recreate
```

Para conferir se as ferramentas estão disponíveis dentro do container, execute
os comandos abaixo no host:

```bash
sudo -u hermes podman exec hermes-agent opencode --version
sudo -u hermes podman exec hermes-agent ccusage --version
sudo -u hermes podman exec hermes-agent codex --version
sudo -u hermes podman exec hermes-agent agy --version
sudo -u hermes podman exec hermes-agent tmux -V
```

## Acessar o terminal do container

Na máquina host, use o comando abaixo para abrir um Bash interativo no container
`hermes-agent`, executando o Podman como o usuário `hermes`:

```bash
sudo -u hermes podman exec -it hermes-agent /bin/bash
```

Com o terminal aberto, é possível realizar o login no Hermes ou executar comandos
diretamente no ambiente do container. Para sair e voltar ao host, use `exit`.

## Configuração da imagem

Os arquivos usados para configurar e executar o container ficam no host em
`/etc/hermes`:

```text
/etc/hermes/
├── Containerfile
└── compose.yaml
```

O `Containerfile` parte da imagem `docker.io/nousresearch/hermes-agent:stable`.
Como `root`, instala `curl`, `ca-certificates`, `tmux`, `git`, `procps`, `sudo`,
`tree` e `mlocate`, além das ferramentas Antigravity CLI (`agy`), Codex CLI
(`codex`), OpenCode (`opencode`) e `ccusage`. O executável do Codex é copiado
com os links simbólicos resolvidos para `/usr/local/bin/codex` e recebe
permissão de execução, tornando-o acessível ao usuário `hermes` também. No
final, a imagem volta para o usuário `hermes` e define `/workspace` como
diretório de trabalho.

O conteúdo usado para construir a imagem é:

```dockerfile
FROM docker.io/nousresearch/hermes-agent:stable

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl ca-certificates tmux git procps sudo tree mlocate \
    && rm -rf /var/lib/apt/lists/*

# Antigravity CLI (agy)
RUN curl -fsSL https://antigravity.google/cli/install.sh | bash \
    && mv /root/.local/bin/agy /usr/local/bin/agy \
    && agy --version

# Codex CLI (instalador standalone oficial)
RUN curl -fsSL https://chatgpt.com/codex/install.sh | sh \
    && cp -L /root/.local/bin/codex /usr/local/bin/codex \
    && chmod +x /usr/local/bin/codex \
    && codex --version

# OpenCode
RUN npm install -g --allow-scripts=@opencode/cli @opencode/cli \
    && ln -sf "$(npm root -g)/@opencode/cli/bin/opencode.exe" /usr/local/bin/opencode \
    && opencode --version \
    && tmux -V

# ccusage (estimativa de consumo de tokens)
RUN npm install -g ccusage \
    && chmod +x "$(npm root -g)/ccusage/src/cli.js" \
    && ln -sf "$(npm root -g)/ccusage/src/cli.js" /usr/local/bin/ccusage \
    && ccusage --version

USER hermes

WORKDIR /workspace
```

O `compose.yaml`, também em `/etc/hermes`, contém a configuração de execução do
container. A home `/home/hermes` no host foi criada para uso exclusivo do
container, sem permissões adicionais, e contém:

```text
/home/hermes/
├── config/
├── container/
├── data/
└── workspace/
```

Os diretórios `data` e `workspace` são montados no container em `/opt/data` e
`/workspace`, respectivamente. O compose também monta
`/home/hermes/data/.gemini/antigravity-cli` em
`/home/hermes/.gemini/antigravity-cli`. Como o gateway usa `HOME=/opt/data`,
seu caminho padrão para os dados do Antigravity CLI é `/opt/data/.gemini`; a
diferença entre esse caminho e o destino do mount está detalhada na seção de
autenticação abaixo. O diretório `/workspace` é usado como diretório de
trabalho.

O conteúdo do `compose.yaml` é:

```yaml
services:
  hermes:
    image: localhost/hermes-opencode:2
    container_name: hermes-agent

    userns_mode: "keep-id:uid=10000,gid=10000"
    user: "0:0"
    extra_hosts:
      - "host.example.invalid:host-gateway" # nome real do host omitido
    volumes:
      - /home/hermes/data:/opt/data
      - /home/hermes/workspace:/workspace
      - /home/hermes/data/.gemini/antigravity-cli:/home/hermes/.gemini/antigravity-cli
    environment:
      HERMES_DASHBOARD: "1"
      HERMES_DASHBOARD_HOST: "0.0.0.0"
      GH_TOKEN: "" # valor censurado
    ports:
      - "127.0.0.1:9119:9119"
    working_dir: /workspace

    command: ["gateway", "run"]
    restart: unless-stopped
```

O serviço usa a imagem local `localhost/hermes-opencode:2`, expõe o dashboard
na porta `9119` somente no endereço local do host e inicia o gateway. O
`GH_TOKEN` foi censurado neste registro. Para o GitHub funcionar corretamente,
o token também foi adicionado diretamente ao arquivo `config.yaml` do Hermes;
esse arquivo fica na configuração persistida do usuário/container e seu
conteúdo não foi informado aqui.

## Acessar a dashboard por túnel SSH

A dashboard está publicada pelo Podman apenas em `127.0.0.1:9119` no host, sem
escutar diretamente em interfaces externas. Para acessá-la a partir da máquina
local, abra um túnel SSH:

```bash
ssh -i priv.key -N -L 9119:127.0.0.1:9119 <usuario-ssh>@<ip-do-host>
```

Mantenha esse comando em execução e abra `http://127.0.0.1:9119` no navegador
local. O encaminhamento conecta a porta local `9119` à porta `9119` em
`127.0.0.1` no host remoto. Substitua `priv.key`, `<usuario-ssh>` e
`<ip-do-host>` pelos valores do seu acesso SSH.

## Editar dados persistidos pelo host

Com o uso, `/home/hermes/data` passa a conter arquivos de configuração,
autenticação, estado, logs, bancos de dados, plugins e outros dados do Hermes.
Como esse diretório é montado no container em `/opt/data`, os arquivos
persistidos podem ser acessados e editados diretamente no host. Isso permite,
por exemplo, ajustar `/home/hermes/data/config.yaml` ou adicionar arquivos
necessários ao Hermes.

Faça alterações com o serviço parado sempre que possível, especialmente em
bancos de dados e arquivos de estado que o Hermes possa estar usando. Preserve
as permissões e a estrutura esperadas pelo container, e mantenha cópias de
segurança antes de mudanças importantes. Arquivos criados no host dentro dessa
árvore ficam disponíveis pelo caminho correspondente em `/opt/data` no
container.

## Autenticação do Codex e do Antigravity CLI

O processo do gateway usa `HOME=/opt/data`; assim, os caminhos padrão das
credenciais são `/opt/data/.codex` para o Codex e `/opt/data/.gemini` para o
Antigravity CLI (`agy`). Não é necessário criar links simbólicos nem alterar o
`HOME` do gateway para essas ferramentas.

O shell auxiliar aberto pela ferramenta de terminal usou incorretamente
`HOME=/opt/data/home`. Por isso, o `agy` solicitou login nesse shell, embora
tenha respondido quando o `HOME` era `/opt/data`; o Codex também reconheceu a
conta nesse caminho. As credenciais criadas no local incorreto foram movidas
para `/opt/data`, e os testes seguintes confirmaram a autenticação:

```text
Codex: CODEX_AUTH_OK
AGY:   AGY_AUTH_OK
```

Em um teste anterior, o Codex respondeu “Olá! Como posso ajudar com o projeto?”
e o AGY retornou `HELLO_AGY`. Até que o shell auxiliar herde o `HOME` correto,
defina `HOME=/opt/data` ao executar testes nele. Essa diferença de ambiente do
shell não altera a configuração do gateway.

## Mensageria

Foi testada uma integração com XMPP para comunicação remota. Apesar do tempo
investido na configuração do Cloudflare One para permitir o acesso remoto, o
Hermes rejeitava as mensagens recebidas pelo XMPP. A hipótese é uma limitação
ou incompatibilidade do plugin com XMPP; a causa não foi confirmada.

A integração adotada é um bot do Telegram. Sua configuração fica persistida em
`/home/hermes/data` e está funcionando corretamente. O acesso ao bot foi
restringido a um único usuário.

## Uso do OpenCode e dos modelos

Para trabalhar com o OpenCode, a preferência é acessá-lo dentro de uma sessão
`tmux`. A interface interativa permite acompanhar o tamanho do contexto e
outros detalhes, alternar o modelo em tempo real e compactar o contexto quando
necessário.

O modo não interativo também pode ser usado em algumas tarefas, desde que haja
confiança total nas atividades executadas pelo modelo.

Como fonte prioritária de modelos no Hermes, foi configurado o NVIDIA NIM pela
interface web, usando uma chave de API. Os harnesses próprios também permitem
usar diretamente, no código, modelos disponibilizados por planos como Claude
Pro, ChatGPT Plus e OpenCode Go. Nesse fluxo, o Nemotron usado no Hermes atua
como orientador das atividades; os modelos acessados pelos harnesses são os
responsáveis pela execução do código.

O Code Mode permite ao modelo orquestrar várias chamadas a ferramentas por
meio de código JavaScript, por exemplo percorrendo resultados e filtrando ou
agregando respostas. Não é necessário para tarefas comuns como editar arquivos
ou executar testes; consulte a
[documentação de chamadas programáticas a ferramentas](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling/).
O provider `openai-codex` do Hermes estava habilitado e autenticado. Um teste
com `gpt-6-luna` respondeu `HERMES_CODEX_PROVIDER_OK` sem alterar o modelo
global do Hermes. Para validar a criação de arquivo com o Codex e Code Mode,
foi usada uma instalação oficial isolada em scratch, pois o host auxiliar
compatível não está instalado na instalação global do container. O executável
global não foi substituído.

Como solução provisória, o Hermes baixou e usou o executável standalone em um
diretório scratch. **Pendência futura:** persistir a correção diretamente no
container, instalando/configurando o host auxiliar compatível com Code Mode
para o Codex global. Até lá, usar a solução local via Hermes e executável
standalone baixado.
