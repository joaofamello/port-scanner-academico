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
| `-p`, `--ports`   | Portas a varrer: `80`, `20-25` ou `22,80,8000-8010`    | 34 portas comuns |
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

## Testes

```bash
go test ./...
go vet ./...
```

## Aviso

Varredura de portas em máquinas de terceiros sem autorização expressa pode ser
ilegal. Utilize a ferramenta apenas em equipamentos próprios, em laboratórios
controlados ou em hosts destinados a esse fim, como `scanme.nmap.org`, mantido
pelo projeto Nmap para testes.
