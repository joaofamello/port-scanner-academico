# Comparação entre o PScan e o Nmap

Este documento apresenta uma comparação prática entre o PScan, desenvolvido na disciplina de Redes de Computadores, e o Nmap, uma ferramenta utilizada para varredura de portas.

O objetivo é comparar os resultados das duas ferramentas, verificando se elas encontram as mesmas portas abertas, além de analisar o tempo de execução e as principais diferenças encontradas durante os testes.


## Ambiente de teste

| Item                 | Valor                                       |
| -------------------- | ------------------------------------------- |
| Sistema operacional  | Windows 11 Home (10.0.26200)                |
| Go                   | 1.26.7 windows/amd64                        |
| Nmap                 | 7.80                                        |
| Alvo local           | `127.0.0.1` com o laboratório Docker ativo  |
| Alvo externo         | `scanme.nmap.org` (`45.33.32.156`)          |

No laboratório com Docker, executado com `docker-compose up -d`, foram disponibilizados quatro serviços: nginx na porta `80`, OpenSSH na `2222`, PostgreSQL na `5432` e Redis na `6379`. Além dessas, o Windows também possuía as portas `135` e `445` abertas.

Para os testes externos utilizamos o `scanme.nmap.org`, que é disponibilizado pelo próprio projeto Nmap para testes de varredura.

Em todos os testes, o Nmap foi executado com `-sT`, utilizando a mesma técnica de conexão TCP implementada pelo PScan. Também utilizamos `-Pn -n` para desativar a descoberta de host e a resolução de DNS. Dessa forma, conseguimos fazer uma comparação mais justa entre as duas ferramentas, já que o PScan não realiza essas etapas.

## Teste 1 — portas 1 a 1024 no host local

Comandos:

```powershell
.\pscan.exe -t 127.0.0.1 -p 1-1024 -w 200 -T 1s
nmap -sT -Pn -n -p1-1024 127.0.0.1
```

| Ferramenta                        | Tempo total | Portas abertas encontradas |
| --------------------------------- | ----------- | -------------------------- |
| PScan                             | 0,04 s      | 80, 135, 445               |
| Nmap (`-sT` padrão)               | 46,98 s     | 80, 135, 445               |
| Nmap (`-T4 --max-retries 0`)      | 22,00 s     | 80, 135, 445               |

As duas ferramentas encontraram exatamente o mesmo conjunto de portas abertas. Porém, tivemos uma diferença considerável no tempo de execução. Ao utilizar `--max-retries 0` no Nmap, o tempo caiu praticamente pela metade, sem alterar o resultado da varredura.

Também tivemos uma diferença na classificação das outras 1021 portas. O PScan classificou essas portas como fechadas, enquanto o Nmap classificou como filtradas.

Nesse caso, o resultado do PScan foi mais adequado, pois no Windows o Nmap depende do Npcap para capturar o tráfego da rede e pode apresentar limitações na interface de `loopback`. Já o PScan utiliza diretamente o `connect()`, recebendo a informação de conexão recusada e classificando a porta como fechada.
## Teste 2 — portas selecionadas do laboratório

Comandos:

```powershell
.\pscan.exe -t 127.0.0.1 -p 22,80,443,2222,3306,5432,6379,8080 -T 1s
nmap -sT -Pn -n -p 22,80,443,2222,3306,5432,6379,8080 127.0.0.1
```

| Ferramenta | Tempo total | Portas abertas encontradas |
| ---------- | ----------- | -------------------------- |
| PScan      | 0,002 s     | 80, 2222, 5432, 6379       |
| Nmap       | 1,33 s      | 80, 2222, 5432, 6379       |

Aqui o resultado foi o mesmo, ambos encontraram as 4 portas abertas.

Temos apenas uma ressalva, pois o Pscan ele classificou a porta `2222` como `unknown`, mas isso aconteceu porque na tabela de portas colocamos apenas 32 portas conhecidas.


## Teste 3 — portas 1 a 1024 em host externo

Comandos:

```powershell
.\pscan.exe -t scanme.nmap.org -p 1-1024 -w 200 -T 2s
nmap -sT -Pn -n -p1-1024 45.33.32.156
```

| Ferramenta | Tempo total | Portas abertas encontradas |
| ---------- | ----------- | -------------------------- |
| PScan      | 12,00 s     | 22, 80                     |
| Nmap       | 106,26 s    | 22, 80                     |

Novamente tivemos o mesmo conjunto de portas abertas, enquanto as outras 1022 portas foram classificadas pelas duas ferramentas como filtradas.

Também percebemos uma diferença no tempo de execução, pois o PScan foi bem mais rápido que o Nmap. Isso acontece porque o PScan faz apenas uma tentativa por porta e espera um tempo fixo pela resposta. Já o Nmap pode realizar novas tentativas quando não recebe resposta, além de controlar a velocidade do envio dos pacotes. Com isso, em uma rede com perda de pacotes, o Nmap pode acabar sendo mais preciso na identificação das portas.

## Teste 4 — identificação de serviço e banner

Comandos:

```powershell
.\pscan.exe -t 127.0.0.1 -p 80,2222,5432,6379 -b -T 2s
nmap -sT -Pn -n -sV -p 80,2222,5432,6379 127.0.0.1
```

| Porta  | PScan `-b`                          | Nmap `-sV`                        |
| ------ | ----------------------------------- | --------------------------------- |
| `80`   | cabeçalho HTTP com `nginx/1.31.4`   | `nginx 1.31.4`                    |
| `2222` | `SSH-2.0-OpenSSH_10.3`              | `OpenSSH 10.3 (protocol 2.0)`     |
| `5432` | `postgresql` (nome da tabela)       | `postgresql?` (sem certeza)       |
| `6379` | `redis` (nome da tabela)            | `Redis key-value store 8.10.1`    |

Nas portas HTTP e SSH, o PScan conseguiu obter praticamente as mesmas informações que o Nmap, incluindo a versão do `nginx` e do `OpenSSH`.

Já nas portas do `PostgreSQL` e `Redis`, o PScan não conseguiu identificar a versão do serviço, pois esses serviços esperam receber uma requisição do cliente antes de responder. Por isso, o PScan aguardou o tempo definido no `-T` e depois utilizou apenas o nome presente na tabela de portas.

O Nmap conseguiu identificar a versão do `Redis` porque utiliza verificações específicas para cada serviço. Já no `PostgreSQL`, nem o Nmap conseguiu identificar corretamente, apresentando apenas `postgresql?`.

## Teste 5 — efeito da quantidade de workers

No host local, com as 1024 primeiras portas, medindo a mediana de cinco
execuções:

| Workers | Tempo (mediana) |
| ------- | --------------- |
| 1       | 0,120 s         |
| 10      | 0,048 s         |
| 50      | 0,052 s         |
| 100     | 0,046 s         |
| 200     | 0,053 s         |
| 500     | 0,053 s         |

No loopback, a resposta acontece praticamente de forma instantânea, então aumentar a quantidade de workers não apresentou uma diferença significativa no tempo da varredura.

Para observar melhor o efeito do paralelismo, realizamos o teste no `scanme.nmap.org`, utilizando as primeiras 256 portas e definindo o tempo de espera em `-T 500ms`:

| Workers | Tempo   |
| ------- | ------- |
| 8       | 16,07 s |
| 32      | 4,02 s  |
| 128     | 1,02 s  |
| 256     | 0,52 s  |

O ganho foi proporcional à quantidade de `workers`. Ao quadruplicar o número de `workers`, o tempo da varredura foi reduzido aproximadamente quatro vezes.

Isso acontece porque a maioria das portas permanece aguardando até atingir o `timeout`. Dessa forma, o tempo total pode ser estimado por:

`(portas / workers) × timeout`

Com 256 `workers` para 256 portas, todas as portas são verificadas ao mesmo tempo, fazendo com que a varredura leve aproximadamente o tempo de um único timeout.

## Teste 6 — sensibilidade ao tempo limite

Ao realizar a varredura apenas nas portas 22 e 80 do `scanme.nmap.org`, observamos que a latência medida pelo Nmap foi de aproximadamente `0,21 s`.

| Timeout | Resultado                  |
| ------- | -------------------------- |
| 50 ms   | 0 abertas, 2 filtradas     |
| 150 ms  | 0 abertas, 2 filtradas     |
| 250 ms  | 2 abertas, 0 filtradas     |
| 500 ms  | 2 abertas, 0 filtradas     |
| 2 s     | 2 abertas, 0 filtradas     |

Quando utilizamos um `timeout` abaixo de `250 ms`, o PScan deixou de identificar duas portas que estavam abertas, gerando um falso negativo. Isso aconteceu porque o tempo de espera definido foi menor que a latência do alvo.

Esse é um dos principais riscos de utilizar um tempo fixo, pois é necessário escolher um valor que seja suficiente para receber a resposta do alvo. Já o Nmap mede o tempo de resposta durante a própria varredura e ajusta esse valor automaticamente.

## Resumo das diferenças

| Aspecto                      | PScan                                  | Nmap                                             |
| ---------------------------- | -------------------------------------- | ------------------------------------------------ |
| Portas abertas encontradas   | Iguais em todos os testes              | Iguais em todos os testes                        |
| Tempo de varredura           | Mais rápido em todos os cenários       | Mais lento por causa das retransmissões          |
| Técnicas de varredura        | Apenas TCP connect                     | connect, SYN, UDP, FIN, Xmas, NULL, ACK e outras |
| Ajuste de tempo              | Timeout fixo definido pelo usuário     | Adaptativo, com base no tempo de ida e volta     |
| Retransmissão de sondas      | Nenhuma                                | Sim, configurável por `--max-retries`            |
| Distinção fechada/filtrada   | Por timeout, sem leitura do pacote     | Pela resposta observada na rede                  |
| Identificação de serviço     | Tabela estática de 34 portas           | Base `nmap-services` e sondas de versão `-sV`    |
| Protocolo de rede            | IPv4 e TCP                             | IPv4, IPv6, TCP, UDP, SCTP                       |
| Descoberta de host           | Não possui                             | Vários métodos de ping                           |
| Extensibilidade              | Não possui                             | Scripts NSE                                      |
| Formatos de saída            | Texto na tela                          | Texto, XML, grepável e script kiddie             |

## Conclusão

Para o objetivo do PScan, que é identificar quais portas TCP estão abertas em um host conhecido, os resultados foram equivalentes aos do Nmap nos testes realizados. Nos ambientes testados, as duas ferramentas encontraram o mesmo conjunto de portas abertas, porém o PScan apresentou um tempo de execução menor.

Porém, isso não significa que o PScan seja melhor que o Nmap. A diferença de tempo acontece porque o PScan utiliza uma abordagem mais simples, sem retransmissão de pacotes, ajuste automático de timeout ou controle da taxa de envio. Nos testes foi possível perceber isso claramente: o PScan foi mais rápido, mas começou a apresentar falsos negativos quando o timeout ficou menor que a latência do alvo.

Também tivemos uma diferença na classificação das portas fechadas e filtradas. O PScan utiliza o retorno do connect() para definir o estado da porta, o que funciona bem quando o alvo responde. Porém, pode não diferenciar corretamente uma rede lenta de um firewall que simplesmente não responde aos pacotes.

Mesmo com essas limitações, essa abordagem mais simples também apresentou resultados positivos, como no teste realizado no loopback do Windows, onde o PScan conseguiu realizar a classificação corretamente enquanto o Nmap apresentou uma limitação de captura.

## Como reproduzir

```powershell
docker-compose up -d
go build -o pscan.exe ./cmd/pscan

# Teste 1
.\pscan.exe -t 127.0.0.1 -p 1-1024 -w 200 -T 1s
nmap -sT -Pn -n -p1-1024 127.0.0.1

# Teste 2
.\pscan.exe -t 127.0.0.1 -p 22,80,443,2222,3306,5432,6379,8080 -T 1s
nmap -sT -Pn -n -p 22,80,443,2222,3306,5432,6379,8080 127.0.0.1

# Teste 3
.\pscan.exe -t scanme.nmap.org -p 1-1024 -w 200 -T 2s
nmap -sT -Pn -n -p1-1024 45.33.32.156

# Teste 4
.\pscan.exe -t 127.0.0.1 -p 80,2222,5432,6379 -b -T 2s
nmap -sT -Pn -n -sV -p 80,2222,5432,6379 127.0.0.1

docker-compose down
```

Os tempos foram medidos com `[Diagnostics.Stopwatch]` do PowerShell e conferidos
com o tempo que cada ferramenta informa ao final da varredura. Valores medidos
em uma única máquina e uma única rede variam conforme o ambiente; o que deve se
manter em outra execução é a ordem de grandeza e o conjunto de portas abertas.
