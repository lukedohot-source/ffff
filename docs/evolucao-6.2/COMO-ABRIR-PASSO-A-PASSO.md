# Como abrir a nova versão — passo a passo, arquivo por arquivo

## Antes de tudo: por que compilar é obrigatório

O ZIP traz **código-fonte**, não um programa pronto e atualizado. O `.exe` que
vem dentro dele é **antigo** — é o mesmo que você já tinha, com os defeitos.

Você **precisa** executar `COMPILAR.cmd`. Sem isso, nada muda na tela.

### E havia um problema que fazia compilar não adiantar

`ABRIR.cmd` executa o arquivo:

```
native\LukeOptimizer-6.1.2.exe
```

Mas o `COMPILAR.cmd` publicava o resultado em outros dois arquivos:

```
native\LukeOptimizer-Engine.exe
native\LukeOptimizer-v5.exe
```

**Ele nunca tocava no arquivo que o `ABRIR.cmd` abre.** Compilar dava certo, o
binário novo era gravado — e o atalho continuava abrindo o antigo. Por isso a
tela seguia amarela e o erro do `TrimEnd` continuava aparecendo.

Corrigido nesta entrega. O `COMPILAR.cmd` agora publica também em
`native\LukeOptimizer-6.1.2.exe`, e **confere o SHA-256** no fim: se o arquivo
que o `ABRIR.cmd` abre não for exatamente o que acabou de ser compilado, a
compilação falha em vez de mentir.

---

## Requisitos

| Item | Detalhe |
|---|---|
| Windows | 10 ou 11, **64 bits** |
| .NET Framework | **4.8** — já vem no Windows 10 (a partir da 1903) e no Windows 11 |
| Administrador | **não** para compilar e abrir; só na hora de **aplicar** um ajuste |
| Visual Studio | **não é necessário**. O compilador usado já vem com o Windows |

---

## Passo 1 — Extrair o ZIP

Clique com o botão direito no `opt.zip` → **Extrair tudo**.

Extraia para uma pasta local e simples, por exemplo:

```
C:\LukeOptimizer
```

**Não extraia** para: `Arquivos de Programas`, `Program Files`, uma pasta de
rede, OneDrive sincronizado, ou direto de dentro do ZIP (clicar duas vezes no
ZIP e rodar de lá **não funciona** — o Windows usa uma pasta temporária).

### Duplo clique ou terminal — os dois funcionam, mas o terminal exige `.\`

Todos os passos abaixo dizem "clique duas vezes". Se você preferir o terminal,
atenção a uma diferença:

| Onde | Como executar |
|---|---|
| Explorador de Arquivos | duplo clique no `.cmd` |
| **PowerShell** | `.\COMPILAR.cmd` — **o `.\` é obrigatório** |
| Prompt de Comando (cmd) | `COMPILAR.cmd` — sem prefixo |

Por segurança, o PowerShell **não executa nada da pasta atual** sem caminho
explícito. Sem o `.\` ele responde:

```
O termo 'compilar.cmd' não é reconhecido como nome de cmdlet, função,
arquivo de script ou programa operável.
```

Isso **não é erro do pacote**. O próprio PowerShell sugere a correção no fim da
mensagem. A mesma regra vale para `.\ABRIR.cmd`, `.\DESBLOQUEAR.cmd` e
`.\TESTAR.cmd`.

### Rode um comando de cada vez — não cole um bloco inteiro

`DESBLOQUEAR.cmd` e `ABRIR.cmd` terminam com **"Pressione qualquer tecla para
continuar"**. Esse `pause` **consome a primeira tecla** do que estiver esperando
no buffer — inclusive de um texto colado.

Colar isto de uma vez:

```powershell
.\DESBLOQUEAR.cmd
cd codigo-fonte\LukeOptimizer
```

faz o `pause` engolir o `c` do `cd`, e o PowerShell recebe `d
codigo-fonte\LukeOptimizer` — que não existe. A partir daí as linhas seguintes
se embaralham e os erros parecem ser do pacote, quando são do buffer de teclado.

**Cole e execute uma linha por vez.** `COMPILAR.cmd` e `TESTAR.cmd` não pausam,
então esses podem ser encadeados sem risco.

### Atenção à pasta: `COMPILAR.cmd` não fica na raiz

| Arquivo | Onde fica |
|---|---|
| `DESBLOQUEAR.cmd`, `ABRIR.cmd`, `DIAGNOSTICO.cmd` | **raiz** do pacote |
| `COMPILAR.cmd`, `TESTAR.cmd` | `codigo-fonte\LukeOptimizer\` |

Rodar `.\COMPILAR.cmd` na raiz dá "não é reconhecido" — é só entrar na pasta
antes com `cd codigo-fonte\LukeOptimizer`.

Ao final você deve ver a pasta `LukeOptimizer-6.1.2` com este conteúdo:

```
LukeOptimizer-6.1.2\
├── ABRIR.cmd                 ← abre o programa
├── DESBLOQUEAR.cmd           ← passo 2
├── DIAGNOSTICO.cmd           ← só se der erro
├── DIAGNOSTICO-MAIN.cmd      ← só se der erro
├── VERIFICAR-PACOTE.ps1      ← confere os hashes do pacote
├── ATUALIZAR-HASHES.ps1      ← chamado pela compilação; não execute sozinho
├── FASE-1-AUDITORIA.md       ← o que foi encontrado no projeto
├── FASE-2-FUNDACAO.md        ← o que foi construído
├── RELATORIO-TESTES-EXECUTADOS.md
├── native\                   ← onde ficam os executáveis
├── codigo-fonte\LukeOptimizer\
│   ├── COMPILAR.cmd          ← passo 3 (o mais importante)
│   ├── TESTAR.cmd            ← só testa, não publica
│   ├── VALIDAR-5.4.ps1       ← o que o COMPILAR.cmd executa
│   ├── source\               ← o código
│   └── tests\                ← as suítes de teste
└── verificacao\atual\        ← logs gerados pela compilação
```

---

## Passo 2 — `DESBLOQUEAR.cmd`

**Clique duas vezes.**

O Windows marca arquivos vindos da internet com "Mark of the Web". Isso faz o
PowerShell recusar os scripts e o SmartScreen bloquear o `.exe`.

Este arquivo roda `Unblock-File` em tudo dentro da pasta. Ele **não desativa
proteção nenhuma** — só remove a marca de "veio da internet" dos arquivos que
você mesmo extraiu.

Esperado: `Arquivos do pacote desbloqueados.` e depois `Pressione qualquer
tecla`.

> Se aparecer erro, clique com o botão direito no **ZIP original** →
> **Propriedades** → marque **Desbloquear** → OK, e extraia de novo.

---

## Passo 3 — `codigo-fonte\LukeOptimizer\COMPILAR.cmd`

**Clique duas vezes.** É o passo que realmente atualiza o programa.

Ele chama o `VALIDAR-5.4.ps1`, que faz, nesta ordem:

| # | O que faz |
|---|---|
| 1 | Localiza o `csc.exe` do .NET Framework em `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\` |
| 2 | Compila o executável → `LukeOptimizer-Engine.new.exe` |
| 3 | Compila a fixture de processo usada pelos testes |
| 4 | Compila e **executa a suíte de engine** (1.114 verificações) |
| 5 | Se um teste falhar → **para aqui** e preserva o executável anterior |
| 6 | Compila e **executa a suíte de interface** |
| 7 | Se um teste falhar → **para aqui** e preserva o executável anterior |
| 8 | Publica o binário em `native\`, **inclusive em `LukeOptimizer-6.1.2.exe`** |
| 9 | Confere o SHA-256 do arquivo publicado contra o recém-compilado |
| 10 | Regenera `MANIFESTO-SHA256.json` e `SHA256SUMS.txt` |

Demora normalmente de 1 a 3 minutos. A janela fica parada durante os testes —
é esperado.

**Sinal de sucesso**, nas últimas linhas:

```
Publicado em native\LukeOptimizer-6.1.2.exe - SHA256 <64 caracteres>
TOTAL: 1114 checks passed, ...
UI smoke passed: ...
Validacao concluida. Nenhuma otimizacao ou desinstalacao real foi executada.
```

Se aparecer `Testes de logica falharam` ou `Testes de interface falharam`, o
executável **não** foi atualizado de propósito. Nesse caso veja o passo 6.

> Nenhuma configuração do Windows é alterada neste passo. Compilar e testar são
> operações de leitura; a suíte usa fixtures próprias.

---

## Passo 4 — conferir os logs (opcional, mas recomendado)

Abra a pasta:

```
verificacao\atual\
```

| Arquivo | O que é |
|---|---|
| `build.log` | compilação do executável |
| `tests.log` | suíte de engine — a **última linha** traz o total |
| `ui.log` | suíte de interface |
| `interface\` | imagens geradas pelo teste de interface |

Na última linha de `tests.log` você deve ler `1114 checks passed` e a lista de
verificações puladas com o motivo de cada uma.

---

## Passo 5 — `ABRIR.cmd`

**Clique duas vezes.** Sem administrador.

Se o código-fonte for mais recente que o executável, ele agora **avisa antes de
abrir** — exatamente para não repetir o "compilei e nada mudou". Se aparecer
esse aviso, feche a janela e volte ao passo 3.

### O que fazer na primeira abertura

1. Clique em **Analisar este PC** e espere terminar. Nada é alterado — é só
   leitura. Sem isso, a compatibilidade de cada opção aparece como
   **"Não detectado"**, que é a resposta honesta antes da leitura.
2. Vá em **Diagnóstico**. Confira a seção **PERFIL DETECTADO**: CPU, núcleos,
   SMT, GPU, RAM, tipo do disco, chassi, Secure Boot, TPM — e, no fim, a lista
   **"Não foi possível ler (e por quê)"**.
3. Vá em **Otimizações**. A grade de categorias deve aparecer preenchida — era
   ela que nascia vazia. Marque **Mostrar também as opções avançadas** para ver
   todas.
4. Em qualquer opção, clique em **Detalhes** para ver a ficha completa:
   evidência, confiança, risco, estágio, compatibilidade **neste PC** com o
   motivo, se precisa de administrador, qual reinício exige e como será
   restaurada.

### Só aplique depois de ler

Ao aplicar, o Windows vai pedir **administrador** (UAC). O backup do valor
anterior é gravado **antes** de qualquer escrita, em:

```
%LOCALAPPDATA%\LukeOptimizer\historico\
```

Para desfazer: **Histórico** → selecione o registro → **Desfazer**. A
restauração confere se outro programa mudou o valor depois e **recusa
sobrescrever** em silêncio.

---

## Passo 6 — se algo der errado

| Sintoma | Arquivo | O que fazer |
|---|---|---|
| `COMPILAR.cmd` diz `.NET Framework 4.x x64 nao encontrado` | — | Instale o .NET Framework 4.8 pelo site da Microsoft e repita |
| Testes falharam e o exe não foi publicado | `verificacao\atual\tests.log` e `ui.log` | Abra o log; a linha `FAILED:` diz qual verificação quebrou |
| `ABRIR.cmd` fecha na hora, sem janela | `DIAGNOSTICO.cmd` | Roda uma versão que imprime o erro na janela. Copie o texto |
| Janela abre e fecha depois de aparecer | `DIAGNOSTICO-MAIN.cmd` | Mostra o código de saída do programa principal |
| Quer conferir se o pacote está íntegro | `VERIFICAR-PACOTE.ps1` | Botão direito → Executar com PowerShell. Compara todos os arquivos com `MANIFESTO-SHA256.json` |
| Só quer rodar os testes, sem publicar | `TESTAR.cmd` | Mesmo processo do `COMPILAR.cmd`, mas não substitui o executável |

**Nunca** desative o Defender, o SmartScreen ou o UAC para fazer o programa
abrir. Se ele precisar disso, o problema é o programa, não a sua proteção.

---

## Resumo em quatro cliques

```
1. DESBLOQUEAR.cmd
2. codigo-fonte\LukeOptimizer\COMPILAR.cmd     ← espere terminar
3. ABRIR.cmd
4. Botão "Analisar este PC"
```

---

## Uma ressalva honesta

As duas correções descritas aqui — a publicação em
`native\LukeOptimizer-6.1.2.exe` e o aviso de binário desatualizado — são
scripts do **Windows** (`.ps1` e `.cmd`). **Não há PowerShell nem cmd neste
ambiente onde trabalhei**, então elas foram revisadas linha a linha, mas **não
foram executadas por mim**. O código C# e as duas suítes de teste, esses sim,
foram compilados e executados.

Se o passo 3 falhar de um jeito inesperado, me mande o texto da janela e o
`verificacao\atual\build.log`.
