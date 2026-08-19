# pword: uma ferramenta CLI multi-processo para contar, localizar e comparar uma palavra em ficheiros de texto

![Python](https://img.shields.io/badge/Python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![Multiprocessing](https://img.shields.io/badge/multiprocessing-blue?style=for-the-badge)

> 📖 Quick note in English: This README is also available in English. To access it, just click [here](README.md).

## Sobre o projeto

`pword` é uma ferramenta de linha de comandos que cria múltiplos **processos filhos** para procurar uma palavra em um ou mais ficheiros de texto de forma concorrente. Dependendo do número de ficheiros e de processos envolvidos, a ferramenta distribui ficheiros inteiros entre os processos filhos ou divide um único ficheiro grande em partes (chunks), para que o trabalho seja feito em paralelo em vez de sequencialmente. O programa regista o progresso periodicamente (contagens parciais, ficheiros processados/pendentes) a um intervalo configurável, e trata o sinal `SIGINT` (Ctrl+C) de forma controlada.

Foi desenvolvido no âmbito da unidade curricular de **Sistemas Operativos**, grupo SO-TI-13, por Guilherme Soares, Duarte Soares e Vitória Correia.

### Funcionalidades
- Procura/contagem de palavras em paralelo em vários ficheiros usando `multiprocessing.Process`
- Três modos de operação, selecionados com `-m`:
  - `c` (count): conta todas as ocorrências da palavra como substring de qualquer token no texto, usando um contador partilhado protegido por um `Lock`
  - `l` (locate): conta e recolhe as linhas únicas que contêm a palavra como substring, com um contador por processo filho e uma `Queue` usada para reunir as linhas encontradas
  - `i`: conta as ocorrências exatas da palavra como token completo (ou seja, `word == token`, e não apenas uma substring), com um contador por processo filho agregado através de uma `Queue`
- Distribuição automática do trabalho entre os processos filhos:
  - se houver mais processos do que ficheiros, o número de processos é reduzido para corresponder ao número de ficheiros
  - se houver mais ficheiros do que processos, os ficheiros são distribuídos alternadamente (round-robin) entre os processos
  - se for indicado um único ficheiro com vários processos, o ficheiro é dividido em partes baseadas em linhas, uma por processo
- Intervalo de registo de progresso configurável (`-i`), impresso em `stdout` ou escrito num ficheiro de log (`-d`), incluindo contagens parciais e o número de ficheiros processados/restantes
- Tratamento controlado de `SIGINT`: os processos filhos ignoram diretamente o `SIGINT` (só o processo pai reage a ele); o processo pai ativa uma flag partilhada que impede o processamento de mais ficheiros, e o programa termina antecipadamente com `sys.exit(0)` caso o sinal chegue antes da divisão dos ficheiros ou antes da criação dos processos filhos

### Utilização

```
./pword -m <c|l|i> -p <n_processos> -i <intervalo> -d <ficheiro_log|stdout> -w <palavra> ficheiro1 [ficheiro2 ...]
```

Exemplos de comandos:
1. `./pword -m c -p 2 -d testLog.log -w palavra testFiles/file.txt`
2. `./pword -m l -w ola -p 1 testFiles/file.txt`
3. `./pword -m i -p 12 -w palavra testFiles/file.txt testFiles/file.txt`
4. `./pword -p 2 -i 1 -w palavra testFiles/file.txt`
5. `./pword -m c -i 1 -d testLog.log -w palavra testFiles/file1.txt testFiles/file2.txt`

Flags:
- `-m`: modo de operação, `c`, `l` ou `i` (por padrão `c`)
- `-p`: número de processos filhos a criar (por padrão `1`)
- `-w`: palavra a procurar (obrigatório)
- `-i`: intervalo de registo do log em segundos
- `-d`: ficheiro de destino do log de progresso (por padrão `stdout` se não for indicado)

**Tempo de registo de log**
- Se `-i` não for fornecido, o valor por padrão será 3 segundos.
- Se um valor for fornecido, este é usado em vez do padrão. Por exemplo, `-i 1` define o intervalo para 1 segundo.
- Enquanto houver processos filhos em execução, o processo pai agrega periodicamente os contadores e imprime (ou escreve no ficheiro de log) a contagem parcial da palavra, o número de ficheiros já processados e o número de ficheiros ainda pendentes.

**Abordagem para a divisão dos ficheiros**
- Caso haja mais processos filhos que ficheiros lidos, o número de processos filhos será igual ao número de ficheiros lidos.
- Caso haja mais ficheiros lidos do que processos filhos, os ficheiros serão distribuídos entre os processos filhos. Por exemplo: existindo 5 ficheiros e 2 processos filhos, os processos filhos irão receber alternadamente cada ficheiro começando pelo primeiro, até que todos os ficheiros sejam distribuídos; assim, o primeiro processo filho ficará com 3 ficheiros e o segundo com 2.
- Caso haja um único ficheiro e vários processos filhos, o ficheiro é dividido em partes (chunks, por intervalos de linhas) correspondentes ao número de processos filhos.

**Comportamento do SIGINT (Ctrl+C)**
- Os processos filhos ignoram diretamente o `SIGINT`, pelo que só o processo pai reage a ele.
- Se o `SIGINT` for ativado antes da divisão dos ficheiros (ou chunks) em unidades de trabalho, o `sys.exit(0)` é acionado e é apresentada uma mensagem, pois o processo é interrompido antes da divisão dos ficheiros.
- Se o `SIGINT` for ativado depois da divisão mas antes de os processos filhos serem criados/distribuírem o trabalho, o `sys.exit(0)` é acionado e é apresentada uma mensagem, pois o processo é interrompido antes da distribuição dos ficheiros.
- Se o `SIGINT` for ativado enquanto os processos filhos já estão em execução, a flag partilhada é ativada para impedir que estes processem mais ficheiros/chunks, permitindo que o programa termine de forma controlada.

### Stack tecnológica
- Python 3
- `multiprocessing` (`Process`, `Value`, `Lock`, `Array`, `Queue`) para processamento paralelo e comunicação entre processos
- Módulo `signal` para o tratamento do `SIGINT`
- Bash (o script `pword` trata da validação/parsing dos argumentos com `getopts` antes de invocar `pword.py`)
