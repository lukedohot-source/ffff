# Entrega desta rodada e status do pedido completo

Resposta ponto a ponto ao pedido de 13 itens. Nem tudo foi feito nesta rodada,
e os itens 13 e parte da lista de entregáveis **não podem ser feitos aqui** —
isso está dito abaixo sem rodeio, com o que você precisa rodar no Windows.

---

## O que não posso entregar deste ambiente

O item 13 pede "compile o projeto e execute os testes", e a lista de
entregáveis pede versão compilada, relatório de testes e hashes dos
executáveis.

**Não há `csc.exe` nem mono neste container Linux.** Não consigo compilar,
não consigo rodar a suíte, e não consigo produzir hashes de um binário novo.
Gerar hash do `.exe` antigo seria pior que não entregar: pareceria validação e
não seria.

O projeto já tem o caminho pronto para isso. No Windows:

```
codigo-fonte\LukeOptimizer\COMPILAR.cmd
```

Ele é o `VALIDAR-5.4.ps1`: compila o executável, compila e roda os testes de
engine, compila e roda o smoke test de interface, e **só publica o binário se
tudo passar** — se um teste falhar, o executável anterior é preservado de
propósito. No fim ele chama `ATUALIZAR-HASHES.ps1`, que regenera
`MANIFESTO-SHA256.json` e `SHA256SUMS*.txt`. Os logs ficam em
`verificacao\atual\` (`build.log`, `tests.log`, `ui.log`).

Testes em Windows 10 e 11 reais, sem administrador, com arquivo em uso e com
UAC cancelado também só existem aí.

O que **eu** verifico a cada rodada, e que vale alguma coisa: balanceamento de
chaves e parênteses ignorando strings e comentários, ausência de construções
posteriores ao C# 5 (o projeto compila com `/langversion:5`), BOM e fim de
linha preservados arquivo a arquivo, consistência das listas por script, e que
o patch reproduz a árvore byte a byte.

---

## Item 4 · Rede — feito nesta rodada

Era o maior buraco real: a página de rede só media (ping, jitter, rota,
adaptadores). **Não havia nenhuma ação.** Agora há, em `source/NetworkTools.cs`.

### Diagnósticos (somente leitura, sem elevação)

| Id | O que faz | O que NÃO faz |
|---|---|---|
| `netdiag-dns` | Lista os servidores DNS de cada adaptador ativo e mede ICMP até cada um | Não mede tempo de consulta DNS — mede o caminho até o servidor. Um DNS pode ir mal no ping e bem na resolução, ou bloquear ICMP. Não troca DNS nenhum |
| `netdiag-mtu` | Busca binária com o bit "não fragmentar" até o gateway, do payload 548 ao 1472, e soma os 28 bytes de cabeçalho | Não altera MTU. Se o gateway descartar pacotes DF — comum — diz **inconclusivo** em vez de chutar |
| `netdiag-quality` | 16 pings: perda, mínimo, máximo, média e variação entre respostas consecutivas | Não mede o servidor do seu jogo, bufferbloat sob carga nem input lag |

O MTU usa `System.Net.NetworkInformation.Ping` com `PingOptions.DontFragment`,
não `ping.exe`: sem processo filho, sem argumento montado com string.

### Ações (allowlist fechada, todas exigem administrador)

| Id | Comando | Derruba conexão | Exige reiniciar |
|---|---|---|---|
| `net-flushdns` | `ipconfig /flushdns` | não | não |
| `net-registerdns` | `ipconfig /registerdns` | não | não |
| `net-arp` | `netsh interface ip delete arpcache` | pausa local curta | não |
| `net-renew` | `ipconfig /renew` | **sim** | não |
| `net-winsock` | `netsh winsock reset` | **sim** | **sim** |
| `net-ipreset` | `netsh int ip reset` | **sim** | **sim** |

Como você pediu, o que interrompe a conexão está separado do que só limpa
cache, e os dois reparos estão separados de tudo. A ordem dos botões na tela é
a ordem da consequência.

**Backup antes das duas destrutivas.** `netsh winsock show catalog` e
`netsh int ip show config` rodam primeiro e a saída vai para o registro do
Histórico. Se essa gravação falhar, **a alteração não é executada** — reparo
sem cópia do estado anterior é risco sem rede de segurança.

**Nenhuma ação de rede tem desfazer automático**, e por isso `Kind="Rede"`
fica fora de `App.CanUndo`. Isso é honestidade, não limitação escondida:
`netsh winsock reset` não tem reversão. Há teste garantindo que essas ações
nunca entrem na cadeia de desfazer.

### O que recusei em rede, e por quê

| Pedido | Por que não |
|---|---|
| Alterar MTU do adaptador | O pedido já dizia "não altere usando valores fixos sem testar". Medir é seguro; escrever MTU errado piora a conexão e o valor certo depende de PPPoE/VPN/operadora. O app mede e informa; a decisão é sua |
| Trocar servidores DNS | Muda quem resolve seus nomes e para onde vão seus pedidos. É escolha de privacidade, não otimização, e não cabe num botão automático |
| "Otimizações de QoS" | `NonBestEffortLimit` e afins só têm efeito com política de QoS configurada. Sem isso é placebo. O catálogo já traz `opt-disablenetworkthrottling` marcado como **experimental** e fora de perfil |
| Desativar economia de energia do adaptador | O estado real fica em `PnPCapabilities` por adaptador, sob um GUID de classe com índice numérico. Escrever ali sem alvo verificado e sem desfazer por adaptador é frágil demais para esta rodada |
| Desativar IPv6 | Quebra recursos que dependem dele e raramente melhora algo. Recusado |

---

## Itens já atendidos antes desta rodada

| Item | Onde está | Observação |
|---|---|---|
| 2 · Registro | `V5Options.cs`, `optimizer-options.json`, engine em `Profile.cs` | Os 12 requisitos (ID, chave, valor, tipo, anterior, novo, descrição, compatibilidade, risco, desfazer, backup, verificação) já existem via `SavedValue` + `ChangeAudit` + `OptionInfo`. `SavedValue` guarda valor, tipo **e ausência** anteriores |
| 3 · Input lag | `InputTuning.cs`, `V5InputItems.cs`, `native-mouse`, `native-gamemode`, `native-capture`, `ApplyFocus` | Comparação antes/depois existe em `ResourceSample` e na página FPS e stutter |
| 5 · Processos | `Resources.cs` | 130 apps em 3 níveis, identidade reconferida por PID + caminho + horário de início antes de encerrar, shell/sessão/anticheat fora por construção, e **reabertura** dos apps fechados |
| 6 · Serviços | `Management.cs` | 41 opcionais em 3 níveis + 54 protegidos que nunca podem ser tocados, conferidos por `ValidateServiceCatalog()` |
| 7 · Limpeza | `StorageManager.cs`, `Maintenance.cs` | Prévia, tamanho, arquivo bloqueado ignorado, revalidação antes de excluir, e filtro de idade que voltou a ser visível e configurável |
| 8 · RAM | `ProductInterface.cs`, `MemoryManager` | Working set de processos elegíveis; a tela já diz que limpeza global de standby exige mecanismo não documentado e permanece indisponível |
| 11 · Erros | `ErrorHelp` | Códigos estruturados (ACCESS_DENIED, FILE_LOCKED, TIMEOUT, CANCELLED…), sem exceção que feche o app |
| 12 · Backup | `Record`, `ChangeAudit` | A restauração confere se outro programa alterou o valor depois e recusa sobrescrever |

---

## O que ainda falta do pedido

| Item | Situação |
|---|---|
| 1 · Categorias e ações rápidas | Parcial. Existem "Limpar temporários", "Liberar RAM", "Reduzir processos", "Reduzir processos: extremo", "Sessão de jogo", "Otimização Ultra". Faltam "Otimizar rede" e "Reduzir input lag" como botão único na tela principal |
| 9 · Perfil da máquina | Não feito. `Machine` já distingue desktop/notebook e tomada, mas não classifica SSD/HD, GPU dedicada nem faixa de RAM, e as recomendações não são filtradas por isso |
| 9 · Tarefas agendadas | Não implementado. Projeto pronto em `06-modulos-novos.md §6.2`, com allowlist de 8 tarefas de coleta |
| 10 · Interface | Parcial. A grade voltou a aparecer e os textos longos saíram dos cartões, mas a navegação por seções e os badges de risco continuam propostos em `04-ui-ux.md` |
| 10 · "visual amarelo e escuro" | **Conflito com o seu pedido anterior.** Você pediu azul na mensagem anterior e o tema azul foi aplicado. Mantive o azul. Para voltar ao amarelo é uma linha: `Theme.Apply(false)` em `source/Interface.cs` |

---

## Instalação

1. Extraia o ZIP numa pasta local (não em rede, não dentro de `Arquivos de Programas`).
2. Se o Windows marcou os arquivos como baixados da internet, rode `DESBLOQUEAR.cmd`.
3. Rode `codigo-fonte\LukeOptimizer\COMPILAR.cmd` e espere terminar.
4. Confira `verificacao\atual\tests.log` e `ui.log`.
5. Abra `ABRIR.cmd`.
6. Antes de aplicar qualquer coisa, clique em **Analisar este PC**.

## Restauração

- **Registro, serviços, energia, inicialização, prioridade:** Histórico →
  selecione o registro → **Desfazer**. A restauração confere se outro programa
  mudou o valor depois e recusa sobrescrever em silêncio.
- **Apps fechados:** Apps e RAM → **Reabrir apps fechados**. Não restaura abas
  nem trabalho não salvo.
- **Limpeza de arquivos:** não tem desfazer. Por isso existe a prévia.
- **Ações de rede:** não têm desfazer. Para `winsock reset` e `int ip reset` o
  estado anterior fica gravado no registro do Histórico, para consulta manual.
- **Tudo:** os backups ficam em
  `%LOCALAPPDATA%\LukeOptimizer\historico`, um JSON por operação.

## Limitações que continuam valendo

1. O `.exe` do pacote é o antigo até você rodar `COMPILAR.cmd`.
2. Nenhuma medição de FPS é feita pelo aplicativo. O que ele compara é CPU,
   RAM, processos e latência — antes e depois, com os números que ele mesmo
   coletou. Não há benchmark inventado em lugar nenhum.
3. Desativar serviço e fechar processo reduzem trabalho em segundo plano.
   Isso **não** é a mesma coisa que ganhar FPS, e o texto de cada item diz isso.
4. DPI real de monitor, elevação UAC, gravação em HKLM sob política de domínio
   e `schtasks` só se comportam de verdade no Windows.
