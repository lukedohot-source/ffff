# FASE 5 — Os scripts saem da interface; a Central de Rede entra

## 1. A correção: eu tinha entendido errado

Na FASE 3 eu mantive as 546 entradas de arquivo (`.bat`, `.reg`, instaladores de
terceiros) visíveis no catálogo, sob a categoria **"Arquivos que você enviou"**.
Argumentei que o nome do arquivo era a identidade e que renomear destruiria a
rastreabilidade.

O argumento vale para um **inventário**. E o pedido nunca foi por um inventário.

Esses arquivos são **material de referência de desenvolvimento**: serviram para
descobrir funcionalidade, que depois virou recurso nativo. Apresentá-los como
coisa clicável na tela do usuário trata material de desenvolvimento como produto.
Eu racionalizei em vez de acatar, depois de o pedido já ter sido feito.

### O que mudou

| Antes | Agora |
|---|---|
| Catálogo mostrava 673 itens, 546 deles arquivos | Mostra as **127 funções executáveis** |
| Categoria "Arquivos que você enviou" | Não existe na experiência normal |
| Tela **Ferramentas**: um botão por `.bat`, rotulado com o nome do arquivo | **20 ferramentas do Windows** de verdade |
| Modo fácil citava "os 14 BATs recebidos" | Texto removido |
| Página "Seus 14 BATs" | Renomeada e fora da navegação normal |
| Atalho `bats` no mapa do protocolo | Removido |

### A auditoria não foi jogada fora

`App.DeveloperMode` — **desligado por padrão, nunca ligado sozinho**. Com ele
ativo, `FeatureNaming.ForUser` devolve o catálogo inteiro, sem renomear nada: lá
o nome do arquivo volta a ser a identidade, porque é para isso que a auditoria
serve. Origem, hash, achados e parecer continuam guardados.

Há teste garantindo os dois lados: que a experiência normal **só** mostra função
executável, e que o modo desenvolvedor recupera **tudo**, sem descarte.

> **Prova de que a integração é real:** quem apagar os `.bat` originais do disco
> continua com todas as funções integradas. Nenhuma delas lê, executa ou depende
> daqueles arquivos.

---

## 2. Tela Ferramentas: ferramentas de verdade

`source/WindowsTools.cs` — 20 consoles e páginas nativas do Windows:

Painel avançado do Windows (God Mode) · Gerenciador de dispositivos ·
Gerenciamento de disco · Serviços · Agendador de tarefas · Visualizador de
eventos · Gerenciador de tarefas · Painel de controle · Recursos do Windows ·
Conexões de rede · Firewall avançado · Propriedades do sistema · Monitor de
recursos · Informações do sistema · Limpeza de disco · Otimizar unidades ·
Diagnóstico de memória · Segurança do Windows · Windows Update · Aplicativos de
inicialização

**Nenhuma delas altera configuração.** São atalhos: o LukeOptimizer abre o
console nativo e sai do caminho. Quem muda algo é a ferramenta do Windows, com a
própria confirmação e o próprio desfazer.

O "God Mode" abre o mesmo destino pelo shell em vez de criar a pasta com CLSID no
disco da pessoa — mesmo resultado, sem deixar lixo.

Teste: todo atalho tem id único, descrição, destino real, e executável resolvido
**pelo nome em System32**, nunca por caminho.

---

## 3. Central de Rede — implementada

`source/NetworkCenter.cs`. O backend já existia e já era testado (6 ações numa
allowlist fechada, 3 diagnósticos somente leitura). Faltava a tradução.

| Na tela | O que faz por baixo |
|---|---|
| **Limpar IP / Internet** | `ipconfig /flushdns` + limpeza ARP + `ipconfig /renew` |
| **Reparar conexão** | `netsh winsock reset` + `netsh int ip reset` |
| **Registrar nomes na rede** | `ipconfig /registerdns` |
| **Testar conexão** | perda, mínimo, máximo, média, variação |
| **Ver servidores DNS** | DNS por adaptador + tempo até cada um |
| **Descobrir tamanho máximo de pacote** | busca binária de MTU até o gateway |

Os comandos aparecem **só em Detalhes técnicos**, com efeito, custo, tempo
limite, se precisa de administrador, o backup tirado antes e a fonte oficial.

### Decisões que valem registro

**As bandeiras de risco são derivadas, não digitadas.** `Disconnects`,
`NeedsRestart` e `Administrator` de cada tarefa vêm das ações que ela compõe. Se
uma ação passar a derrubar a conexão, o aviso da tarefa acompanha sozinho.

**Uma elevação por tarefa, não por comando.** O verbo `--network-actions` executa
a sequência sob um único UAC. Pedir confirmação a cada comando treina a pessoa a
clicar "sim" sem ler. Cada id passa por `NetworkTools.Resolve` **antes** da
elevação: um id fora da allowlist derruba a chamada ali, não no meio da sequência
já elevada.

**Uma tarefa lê ou altera, nunca as duas.** Misturar faria o aviso de risco
mentir. Há teste.

### O que NÃO entrou, e por quê

**ALTERAR DNS.** Você pediu, e eu não implementei. Trocar DNS exige gravar
configuração por adaptador, com backup e restauração individuais — senão a pessoa
fica sem internet e sem caminho de volta. Isso não cabia junto com o resto desta
rodada, e seu próprio pedido proíbe botão que promete e não cumpre. É o primeiro
item da próxima fase.

---

## 4. Uma fonte de verdade para os testes

Você apontou totais conflitantes. Estavam certos: o `tests.log` empacotado dizia
**981 verificações**; a build atual faz **1681**. Eram duas verdades no mesmo ZIP.

`verificacao/atual/` foi esvaziada e agora traz só um `LEIA-ME.txt` explicando
que **quem gera essa pasta é o `COMPILAR.cmd`**, que a regenera inteira a cada
execução. Pasta vazia = pacote ainda não compilado neste PC.

---

## 5. Verificação

```
Engine:     1681 verificações, 6 puladas   (era 1390)
Interface:  suíte completa, 1 bloco pulado
Executável: compila
```

**+291 verificações** nesta fase. As que importam:

- a experiência normal **só** mostra função executável, nunca arquivo recebido
- nenhuma categoria normal se chama por tipo de arquivo
- modo desenvolvedor recupera a auditoria inteira, sem descarte
- modo desenvolvedor está desligado por padrão e não se liga sozinho
- toda ferramenta do Windows tem destino real — **sem botão vazio**
- toda tarefa de rede aponta para ação ou diagnóstico que existe na allowlist
- a confirmação diz o custo **antes**, em português
- o comando aparece **só** em Detalhes técnicos
- um id inválido na sequência é recusado **antes** de qualquer elevação

**Não testado em Windows real:** a tela Rede executando de fato, a elevação da
sequência composta, e os 20 atalhos abrindo cada console.

---

## 6. O que continua pendente

Do seu pedido, por ordem do que eu faria em seguida:

| | |
|---|---|
| **ALTERAR DNS** | precisa de backup e restauração por adaptador |
| Central de limpeza | analisar → mostrar tamanho → remover → mostrar o removido |
| Reparar Windows | SFC e DISM pelo ExecutionHub |
| Energia | planos, hibernação, inicialização rápida |
| Personalização | Taskbar, Start, Explorer, aparência, mouse, teclado |
| Sidebar de 20 seções | hoje são 10 |
| Busca global e Ctrl+K | |
| Assistente de primeira execução | |

Nenhum deles vira card "Em breve". Entram quando estiverem completos: UI,
backend, estado, compatibilidade, execução, verificação, erro, log e rollback.
