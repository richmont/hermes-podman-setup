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
EnvironmentFile=/etc/hermes/hermes-mariadb.conf

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

O `EnvironmentFile` carrega as variáveis do MariaDB no ambiente do processo
`podman-compose`. Os containers não recebem esse ambiente diretamente: o
serviço `mariadb` lê o mesmo arquivo pelo `env_file` do `compose.yaml`. O
`EnvironmentFile` só tem efeito se o `compose.yaml` usar interpolação, como
`${MARIADB_PASSWORD}`, e não é carregado quando o `podman-compose` é executado
manualmente com `sudo -u hermes`. O systemd lê o arquivo como `root` antes de
trocar para o usuário `hermes`, então a permissão `600` não impede a leitura.

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
├── compose.yaml
└── hermes-mariadb.conf
```

O `Containerfile` parte da imagem `docker.io/nousresearch/hermes-agent:stable`.
Como `root`, instala `curl`, `ca-certificates`, `tmux`, `git`, `procps`, `sudo`,
`tree`, `plocate` e `mariadb-client` (cliente usado para acessar o MariaDB),
além das ferramentas Antigravity CLI (`agy`), Codex CLI
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
        curl ca-certificates tmux git procps sudo tree plocate mariadb-client \
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
├── mariadb/
└── workspace/
```

Os diretórios `data` e `workspace` são montados no container em `/opt/data` e
`/workspace`, respectivamente. O compose também monta
`/home/hermes/data/.gemini/antigravity-cli` em
`/home/hermes/.gemini/antigravity-cli`. Como o compose define
`HOME=/opt/data`, o caminho padrão para os dados do Antigravity CLI é
`/opt/data/.gemini`; a
diferença entre esse caminho e o destino do mount está detalhada na seção de
autenticação abaixo. O diretório `/workspace` é usado como diretório de
trabalho.

O conteúdo do `compose.yaml` é:

```yaml
services:
  mariadb:
    image: docker.io/library/mariadb:11.8
    container_name: hermes-mariadb
    env_file:
      - /etc/hermes/hermes-mariadb.conf
    volumes:
      - /home/hermes/mariadb:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  hermes:
    image: localhost/hermes-opencode:2
    container_name: hermes-agent
    depends_on:
      - mariadb

    userns_mode: "keep-id:uid=10000,gid=10000"
    user: "0:0"
    extra_hosts:
      - "host.example.invalid:host-gateway" # nome real do host omitido
    volumes:
      - /home/hermes/data:/opt/data
      - /home/hermes/workspace:/workspace
      - /home/hermes/data/.gemini/antigravity-cli:/home/hermes/.gemini/antigravity-cli
    environment:
      HOME: /opt/data
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
na porta `9119` somente no endereço local do host e inicia o gateway. A
variável `HOME: /opt/data` vale para todo o container, inclusive para comandos
abertos com `podman exec`; sem ela, o `agy` e o `codex` não encontram as
credenciais persistidas em `/opt/data` (veja a seção de autenticação). O
`GH_TOKEN` foi censurado neste registro. Para o GitHub funcionar corretamente,
o token também foi adicionado diretamente ao arquivo `config.yaml` do Hermes;
esse arquivo fica na configuração persistida do usuário/container e seu
conteúdo não foi informado aqui.

## Banco de dados MariaDB

O serviço `mariadb` (container `hermes-mariadb`) usa a imagem oficial
`docker.io/library/mariadb:11.8` (série LTS) e roda no mesmo Compose do Hermes.
O `podman-compose` coloca os dois serviços na rede padrão do projeto, onde o
nome do serviço é resolvido por DNS; assim, o Hermes acessa o banco em
`mariadb:3306`. A porta não é publicada no host nem em interfaces externas.

### Arquivo de variáveis

As credenciais ficam em `/etc/hermes/hermes-mariadb.conf`, carregado pelo
`env_file` do serviço. O arquivo é lido pelo Podman como lista de variáveis,
não por um shell: cada linha deve estar no formato `NOME=valor`, **sem o
prefixo `export`** e sem aspas (aspas seriam incluídas no valor):

```text
MARIADB_ROOT_PASSWORD=<senha-root>
MARIADB_DATABASE=hermes
MARIADB_USER=hermes
MARIADB_PASSWORD=<senha-hermes>
```

Os valores reais foram omitidos neste registro. Para remover o `export` de um
arquivo já criado e restringir o acesso a ele:

```bash
sudo sed -i 's/^export //' /etc/hermes/hermes-mariadb.conf
sudo chown hermes:hermes /etc/hermes/hermes-mariadb.conf
sudo chmod 600 /etc/hermes/hermes-mariadb.conf
```

Essas variáveis só são aplicadas na **primeira inicialização**, quando
`/var/lib/mysql` está vazio: o entrypoint cria o banco `hermes`, o usuário
`hermes` com acesso a ele e define a senha do `root`. Alterar o arquivo depois
não muda as senhas já gravadas; para isso, use `ALTER USER` no banco.

### Dados persistidos

Os dados ficam em `/home/hermes/mariadb`, montado em `/var/lib/mysql`. Crie o
diretório como usuário `hermes` antes do primeiro `up`:

```bash
sudo -u hermes mkdir -p /home/hermes/mariadb
```

Dentro do container, o MariaDB roda com o UID `999`, que no Podman rootless é
mapeado para um UID subordinado do usuário `hermes`. Por isso, no host, os
arquivos desse diretório aparecem com um dono numérico diferente; para
inspecioná-los, use `sudo -u hermes podman unshare ls -l /home/hermes/mariadb`.

### Acesso pelo Hermes

O container `hermes-agent` **não recebe as credenciais do banco por variáveis
de ambiente**: o `env_file` do `compose.yaml` pertence apenas ao serviço
`mariadb`, e o `EnvironmentFile` do systemd afeta somente o processo
`podman-compose`. Assim, a senha do `root` não fica exposta no container do
Hermes.

O acesso usa o usuário `hermes` do banco por meio do arquivo de opções do
cliente `/home/hermes/data/.my.cnf`, visto no container como
`/opt/data/.my.cnf`. Como o container usa `HOME=/opt/data`, o cliente `mariadb`
lê `~/.my.cnf` automaticamente, e o Hermes se conecta sem informar senha na
linha de comando ou na conversa com o modelo.

#### Conteúdo

```ini
[client]
host=mariadb
port=3306
user=hermes
password="<senha-hermes>"

[mysql]
database=hermes
```

- `[client]` é lido por todos os clientes (`mariadb`, `mariadb-dump`,
  `mariadb-admin` etc.); `[mysql]` é lido apenas pelo cliente interativo
  `mariadb`. O `database` fica em `[mysql]` porque outros clientes, como o
  `mariadb-dump`, recusam essa opção com o erro `unknown variable`;
- `host=mariadb` é o nome do serviço no Compose, resolvido pela rede interna;
- `user`, `password` e `database` devem ser iguais a `MARIADB_USER`,
  `MARIADB_PASSWORD` e `MARIADB_DATABASE` do `hermes-mariadb.conf`;
- as aspas em `password` permitem senhas com `#`, `;` ou espaços, que seriam
  interpretados como comentário ou separador sem elas. O valor dentro das aspas
  é a senha literal.

#### Criação

A configuração é feita **uma única vez**. Como o arquivo fica em
`/home/hermes/data`, ele persiste ao recriar o container, ao reconstruir a
imagem e ao reiniciar o host. Crie-o como usuário `hermes`, com permissão
`600` desde o início, para que a senha nunca fique legível por outros usuários:

```bash
sudo -u hermes install -m 600 /dev/null /home/hermes/data/.my.cnf
sudo -u hermes nano /home/hermes/data/.my.cnf
```

O dono `hermes` no host corresponde ao UID `10000` no container
(`userns_mode: keep-id`). O cliente `mariadb` ignora arquivos de opções com
permissão de escrita para todos; mantenha o modo `600`.

#### Uso

Dentro do container, o cliente usa o arquivo sem argumentos extras:

```bash
mariadb -e "SHOW TABLES;"
mariadb-dump hermes > /workspace/hermes.sql
```

Se algum shell rodar com outro `HOME` (veja a seção de autenticação), defina
`HOME=/opt/data` ou passe `--defaults-extra-file=/opt/data/.my.cnf`. Bibliotecas
de aplicações (Python, Node etc.) não leem o `.my.cnf` por padrão; para elas,
use os mesmos dados na string de conexão, por exemplo
`mysql://hermes:<senha-hermes>@mariadb:3306/hermes`.

#### Cuidados

- O agente tem acesso ao próprio container e consegue ler o arquivo; o
  `.my.cnf` evita repetir a senha, mas não a esconde do Hermes. Por isso o
  acesso usa o usuário `hermes`, restrito ao banco `hermes`, e não o `root`.
- Ao trocar a senha do usuário com `ALTER USER`, atualize também o `.my.cnf`.
  O `MARIADB_PASSWORD` do `hermes-mariadb.conf` só é usado na primeira
  inicialização, mas mantê-lo igual evita confusão em uma restauração.

### Aplicar e validar

Após reconstruir a imagem com o `mariadb-client` (seção “Reconstruir a imagem
e recriar o container”), suba os serviços e confira o banco e o acesso pelo
Hermes:

```bash
sudo -u hermes podman-compose --in-pod=false -f /etc/hermes/compose.yaml up -d --force-recreate
sudo -u hermes podman exec hermes-mariadb healthcheck.sh --connect --innodb_initialized && echo MARIADB_OK
sudo -u hermes podman exec hermes-agent mariadb \
    -e "SELECT CURRENT_USER(), DATABASE(), VERSION();"
```

Para um console administrativo como `root`, abra o cliente dentro do próprio
container do banco (a senha é solicitada interativamente):

```bash
sudo -u hermes podman exec -it hermes-mariadb mariadb -u root -p
```

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

**Pendência futura:** implementar um backup reproduzível dos arquivos
persistidos do container e do ambiente no host. Preservar, no mínimo:

- `/home/hermes/workspace/` — projetos, inclusive alterações ainda não
  commitadas;
- `/home/hermes/config/` e `/home/hermes/container/` — configurações prontas
  mantidas nessas pastas;
- `/home/hermes/data/` — configuração e estado persistido do Hermes;
- `/home/hermes/mariadb/` — dados do MariaDB; preferir um dump lógico
  consistente (`mariadb-dump --single-transaction` executado no container
  `hermes-mariadb`) em vez de copiar os arquivos com o servidor em execução;
- `/etc/hermes/Containerfile` e `/etc/hermes/compose.yaml` — arquivos usados
  para construir e iniciar o container.

Oferecer dois modos explícitos. No **backup seguro**, preservar os caminhos
acima, mas ignorar as credenciais `/home/hermes/data/.codex/` e
`/home/hermes/data/.gemini/` (equivalentes a `/opt/data/.codex/` e
`/opt/data/.gemini/` dentro do container), além de excluir ou sanitizar
`/home/hermes/data/config.yaml`, que pode conter o `GH_TOKEN` e outras chaves,
e as senhas do banco em `/etc/hermes/hermes-mariadb.conf` e
`/home/hermes/data/.my.cnf`.
Também ignorar arquivos temporários descartáveis, como
`/home/hermes/data/cache/scratch/`. O processo deve registrar que as
credenciais precisarão ser configuradas novamente após a restauração.

No **backup inseguro**, incluir também esses diretórios de credenciais e o
`config.yaml` original para restaurar a autenticação e as configurações sem
reconfiguração; avisar claramente que o arquivo contém segredos e protegê-lo
com acesso restrito e armazenamento seguro. Em ambos os modos, definir como
tratar outras credenciais que venham a ser adicionadas, restaurar permissões e
validar a recuperação do ambiente.

## Autenticação do Codex e do Antigravity CLI

O `compose.yaml` define `HOME: /opt/data` no ambiente do container. Assim, os
caminhos padrão das credenciais são `/opt/data/.codex` para o Codex e
`/opt/data/.gemini` para o Antigravity CLI (`agy`), ambos persistidos em
`/home/hermes/data` no host. Sem essa variável, as ferramentas procuram as
credenciais em outro diretório e pedem login novamente. Não é necessário criar
links simbólicos.

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
e o AGY retornou `HELLO_AGY`. Se o shell auxiliar continuar usando
`HOME=/opt/data/home` mesmo com a variável definida no compose, defina
`HOME=/opt/data` ao executar testes nele. Para conferir o valor no container:

```bash
sudo -u hermes podman exec hermes-agent sh -c 'echo $HOME'
```

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
