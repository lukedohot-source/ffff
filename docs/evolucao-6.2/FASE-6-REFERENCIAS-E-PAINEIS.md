# FASE 6 — DNS completo, quatro referências auditadas, painéis compactos

## 1. Gerenciador de DNS — o item que estava pendente desde a FASE 5

Na FASE 5 eu escrevi: *"ALTERAR DNS. Você pediu, e eu não implementei."* Agora está
implementado, com o ciclo inteiro:

```
DETECTAR → PRÉVIA → BACKUP → APLICAR → CONFERIR → REGISTRAR → DESFAZER
```

**12 provedores** (Cloudflare nas três variantes, Google, Quad9, OpenDNS, AdGuard,
Mullvad, automático e personalizado), escolhidos numa **caixa de escolha** — porque doze
opções não são doze interruptores.

### As decisões que valem registro

**O nome do adaptador nunca atravessa a fronteira de elevação.** Nome de adaptador é
texto livre: a pessoa pode renomear para `Ethernet" & calc.exe`. O pedido que vai para o
processo administrador carrega **provedor e dois números** — `dns-cloudflare|14|14||` — e
o processo elevado relê os adaptadores e encontra o certo pelo índice, exigindo
coincidência em **todas** as famílias declaradas. Há teste com adaptador de nome hostil.

**Endereço abreviado é recusado.** `IPAddress.TryParse` aceita as formas herdadas de
`inet_addr`: `"1.1.1"` vira 1.1.0.1 e `"8"` vira 0.0.0.8. Quem digitasse isso veria o
endereço "aceito" e ficaria sem resolver nome nenhum. Exigir os quatro octetos fecha essa
porta — o teste que pegou isso foi escrito antes do código passar.

**Falha no meio volta sozinha; conferência que não bate NÃO volta.** São casos
diferentes: meia troca deixaria metade das consultas indo para o resolvedor antigo, então
o rollback é automático. Já uma conferência que não bate pode ser só o Windows não ter
atualizado a leitura — desfazer por conta própria aí seria trocar o DNS duas vezes sem a
pessoa pedir. O registro diz `VERIFY_FAILED` e Desfazer fica a um clique.

**"Automático" não finge conferência.** Quando o adaptador volta ao DHCP, o roteador já
entrega os servidores dele e o Windows mostra esses endereços sem dizer de onde vieram.
A etapa de conferência escreve *"sem como conferir"* em vez de "conferido".

**Desfazer recusa sobrescrever mudança posterior.** Se alguém trocou o DNS fora do app
depois da aplicação, a restauração para e explica, em vez de apagar a escolha mais nova.

---

## 2. Os quatro projetos de referência

Auditoria completa em **`REFERENCE_PARITY_REPORT.md`** (588 funções, tabela item a item).

| | |
|---|---:|
| Ajustes extraídos dos quatro projetos | **1.655** |
| Pares (chave, valor) distintos | **935** |
| Funções identificadas | **588** |
| O mesmo ajuste em mais de um projeto | **175** (20 nos quatro) |

### A duplicação que você previu é real, e está medida

Vinte ajustes estão nos **quatro projetos ao mesmo tempo** — `GameDVR_Enabled`,
`AllowTelemetry`, `AdvertisingInfo\Enabled`, `HideFileExt`, `TaskbarAl`,
`ShowTaskViewButton`, entre outros. Cada um tem **um id só** aqui.

A comparação não foi feita por nome. Cada projeto batiza a mesma coisa de um jeito
("HAGS", "Hardware Accelerated GPU Scheduling", "GPU Scheduling"); comparar nomes
produziria exatamente a duplicação que você proibiu. A chave é **o que o Windows
realmente guarda**: o par (chave de Registro, nome do valor).

### O teste de deduplicação achou dois duplicados que já existiam aqui dentro

| Valor | Ids em conflito | Resolução |
|---|---|---|
| `Mozilla\Firefox · DisableTelemetry` | `v5-native-firefox-telemetry`, `opt-disablefirefoxtelemetry` | Ficou o v5 — só aplica se o Firefox 60+ estiver mesmo instalado |
| `Explorer · DisableSearchBoxSuggestions` | `opt-disablestartmenuads` (pacote de 22), `opt-usersearchsuggestions` | Ficou o dedicado; o pacote perdeu esse valor |

O teste roda no build. Um segundo id escrevendo o mesmo valor derruba a compilação dos
testes — não depende de alguém lembrar de conferir.

### Licenças — o achado que mudou o que pode ser feito

Este pacote **já se distribui sob GPLv3** (`licenses/INTEGRACAO.txt`).

| Projeto | Licença | Consequência |
|---|---|---|
| Sophia Script | MIT | Compatível |
| Win11Debloat | MIT | Compatível |
| optimizerNXT | GPL-3.0 | Compatível (mesma licença) |
| **Winhance** | **PolyForm Shield 1.0.0** | **Código inutilizável aqui** |

A PolyForm Shield é licença *fonte-disponível*, não open source: proíbe usar o software
para competir com o licenciante — e um otimizador de Windows compete diretamente com o
Winhance. Uma licença com restrição de uso também é **incompatível com a GPLv3**, que não
admite restrições adicionais. Não é preferência: é impedimento. O Winhance foi lido como
se lê a documentação de um concorrente, e nenhuma linha dele entrou aqui.

**Critério de entrada das configurações, e ele foi restritivo:** só entrou ajuste
confirmado em **mais de uma referência independente**. Quando quatro projetos que não se
falam escrevem o mesmo valor da mesma chave, o fato está estabelecido. O que aparecia em
uma referência só e não deu para confirmar na documentação da Microsoft ficou de fora —
marcado como ausente no relatório, não entregue como função.

---

## 3. Painéis compactos — 39 configurações novas

Quatro páginas novas na barra lateral: **Personalizar Windows**, **Privacidade**,
**Jogos**, **Windows Update**.

```
BARRA DE TAREFAS
Botão Visão de Tarefas                                    [ ]
Remove da barra o botão que mostra as áreas de trabalho…

Agrupar botões da barra              [ Quando cheia ▼ ]
Decide quando janelas do mesmo programa viram um botão só.

Finalizar tarefa no botão direito ✓                       [ ]
Acrescenta "Finalizar tarefa" ao menu do botão direito…
```

**Linha compacta, não cartão gigante.** Nome + indicador + controle à direita, descrição
curta embaixo, e o **ⓘ** abre a ficha completa: o que faz, estado atual, classificação,
reinício, administrador, compatibilidade, como voltar atrás, detalhes técnicos e fonte.

**Três controles, e a escolha entre eles não é estética:**

- **Interruptor** quando só há dois estados.
- **Caixa de escolha** quando há mais de dois. "Sempre", "Quando cheia" e "Nunca" não são
  três chaves independentes — são uma decisão só, e três interruptores mutuamente
  exclusivos mentiriam sobre isso.
- **Botão** para ação. Rodar uma verificação de arquivos não é estado que fica ligado.

**Nada aplica sozinho.** Marcar acumula alteração pendente; a barra de baixo mostra
`3 ALTERAÇÕES PENDENTES` com **Descartar · Revisar · Aplicar 3 alterações**. Revisar
mostra opção por opção: estado agora, o que vai ficar, risco, reinício, administrador e
como desfazer. Uma elevação para o lote inteiro.

**Indicadores pequenos** ao lado do nome: ✓ recomendado · ⚠ avançado · 🧪 experimental ·
↻ exige reinício. Avançadas e experimentais ficam **escondidas por padrão**, com filtro
para revelar. Incompatíveis com este PC também.

**A busca acha o nome técnico.** Digitar `HAGS` ou `HwSchMode` encontra "Agendamento de
GPU pelo hardware"; `Win32PrioritySeparation` encontra "Prioridade de programas em
primeiro plano".

**As novas opções NÃO aparecem na tela Otimizações.** Elas têm painel próprio; repeti-las
encheria justamente a tela que você pediu para enxugar — e seriam as mesmas opções, com o
mesmo id e o mesmo backend, aparecendo duas vezes.

### Uma reclassificação que eu preciso declarar

`HwSchMode`, `Win32PrioritySeparation` e `SystemResponsiveness` estavam numa lista de
"placebos conhecidos" e eram **proibidos** de virar opção executável — decisão de uma
fase anterior, minha. Seus itens 20, 21 e 125 pedem os três. Mudei, e explico:

- **Agendamento de GPU não é placebo.** É recurso documentado da Microsoft, com
  interruptor próprio em Configurações do Windows. Classificá-lo como placebo era exagero
  meu. Virou opção **avançada**, dizendo que o efeito varia por placa e driver.
- **Os outros dois são valores reais cujo GANHO é que não tem comprovação.** Isso é
  **experimental**, não placebo — e é assim que entraram, com 🧪 e "EXPERIMENTAL" escrito
  no texto que a pessoa lê.
- Continuam proibidos de executar: `LargeSystemCache`, `DisablePagingExecutive`,
  `IoPageLockLimit`, `TcpAckFrequency`, `TCPNoDelay`, `SvcHostSplitThresholdInKB`.

O que protege não é esconder: é nenhum deles vir marcado, nenhum entrar em perfil
automático, e o experimental dizer que é experimental. Três testes garantem isso.

---

## 4. Verificação

```
Motor:      2893 verificações, 6 puladas   (era 1803)
Interface:  suíte completa + 4 painéis, 39 configurações
Executável: compila
```

**+1090 verificações.** As que importam:

- nenhum id do catálogo grava o mesmo valor da mesma chave que outro — **um id por função**
- nenhum projeto de referência é citado no texto que o usuário lê
- toda configuração aponta para documentação da Microsoft, não para o projeto de origem
- toda configuração declara risco, e o experimental diz que é experimental
- nenhuma configuração nova vem marcada nem entra em perfil automático
- a tela manda para a elevação exatamente os ids marcados — nem mais, nem menos
- um adaptador de nome hostil nunca chega à linha de comando do DNS
- falha no meio da troca de DNS volta ao estado anterior; conferência que não bate não volta
- desfazer recusa sobrescrever DNS alterado depois

### Não testado

- **Em Windows real:** os painéis aplicando de fato, a troca de DNS executando `netsh`, e
  a elevação em lote. O ciclo de DNS foi exercitado ponta a ponta contra um adaptador
  simulado que obedece aos comandos — o que não cobre o `netsh` de verdade.
- **Visual:** neste ambiente de build o `DrawToBitmap` do mono/Xvfb devolve imagem vazia
  **para todos os testes**, não só os novos. Os painéis foram verificados por estrutura
  (controle existe, conta certo, envia o id certo), não por aparência. O visual precisa de
  conferência em Windows.

### Uma falha que só o Windows pegou

A primeira compilação no PC do usuário falhou com **CS1985: não é possível esperar no
corpo de uma cláusula catch**. Eu tinha escrito um `await` dentro de um `catch` no
`DnsInterface.cs` — construção que só passou a ser válida no C# 6, e este projeto compila
com `/langversion:5`.

O `mcs` do mono, que eu uso aqui, **aceitou em silêncio mesmo com `/langversion:5`**. Ou
seja: compilar limpo no meu ambiente não prova que compila no `csc` do Windows. As
restrições que o mono deixa passar são a `await` em `catch` e em `finally`.

Corrigido guardando o motivo da recusa e mostrando a tela depois do `catch` — o mesmo
formato que `FunctionalInterface` já usava. E ficou uma varredura no processo de revisão
procurando `await` dentro de `catch`/`finally` em todos os arquivos, já que o compilador
daqui não acusa.

---

## 5. O que continua faltando

Do seu pedido de 193 itens, o que **não** foi feito — e está assim no relatório, não
escondido:

| | |
|---|---|
| Aplicativos do fabricante (OEM) | detecção de fabricante antes de mostrar |
| Central de IA · Recall · Click to Do | dependem de build/hardware que não dá para detectar sem uma máquina com eles |
| Energia completa | AC/DC, sono, USB, PCI Express |
| Recursos do Windows · WSL · Sandbox · Hyper-V | exige DISM |
| Telemetria de Chrome, Office, Visual Studio, NVIDIA | Firefox e Edge já têm dono |
| Estado de VBS, Defender, SmartScreen, Restauração | mostrar estado, nunca desligar |
| Criador de mídia do Windows | fase posterior, como você colocou |
| Barra de pendências global · Ctrl+K · assistente inicial | a barra existe por painel; global não |
| Pastas do usuário · associações de arquivo · terminal padrão | |

Nenhum deles virou card "Em breve". Entram quando estiverem completos: nome, interface,
backend, estado, compatibilidade, execução, verificação, erro, log e rollback.
