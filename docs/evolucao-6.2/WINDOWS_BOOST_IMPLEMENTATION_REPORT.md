# RELATÓRIO DE IMPLEMENTAÇÃO — Windows Boost e Windows Boost Essential

Companheiro de `WINDOWS_BOOST_COMPLETE_INVENTORY.md` (o que existe nos pacotes) e de
`WINDOWS_BOOST_PARITY_REPORT.md` (os números). Este aqui diz **o que virou código**.

---

## 1. O que a auditoria encontrou

| | |
|---|---:|
| Arquivos com comportamento | **86** |
| Comportamentos extraídos do conteúdo | **2.518** |
| Ajustes de Registro distintos | **183** |
| …repetidos nos **dois** pacotes | **73** |
| …que **já existiam** no LukeOptimizer | **35** |
| Executáveis de jogo com prioridade por IFEO | **87** |
| Serviços distintos | **31** |
| Pacotes de aplicativo | **31** |

A duplicação é o traço mais marcante. As quatro tarefas agendadas de telemetria aparecem
**11 vezes cada**; as exclusões de temporários, 11 a 13 vezes. Isso porque cada script
"otimizar jogo X" é uma cópia integral do anterior com um executável trocado — e, em três
casos, **nem o executável foi trocado**.

---

## 2. Funções implementadas nesta rodada

19 configurações novas, todas pelo motor transacional existente: detecção de estado,
backup antes de escrever, verificação depois, Histórico e Desfazer.

| Feature ID | Nome na tela | Origem no pacote | Categoria | Reinício |
|---|---|---|---|---|
| `set-gpu-mpo` | Sobreposição de camadas de vídeo | `Desativar MPO.reg` (AMD, NVIDIA, INTEL — 3 cópias) | Avançado | sim |
| `set-gaming-fullscreen` | Otimizações de tela cheia | `GameDVR_FSEBehaviorMode` nos dois pacotes | Avançado | sair e entrar |
| `set-gaming-audio-capture` | Gravar áudio junto com o vídeo | `Desativar Game DVR.reg` | Opcional | não |
| `set-gaming-capture-battery` | Gravar em segundo plano na bateria | `Desativar Game DVR.reg` | Opcional | não |
| `set-gaming-gamebar-widgets` | Widgets da Game Bar | `iGust Windows Boost.bat` | Opcional | não |
| `set-gaming-mmcss` | Prioridade do agendador para jogos | tarefa `Games` do MMCSS, nos dois pacotes | **Experimental** | sim |
| `set-lab-timer-serialize` | Serializar expiração de timers | `SerializeTimerExpiration` | **Experimental** | sim |
| `set-amd-web-content` | Conteúdo online do AMD Software | `Desativar AMD Overlay e Telemetria.reg` | Opcional | não |
| `set-amd-auto-update` | Atualizações automáticas do AMD Software | idem | Opcional | não |
| `set-amd-ulps` | Economia de energia entre GPUs AMD | `Desativar ULPS.reg` | Avançado | sim |
| `set-nvidia-shadowplay` | Gravação da NVIDIA (ShadowPlay) | `Desativar NVIDIA ShadowPlay.reg` | Opcional | não |
| `set-search-history` | Histórico de pesquisa | `Desativar Sugestões de Pesquisa.bat` | Opcional | não |
| `set-privacy-activity-upload` | Enviar histórico de atividades | `Desativar histórico de atividade.bat` | Recomendado | não |
| `set-privacy-error-reporting` | Relatórios de erro do Windows | `iGust Windows Boost.bat` | Opcional | não |
| `set-privacy-consumer-experiences` | Experiências do consumidor | `Desativar Anúncios e sugestões.bat` | Recomendado | não |
| `set-system-pca` | Assistente de compatibilidade de programas | `iGust Windows Boost.bat` | Avançado | não |
| `set-update-no-internet-locations` | Buscar atualizações só no servidor da empresa | `Desativar Windows Telemetria.bat` | Avançado | não |
| `set-restore-frequency` | Permitir pontos de restauração seguidos | `Fazer backup do Windows.bat` | Recomendado | não |
| `set-appearance-visualfx` | Efeitos visuais do Windows | `Desativar efeitos visuais.bat` | Opcional | sair e entrar |

Todas têm: UI · backend · detecção de estado · compatibilidade · prévia · backup ·
aplicação · verificação · erro tratado · log · rollback.

### Detecção por fabricante de GPU

As quatro opções de AMD e NVIDIA só aparecem quando **aquela** placa existe na máquina.
Os pacotes originais gravam esses valores em qualquer PC — escrever `EnableUlps` num
computador com NVIDIA não faz nada além de sujar o Registro.

Quando a placa não pode ser identificada com segurança, a opção **aparece**. Errar
escondendo é pior que errar mostrando: a ficha do ⓘ diz qual placa a opção exige e qual
foi detectada.

---

## 3. Separações que os pacotes não fizeram

**Telemetria ≠ Windows Update.** O script `Desativar Windows Telemetria.bat` também grava
`DoNotConnectToWindowsUpdateInternetLocations`, que impede o Windows Update de falar com
a Microsoft. Num PC doméstico isso essencialmente **para as atualizações de segurança**.
Virou feature separada, no painel de Windows Update, com o efeito escrito.

**"Prioridade da GPU" que não é de GPU.** Dois `.reg` com nomes diferentes — um diz
"Aumentar Prioridade da GPU (DirectX e Renderização)", o outro "Aumentar velocidade ao
abrir pastas" — gravam **exatamente o mesmo valor**: `NtfsDisableLastAccessUpdate`. Não
tem relação com GPU nem com DirectX. É uma feature só, e já existia aqui como
`opt-disablentfstimestamp`.

**"Otimização do sistema de arquivos" que é de rede.** `Habilitar a otimização do sistema
de arquivos.reg` altera `EnableOplocks` do LanmanServer — cache de arquivos compartilhados
em rede.

---

## 4. Defeitos encontrados e NÃO reproduzidos

Confirmados lendo o conteúdo, não o nome:

| Arquivo | Defeito |
|---|---|
| `Otimizar CALL OF DUTY BLACK OPS (TODOS).bat` | Grava prioridade para `FortniteClient-Win64-Shipping.exe` |
| `Otimizar GTA V.bat` | Também grava para `FortniteClient-Win64-Shipping.exe`, não para `GTA5.exe` |
| `Otimizar RED DEAD REDEMPTION 2.bat` | Limpa `FortniteGame\Saved\WebCache` |
| `powercfg -duplicatescheme` + `/setactive SCHEME_CURRENT` | Cria o plano Desempenho Máximo e **reativa o plano atual** — o plano novo nunca é ativado, e cada execução cria outra cópia |

Uma suspeita minha **não** se confirmou: achei que `%windows%` fosse variável inexistente
nas linhas de limpeza. Está definida (`set "windows=%windir%"`) e as exclusões funcionam.
Registro aqui porque quase reportei um defeito que não existe.

---

## 5. O que NÃO entrou

| Item | Motivo |
|---|---|
| `Ativar Windows.bat` e `Ativar Pacote Office.bat` | Baixam e executam payload remoto de ativação. Não é otimização. Não integrado, não executado, sem botão. |
| `RequirePlatformSecurityFeatures = 0`, `LsaCfgFlags = 0` | Desligam proteção de virtualização e da autoridade de segurança local. O pedido é explícito: mostrar estado, nunca desligar por padrão. Mostrar estado ainda **não** foi implementado — está como ausente. |
| `LargeSystemCache` | Continua na lista de placebos proibidos de executar. Não há motivo técnico defensável em Windows moderno. |
| `del %windows%\system32\dllcache\*.*` | Pasta de Windows antigo; apagá-la em Windows moderno não tem efeito útil. Não reproduzido. |
| Executáveis de terceiros (DNS Jumper, Autoruns, ISLC, EmptyStandbyList, Firemin, MSI Mode Utility, OpenHardwareMonitor, FilterKeysSetter) | Nenhum redistribuído. As capacidades úteis devem ser implementadas nativamente — parte já está (DNS, memória, inicialização), parte não. |

---

## 6. O que continua AUSENTE

Honestamente listado, não escondido. Dos **148** ajustes de Registro ausentes, **19**
foram implementados nesta rodada. O que falta, por ordem de valor:

| Família | Ausentes | Observação |
|---|---:|---|
| Prioridade por jogo (IFEO) | 87 executáveis | São **dados**, não 87 features. Precisam do Game Profile Engine com detecção de jogo e prioridade escolhida pelo usuário — não `CpuPriorityClass=3` fixo para todos. |
| Energia (AC/DC, ocioso do processador, USB) | 13 | Precisa de backend `powercfg` por plano, com captura do GUID — e sem o defeito do `SCHEME_CURRENT`. |
| Serviços | 16 | O catálogo de serviços existe; faltam estes 16 com descrição humana e detecção de dependência. |
| Aplicativos (AppX) | 21 | O gerenciador existe com 141 definições; faltam estas, com Package Family Name correto em vez de curinga `*store*`. |
| Tarefas agendadas de telemetria | 4 | Consolidator, ProgramDataUpdater, Autochk Proxy, DiskDiagnosticDataCollector. |
| BCD (`useplatformtick`, `disabledynamictick`, `tscsyncpolicy`) | 3 | Exige backup da configuração de inicialização antes. Laboratório. |
| Estado de VBS, HVCI, SmartScreen, LSA | 4 | Mostrar estado, nunca desligar. |
| Memória (MMAgent, standby list, working sets) | 2 comandos | `Disable-MMAgent`/`Enable-MMAgent` e limpeza de standby. |
| Cache de shaders NVIDIA/AMD | — | Ação de limpeza, com aviso de recompilação. |

---

## 7. Verificação

```
COMPILADO ........................ sim
TESTE UNITÁRIO ................... 3434 verificações, 6 puladas
TESTE DE INTEGRAÇÃO .............. dentro da suíte (transação, rollback, deduplicação)
UI SMOKE ......................... 4 painéis, 58 configurações
TESTADO EM WINDOWS REAL .......... NÃO
```

**Nenhuma das 19 funções novas foi aplicada em Windows real.** Os testes usam registro
simulado. Para provar o efeito na máquina de verdade existe a **Conferência real** em
Diagnóstico → *Provar que aplicar funciona*, que aplica e desfaz cada opção e diz, uma a
uma, se o Windows mudou e se voltou ao original.
