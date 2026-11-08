# Simulador de Memória Virtual

**Sistemas Operativos · Trabalho Prático 2 · 2026/2027**

Este projeto estende o simulador de Sistemas Operativos fornecido na unidade curricular, acrescentando quatro algoritmos de substituição de páginas. A aplicação envia sequências de acessos à memória virtual e o simulador gere as frames físicas, as faltas de página e as operações de *swap*.

**Autor(es):** Guilherme Quartin Coelho a22309752, Afonso Quartau a22304676


## Algoritmos

O simulador permite selecionar cinco algoritmos através do argumento `--vm`:

| Algoritmo | Critério de substituição |
| --- | --- |
| **FIFO** | Substitui a página carregada há mais tempo; implementação fornecida como ponto de partida. |
| **RANDOM** | Seleciona aleatoriamente uma das frames candidatas. |
| **LRU** | Seleciona a página com o instante de último acesso (`last_accessed`) mais antigo. |
| **NRU** | Utiliza o bit de utilização e um histórico de referências de 8 bits para favorecer páginas menos recentemente utilizadas. |
| **CLOCK** | Percorre circularmente os candidatos e concede uma segunda oportunidade às páginas cujo `use_bit` está ativo. |

O escalonamento utilizado nos testes é o **FIFO**, fornecido no código-base. Os algoritmos de substituição de páginas são independentes do escalonamento do CPU.

## Estrutura do projeto

| Ficheiro/pasta | Função |
| --- | --- |
| `ossim.c` | Simulador, parâmetros de execução e processamento dos pedidos. |
| `application.c` | Aplicação que envia ao simulador os acessos definidos num cenário CSV. |
| `virtmem.c` | Gestão de páginas, frames, *swap* e algoritmos de substituição. |
| `virtmem.h`, `virtmem_types.h` | Declarações e estruturas de dados da memória virtual. |
| `scheduler.c` | Escalonamento dos processos. |
| `run.sh` | Execução do simulador e das aplicações com registo em ficheiros `.log`. |
| `cenarios/` | Cenários CSV utilizados nos testes. |
| `FORMATO_CSV.md` | Documentação do formato dos cenários. |
| `enunciado.md` | Enunciado original do trabalho. |

## Compilação

O projeto utiliza **C11** e **CMake 3.25 ou superior**. Pode ser compilado num ambiente Linux, incluindo GitHub Codespaces.

A partir da pasta principal do repositório:

```bash
cmake -S . -B build
cmake --build build
```

São gerados os executáveis `build/ossim` e `build/application`.

## Como executar

Com o script `run.sh`, substitui-se `ALGORITMO` por `FIFO`, `RANDOM`, `LRU`, `NRU` ou `CLOCK`:

```bash
sh run.sh --vpages 12 --pframes 4 --threshold 1 --vm ALGORITMO --scen cenarios/teste_comparativo.csv
```

Significado dos principais parâmetros:

| Parâmetro | Significado |
| --- | --- |
| `--vpages 12` | Número de páginas virtuais por processo. |
| `--pframes 4` | Número de frames físicas disponíveis na RAM simulada. |
| `--threshold 1` | Número mínimo de frames livres que o simulador procura manter antes de tratar o pedido seguinte. |
| `--vm ALGORITMO` | Algoritmo de substituição escolhido. |
| `--scen ficheiro.csv` | Cenário de acessos da aplicação. |

O script cria ficheiros `ossim-*.log` e `basic-*.log`. Se a execução permanecer à espera no terminal depois de a aplicação terminar, pode encerrar-se com **Ctrl+C** e consultar o registo mais recente:

```bash
tail -n 20 "$(ls -t ossim-*.log | head -1)"
```

Para observar apenas as páginas escolhidas para substituição:

```bash
grep "EVICTION:" "$(ls -t ossim-*.log | head -1)"
```

As mensagens de diagnóstico identificam o algoritmo, o processo (`pid`), a página virtual expulsa (`vpn`) e a frame libertada (`pfn`).

## Cenários de teste

### `cenarios/teste_fifo_lru.csv`

Utiliza a sequência de acessos `1 → 2 → 3 → 1 → 4 → 5` para comparar decisões de substituição. Com quatro frames e `--threshold 1`, a primeira página expulsa nos testes foi:

| Algoritmo | Página substituída |
| --- | ---: |
| FIFO | 1 |
| LRU | 2 |
| NRU | 2 |
| CLOCK | 1 |
| RANDOM | 1 e 2, em duas execuções distintas |

O FIFO escolheu a página carregada há mais tempo. O LRU escolheu a página cujo último acesso era mais antigo. Os restantes resultados mostram o comportamento das implementações utilizadas neste cenário.

Para reproduzir o teste com um algoritmo:

```bash
sh run.sh --vpages 30 --pframes 4 --threshold 1 --vm LRU --scen cenarios/teste_fifo_lru.csv
```

### `cenarios/teste_comparativo.csv`

Contém **8 bursts e 32 acessos**, incluindo leituras, escritas e páginas repetidas. Foi executado com uma aplicação, o escalonador FIFO, `--vpages 12`, `--pframes 4` e `--threshold 1`.

Os acessos são indicados no CSV entre parênteses retos: `1` representa uma leitura da página 1 e `-2` uma escrita na página 2. O formato completo encontra-se em `FORMATO_CSV.md`.

## Resultados experimentais

As estatísticas seguintes foram obtidas nas execuções realizadas para o cenário `teste_comparativo.csv`:

| Algoritmo | Acessos | Page faults | Swap-in | Swap-out | Taxa de faults |
| --- | ---: | ---: | ---: | ---: | ---: |
| FIFO | 32 | 29 | 23 | 9 | 90,6% |
| RANDOM* | 32 | **22** | **16** | **8** | **68,75%** |
| LRU | 32 | 29 | 23 | 9 | 90,6% |
| NRU | 32 | 27 | 21 | 9 | 84,4% |
| CLOCK | 32 | 29 | 23 | 9 | 90,6% |

*RANDOM: média de três execuções; os restantes algoritmos correspondem a uma execução cada.*

As três execuções do RANDOM produziram, respetivamente, **22, 21 e 23 faltas de página**, evidenciando a variação dos resultados entre execuções.

### Interpretação

Neste cenário, **RANDOM apresentou o menor número médio de faltas de página** e de operações de `swap-in`. Isso não permite concluir que RANDOM seja o melhor algoritmo em termos gerais: o desempenho depende da sequência de acessos e, neste caso, das escolhas pseudoaleatórias. FIFO, LRU e CLOCK registaram totais iguais neste cenário, apesar de poderem selecionar vítimas diferentes, como observado no teste específico.

Estes resultados documentam as execuções efetuadas; para uma avaliação mais ampla, devem ser considerados cenários adicionais com outros padrões de localidade e utilização da memória.

## Preparação da entrega

O projeto inclui um alvo CMake para gerar o arquivo de entrega:

```bash
cmake --build build --target archive
```

**Atenção à pasta dos cenários:** no `CMakeLists.txt` original, o alvo `archive` procura ficheiros em `scenarios/`, mas os ficheiros deste projeto estão em `cenarios/`. Antes de gerar o ZIP, é necessário alterar essa referência de `scenarios/*` para `cenarios/*` no `CMakeLists.txt`.

Depois de gerar `build/entrega.zip`, confirmar que os cenários foram incluídos:

```bash
unzip -l build/entrega.zip
```

A entrega no Moodle é composta pelo ZIP com código e cenários e pelo relatório. O código deve estar no GitHub e os docentes devem ser convidados para o repositório, conforme indicado em `enunciado.md`.

## Documentação adicional

Consultar `enunciado.md` para os requisitos formais do trabalho e `FORMATO_CSV.md` para criar ou adaptar cenários de execução.
