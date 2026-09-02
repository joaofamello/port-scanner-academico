# PScan

Port scanner TCP desenvolvido em Go para a disciplina de Redes de Computadores.

A ferramenta realiza varredura do tipo _TCP connect_: para cada porta, tenta abrir
uma conexão completa e classifica o resultado como aberta, fechada ou filtrada.
As conexões são distribuídas entre goroutines para acelerar a varredura.

## Requisitos

- Go 1.26 ou superior
- Docker com Compose

## Laboratório Docker local

O laboratório fornece serviços locais e controlados para validar a varredura TCP
do scanner. Todos os containers compartilham a rede `port-scanner-lab`, e as
portas são publicadas no host local para que o alvo da varredura seja
`127.0.0.1`.

Para iniciar todo o laboratório em segundo plano, execute um único comando na
raiz do projeto:

```bash
docker-compose up -d
```

| Serviço    | Imagem utilizada     | Porta exposta no host |
| ---------- | -------------------- | --------------------- |
| HTTP       | `nginx:alpine`       | `80`                  |
| SSH        | `panubo/sshd:latest` | `22`                  |
| PostgreSQL | `postgres:alpine`    | `5432`                |
| Redis      | `redis:alpine`       | `6379`                |

Por exemplo, para verificar as portas abertas pelo laboratório:

```bash
go run ./cmd/pscan -target 127.0.0.1 -ports 80,22,5432,6379
```

Para finalizar o laboratório e remover os containers:

```bash
docker-compose down
```

## Compilação

```bash
go build ./cmd/pscan
```

O comando gera o executável `pscan` (`pscan.exe` no Windows) na raiz do projeto.

Durante o desenvolvimento também é possível executar sem gerar o binário:

```bash
go run ./cmd/pscan -target 127.0.0.1
```

## Uso

```bash
./pscan -target <alvo> [opções]     # Linux e macOS
.\pscan.exe -target <alvo> [opções] # Windows
```

O alvo aceita endereço IP, domínio ou URL completa. Domínios são resolvidos para
IPv4 antes da varredura.

### Opções

| Opção             | Descrição                                              | Padrão           |
| ----------------- | ------------------------------------------------------ | ---------------- |
| `-t`, `--target`  | URL, domínio ou IP que será varrido (obrigatório)      | —                |
| `-p`, `--ports`   | Portas a varrer: `80`, `20-25` ou `22,80,8000-8010`    | 32 portas comuns |
| `-w`, `--workers` | Quantidade de conexões simultâneas                     | `100`            |
| `-T`, `--timeout` | Tempo máximo de espera por porta (`500ms`, `2s`, `1m`) | `2s`             |
| `-b`, `--banner`  | Tenta ler o banner das portas abertas                  | desativado       |
| `-h`, `--help`    | Exibe a ajuda de utilização                            | —                |

### Exemplos

```bash
# Ajuda
./pscan -h

# Portas comuns do host local
./pscan -target 127.0.0.1

# Porta única
./pscan -target 127.0.0.1 -ports 22

# Intervalo de portas
./pscan -target 192.168.0.1 -ports 20-25

# Lista combinada com intervalo
./pscan -target 192.168.0.1 -ports 22,80,8000-8010

# Domínio, com mais workers e timeout reduzido
./pscan -target scanme.nmap.org -ports 1-1024 -workers 200 -timeout 500ms

# URL completa, com captura de banner
./pscan -target https://scanme.nmap.org/teste -ports 22,80 -banner
```

### Exemplo de saída

```
 ____   ____
|  _ \ / ___|   ___    __ _  _ __
| |_) |\___ \  / __|  / _` || '_ \
|  __/  ___) || (__  | (_| || | | |
|_|    |____/  \___|  \__,_||_| |_|

PScan v1.0 - varredura TCP connect sobre IPv4

Alvo:    127.0.0.1
IP:      127.0.0.1
Portas:  22 | Workers: 100 | Timeout: 300ms | Banner: desativado

PORTA   ESTADO   SERVIÇO
135     open     msrpc
445     open     smb

22 portas verificadas: 2 abertas, 20 fechadas, 0 filtradas
Tempo total: 3ms
```

Apenas as portas abertas aparecem na tabela; as demais entram na contagem do resumo.
A tabela é ordenada pelo número da porta, e banners longos são limitados a 120
caracteres para preservar a legibilidade. A coluna `BANNER` só é exibida quando
a opção `-banner` está ativa.

### Códigos de saída

| Código | Situação                                                   |
| ------ | ---------------------------------------------------------- |
| `0`    | Varredura concluída, ou ajuda exibida                      |
| `1`    | Parâmetro inválido, alvo não resolvido ou erro de execução |

## Como funciona

### Conexão TCP

Para cada porta o PScan realiza um _TCP connect_: pede ao sistema operacional
uma conexão completa, com o handshake de três vias (`SYN`, `SYN-ACK` e `ACK`).
Não são montados pacotes crus, toda a conversa é feita pela chamada `connect()`
do sistema, exposta em Go pela função `net.DialTimeout`. Por isso a ferramenta
não exige privilégio de administrador.

O alvo é resolvido para um único endereço IPv4 antes da varredura começar, e
todas as portas são testadas contra esse mesmo IP. Quando o handshake é
concluído, a conexão é encerrada em seguida. Com a opção `-banner` ativa, antes
de fechar a conexão o PScan ainda tenta ler até 1024 bytes enviados pelo
serviço.

### Estados das portas

O estado é definido pelo retorno do handshake, sem leitura direta dos pacotes:

| Estado     | Como é identificado                        | O que costuma indicar                              |
| ---------- | ------------------------------------------ | -------------------------------------------------- |
| `open`     | O handshake foi concluído                  | Existe um serviço aceitando conexões na porta       |
| `closed`   | O alvo respondeu recusando a conexão (RST) | A porta foi alcançada, mas nenhum serviço a escuta  |
| `filtered` | Nenhuma resposta chegou dentro do timeout  | Firewall descartando os pacotes, ou alvo muito lento |

Como o estado `filtered` é decidido apenas pela ausência de resposta, ele não
diferencia um firewall de uma rede congestionada. Uma porta aberta em um alvo
lento demais pode ser classificada como filtrada.

### Papel do timeout

O timeout define quanto tempo o PScan espera pela resposta de cada porta antes
de desistir e classificá-la como filtrada. O valor é fixo, o mesmo para todas as
portas, e é ajustado com `-T`. O mesmo valor também é usado como prazo de
leitura do banner.

A escolha do valor equilibra dois riscos. Um timeout menor que a latência do
alvo gera falso negativo, com portas abertas aparecendo como filtradas. Um
timeout maior deixa a varredura mais lenta, porque cada porta sem resposta
segura um worker até o prazo terminar.

Uma referência prática é usar um valor com folga sobre a latência até o alvo. Em
`scanme.nmap.org`, com latência de aproximadamente 210 ms, timeouts abaixo de
`250ms` deixaram de identificar as portas 22 e 80. No host local e no
laboratório Docker, valores como `500ms` são suficientes.

### Arquitetura baseada em workers

A varredura é concorrente. O PScan cria a quantidade de goroutines definida em
`-w`, os workers, e distribui as portas entre elas por um canal de trabalho.
Cada worker retira uma porta da fila, executa a tentativa de conexão e devolve o
resultado por um segundo canal, junto com o índice de origem.

O resultado é gravado na posição correspondente da lista, então a ordem das
portas informadas é preservada, independentemente de qual worker terminou
primeiro. Um `sync.WaitGroup` aguarda todos os workers encerrarem antes de
fechar o canal de resultados. Pedir mais workers do que portas não tem efeito,
pois a quantidade é reduzida ao número de portas a varrer.

Como cada worker fica bloqueado esperando a resposta de uma porta, no pior caso,
em que nenhuma porta responde, o tempo total se aproxima de:

`(portas / workers) × timeout`

O ganho, porém, não é ilimitado. A partir de certo ponto o gargalo passa a ser a
rede e a quantidade de sockets que o sistema operacional consegue manter
abertos, e aumentar os workers deixa de reduzir o tempo. As medições estão em
[docs/relatorio.md](docs/relatorio.md).

## Testes

```bash
go test ./...
go vet ./...
```

## Comparação com o Nmap

O documento [docs/comparacao-nmap.md](docs/comparacao-nmap.md) registra uma
comparação prática entre o PScan e o Nmap: portas encontradas, tempo de
varredura, identificação de serviço e o efeito dos parâmetros de workers e
timeout.

## Integrantes

| Integrante      | GitHub                                           | Contribuições                                                                                                                                                                                                |
| --------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Jurandir Neto   | [@Jurandirtvaz](https://github.com/Jurandirtvaz) | Resolução do alvo por IP, domínio ou URL, interpretação da seleção de portas, interface de linha de comando, captura opcional de banner, testes da CLI e do pool, documentação de uso e comparação com o Nmap |
| José Gustavo    | [@Gustavo7a](https://github.com/Gustavo7a)       | Contratos base dos tipos de resultado, captura de banner com a bateria de testes, relatório de análise de desempenho dos workers                                                                              |
| João Francisco  | [@joaofamello](https://github.com/joaofamello)   | Estrutura inicial do projeto e dos packages, conexão TCP com timeout, pool concorrente de varredura, ordenação dos resultados e limite de banner, laboratório Docker                                          |
| Cauã de Souza   | [@cauaofsouza](https://github.com/cauaofsouza)   | Análise do tráfego da varredura no Wireshark, com as capturas de porta aberta, fechada e filtrada relacionadas aos resultados apresentados pelo scanner                                                       |

## Aviso

O PScan foi desenvolvido para fins de estudo, na disciplina de Redes de
Computadores. Sua utilização é permitida apenas em `127.0.0.1`, em equipamentos
da própria rede local do usuário e em `scanme.nmap.org`, host mantido pelo
projeto Nmap para testes de varredura. O laboratório Docker descrito acima
fornece um ambiente pronto para esses testes.

Varredura de portas em máquinas de terceiros sem autorização expressa pode ser
ilegal. Os integrantes do projeto não se responsabilizam pelo uso inadequado da
ferramenta, nem por qualquer utilização com finalidade maliciosa.
