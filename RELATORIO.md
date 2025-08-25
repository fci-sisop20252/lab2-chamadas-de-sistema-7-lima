# 📝 Relatório do Laboratório 2 - Chamadas de Sistema

---

## 1️⃣ Exercício 1a - Observação printf() vs 1b - write()

### 💻 Comandos executados:
```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### 🔍 Análise

**1. Quantas syscalls write() cada programa gerou?**
- ex1a_printf: 8 syscalls
- ex1b_write: 7 syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**
Há diferença, pois o printf é capaz de escrever textos para o usuário, formatar dados e é utilizado para casos que não seja necessário uma performance crítica.
E o write é capaz de escrever, além de texto, dados binários, controlar quando enviar os dados e possui um comportamento previsível.

**3. Qual método é mais previsível? Por quê você acha isso?**

```
O write() é mais previsível, porque cada chamada envia os dados imediatamente para o sistema operacional, gerando sempre uma syscall. Ele não depende de buffer, de quebras de linha ou do tamanho da string.
```

---

## 2️⃣ Exercício 2 - Leitura de Arquivo

### 📊 Resultados da execução:
- File descriptor: _____
- Bytes lidos: _____

### 🔧 Comando strace:
```bash
strace -e openat,read,close ./ex2_leitura
```

### 🔍 Análise

**1. Qual file descriptor foi usado? Por que não começou em 0, 1 ou 2?**

```
O descriptor 3 foi usado, porque os três primeiros (0, 1 e 2) já têm funções fixas no Linux: 0 é a entrada padrão, 1 a saída padrão e 2 o erro padrão. Então, quando o programa abre um arquivo, o primeiro número livre disponível é o 3.
```

**2. Como você sabe que o arquivo foi lido completamente?**

```
O teste1.txt ocupa 124 bytes, e o programa usa um buffer de 128 bytes, deixando um espaço para o caractere nulo \0.
Como o arquivo cabe inteiramente no buffer, ele é carregado de uma vez só na memória.
```

**3. Por que verificar retorno de cada syscall?**

```
Verificar o retorno de cada syscall ajuda o programa a perceber se algo deu errado e agir de acordo.
```

---

## 3️⃣ Exercício 3 - Contador com Loop

### 📋 Resultados (BUFFER_SIZE = 64):
- Linhas: _____ (esperado: 25)
- Caracteres: _____
- Chamadas read(): _____
- Tempo: _____ segundos

### 🧪 Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
|-------------|-----------------|-----------|
| 16          |                 |           |
| 64          |                 |           |
| 256         |                 |           |
| 1024        |                 |           |

### 🔍 Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

```
O tamanho do buffer influencia diretamente quantas vezes o read() precisa ser chamado para ler um arquivo. Buffers maiores conseguem pegar mais dados de uma vez, reduzindo o número de chamadas, enquanto buffers menores exigem mais chamadas para ler tudo.
```

**2. Todas as chamadas read() retornaram BUFFER_SIZE bytes? Discorra brevemente sobre**

```
As chamadas de read() nem sempre retornam a quantidade exata de bytes do buffer.

Na primeira leitura com 1024 bytes de buffer, todos os bytes foram preenchidos.

Na segunda, vieram apenas 276 bytes, que eram os que restavam do arquivo de 1300 bytes.

A terceira leitura retornou 0, mostrando que o arquivo chegou ao fim (EOF).
```

**3. Qual é a relação entre syscalls e performance?**

```
O número de syscalls afeta diretamente a performance do programa. Cada read() faz a transição entre o espaço do usuário e o kernel, que é relativamente cara. Quanto mais chamadas forem feitas, maior a sobrecarga, o que pode deixar a leitura menos eficiente. Usar buffers maiores reduz o número de syscalls, alivia a sobrecarga e melhora a performance.
```

---

## 4️⃣ Exercício 4 - Cópia de Arquivo

### 📈 Resultados:
- Bytes copiados: _____
- Operações: _____
- Tempo: _____ segundos
- Throughput: _____ KB/s

### ✅ Verificação:
```bash
diff dados/origem.txt dados/destino.txt
```
Resultado: [ ] Idênticos [ ] Diferentes

### 🔍 Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

```
O write() nem sempre grava todos os bytes que o read() trouxe, seja por o buffer estar cheio ou por algum erro de entrada/saída. Conferir isso garante que todos os dados lidos realmente foram escritos.
```

**2. Que flags são essenciais no open() do destino?**

```
O_WRONLY, O_CREAT, O_TRUNC 
```

**3. O número de reads e writes é igual? Por quê?**

```
Sim, porque cada read() que traz dados válidos é imediatamente seguido por um write() que grava exatamente o mesmo conteúdo.
```

**4. Como você saberia se o disco ficou cheio?**

```
Dá para saber que o disco encheu se o write() não conseguir gravar todos os bytes lidos, retornando menos que o esperado ou -1 com o erro ENOSPC.
```

**5. O que acontece se esquecer de fechar os arquivos?**

```
Se os arquivos não forem fechados, os file descriptors permanecem ocupados até o fim do programa, o que pode causar vazamento de descriptors se muitos arquivos forem abertos.
Além disso, em arquivos gravados, parte dos dados pode ficar sem ser escrita no disco
```

---

## 🎯 Análise Geral

### 📖 Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

```
As syscalls são chamadas que o programa faz em modo usuário para pedir serviços ao kernel. O strace mostra essas chamadas, evidenciando que sempre que o programa precisa acessar o disco, ele solicita ao kernel, já que o usuário não mexe diretamente no hardware.
```

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

```
Pelo que entendi, os file descriptors são importantes porque servem como identificadores que o kernel usa para gerenciar arquivos abertos e direcionar as operações de leitura e escrita.
```

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

```
Buffers maiores melhoram a performance porque permitem ler e escrever mais dados de uma vez, reduzindo o número de chamadas read() e write() e diminuindo o custo das transições para o kernel.
```

### ⚡ Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** _____

**Por que você acha que foi mais rápido?**

```
O ./ex4_copia é mais rápido porque só se preocupa em copiar os bytes do arquivo, sem fazer checagens extras. O cp, mesmo copiando dois arquivos .txt, precisa verificar permissões, links, diretórios e outras opções, o que deixa o processo mais lento. Por isso, para tarefas simples, um programa compilado costuma ser mais eficiente que o cp.
```

---

## 📤 Entrega
Certifique-se de ter:
- [ ] Todos os códigos com TODOs completados
- [ ] Traces salvos em `traces/`
- [ ] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
# Bom trabalho!
