# FASE 2 — Fundação

O que esta fase entrega é a base que permite as fases seguintes sem transformar
o código num monólito: um **contrato único por otimização**, um **perfil de
máquina estruturado**, um **motor de compatibilidade com quatro estados** e
**invariantes conferidos por teste**.

Nada foi reescrito. IDs, perfis, histórico e desfazer continuam sobre as
estruturas atuais — trocar o modelo quebraria backups já gravados no PC das
pessoas. A fundação é uma camada **por cima**, e a migração é item a item.

---

## 1. Arquivos

| Arquivo | Conteúdo |
|---|---|
| `source/MachineProfile.cs` (novo) | `MachineProfile` + `App.ReadProfile` + `App.ProfileReport` |
| `source/Optimizations.cs` (novo) | `Evidence`, `Stage`, `RestartPlan`/`RestartScope`, `OptimizationRequirements`, `OptimizationDescriptor`, `CompatibilityEngine`, `OptimizationRegistry` |
| `tests/FoundationTests.cs` (novo) | 95 verificações |
| `source/ExtremeProfile.cs` | grupo vazio não pode vir marcado (defeito 3.4 da auditoria) |
| `source/V5Options.cs` | expõe `BuildRange`, para o contrato não repetir números |
| `source/Interface.cs` | lê o perfil na análise; mostra perfil e score no Diagnóstico |
| `source/UnifiedInterface.cs` | ficha "Detalhes" passa a vir do contrato |
| `compilacao.rsp`, `VALIDAR-5.4.ps1` | registram os arquivos novos |

---

## 2. Contrato único (`OptimizationDescriptor`)

Reúne sob o mesmo `Id` o que estava em quatro lugares — `Item` (o que escrever),
`OptionInfo` (classificação, risco, reinício), `PerformanceEntry` (efeito, custo,
fonte) e texto solto na interface — e acrescenta o que faltava:

```
Id · Título · Categoria · Subcategoria
Risco          (dano possível)
Confiança      (qualidade da EVIDÊNCIA — eixo diferente do risco)
Evidência      (de onde vem a afirmação)
Estágio        (Estável · Beta · Experimental · Desenvolvedor)
Efeito · Contrapartida · Passos · Fonte
Classificação · Reinício · Desfazer · Estratégia de backup
Administrador · Reversível · Elegível a perfil automático
Requisitos     (build mín./máx., fabricante de CPU/GPU, mídia do disco,
                RAM mínima, desktop, tomada, SMT)
Tags           (para a busca global)
```

**127 descritores**, um por otimização executável. Nem um a mais.

---

## 3. Níveis de evidência — e por que um deles é desconfortável

| Nível | Quantas | Confiança |
|---|---|---|
| Documentada pela Microsoft | 52 | Alta |
| **Adaptada de projeto de código aberto (auditável, sem medição própria)** | **61** | Média |
| Documentada pelo fabricante | 2 | Alta |
| Experimental / teste A/B | 6 | Experimental |
| Não recomendada | 6 | Média |

O rótulo do meio é o ponto. **61 das 127 opções executáveis vieram de um projeto
de otimizador no GitHub.** Seria fácil chamá-las de "tecnicamente justificadas"
— soa melhor e ninguém conferiria. Mas na hierarquia de fontes do próprio
pedido, um projeto de terceiro é prioridade 7, e o pedido diz explicitamente que
"outros optimizers não são prova técnica".

Então elas têm rótulo próprio, que diz exatamente o que são: **código auditável,
sem medição própria**. `github.com/microsoft/...` continua contando como
Microsoft, porque aí é a Microsoft publicando código.

A dedução é determinística e testada: a mesma URL sempre dá o mesmo nível.
Itens cujo problema não é a fonte e sim o efeito — `NetworkThrottlingIndex`,
compressão de memória, `SystemResponsiveness`, `Win32PrioritySeparation`,
desativar SmartScreen/VBS/HVCI/TPM/SMB2 — têm nível declarado à mão, e **fonte
da Microsoft não os promove**. Há teste para isso.

### Confiança não é risco

Um item de **alta confiança** pode ter **risco alto**: está muito bem documentado
que desativar o SmartScreen reduz proteção. Um de **baixa confiança** pode ter
risco baixo: reversível, inofensivo, mas sem medição. Misturar os dois eixos é
como a maioria dos "otimizadores" convence o usuário — por isso são campos
separados, com teste garantindo a separação.

---

## 4. Perfil de máquina (`MachineProfile`)

Lê, quando o Windows expõe: nome e **fabricante** da CPU, núcleos físicos e
lógicos, **SMT/Hyper-Threading**, nome e **fabricante** da GPU, **bytes** de RAM
e faixa, **tipo de mídia** do disco do sistema (NVMe/SSD/HD), espaço livre,
**chassi** (desktop/notebook), energia, **tipo e velocidade do link de rede**,
**Secure Boot**, **TPM**, fabricante, modelo e versão do BIOS.

### A regra que sustenta tudo

O que não for lido do sistema fica **vazio**, o par `*Known` fica `false`, e o
motivo entra em `Notes`. Nunca há valor plausível inventado no lugar. O relatório
mostra literalmente "não detectado" e, no fim, uma lista **"Não foi possível ler
(e por quê)"**.

Detalhes que importam:

- **SMT** é deduzido de lógicos > físicos — aritmética sobre dois números lidos,
  não chute. Se um faltar, `SmtKnown` fica `false`.
- **Mídia do disco**: `MSFT_PhysicalDisk.MediaType` 4 = SSD, 3 = HD, `BusType`
  17 = NVMe. Os valores **0 e 1 significam "não especificado"/"desconhecido" e
  NÃO viram SSD**. A letra da unidade é ligada ao disco físico por partição, em
  vez de supor "disco 0".
- **Chassi**: códigos SMBIOS de `Win32_SystemEnclosure`. Se falhar, a bateria
  serve de segundo indício — e o texto diz que foi deduzido.
- Cada leitura é isolada: uma falha vira nota, não derruba a análise.

O perfil é lido **na thread de fundo da análise**, junto do resto. WMI tem
timeout de 8 s por consulta; fazer isso a cada clique em "Detalhes" travaria a
janela.

---

## 5. Motor de compatibilidade — quatro estados, nunca dois

| Estado | Significado | Pode executar? |
|---|---|---|
| **Compatível** | requisitos conferidos neste PC | sim |
| **Não compatível** | requisito conferido e **não** atendido | não |
| **Não detectado** | o PC não contou o que precisa ser conferido | **não** |
| **Experimental** | compatível, mas sem medição que comprove ganho | sim, se escolhido |

"Não detectado" é o estado central. Ele **não** é "não compatível" e **não** é
"compatível". Aplicar às cegas é exatamente o que o pedido proíbe, então ele
bloqueia a execução automática — e diz o que faltou ler.

Exemplos que existem como teste:

- GPU não identificada + opção só para NVIDIA → **Não detectado**, nunca
  "compatível por otimismo".
- Mídia do disco não reconhecida + opção para SSD → **Não detectado**.
- Build não lida + opção que depende da versão → **Não detectado**.
- Experimental num PC que atende a tudo → continua **Experimental**. Compatível
  não significa recomendado.

### Requisitos não são inventados

A tentação de declarar requisito para parecer sofisticado é grande — e requisito
falso vira "Não compatível" falso. Por isso:

- só **um** requisito é escrito à mão (`native-power`: desktop + tomada, que
  `App.Compatibility` já exigia);
- as faixas de build vêm de onde **já estavam declaradas**: `ImportedOptions`
  (`Minimum`/`Maximum` por regra) e `V5Options` (`BuildRange`, exposto nesta
  fase). Copiar os números para uma segunda tabela criaria duas verdades que
  divergiriam no primeiro ajuste. Há teste conferindo que o contrato espelha a
  regra de origem.

---

## 6. Estágio, reinício e score

**Estágio** (feature flags): Estável · Beta · Experimental · Desenvolvedor.
Só **Estável** pode entrar em perfil automático — e há invariante recusando
cadastrar experimental ou não recomendada como automática.

**Reinício acumulado**: `RestartScope.For(item)` deduz o escopo do alvo real da
gravação (`HKLM\SYSTEM` → reiniciar; `Explorer` → reabrir o Explorer; resto →
nova sessão). `RestartScope.Aggregate` pede **uma vez**, pelo escopo mais forte,
em vez de uma vez por opção.

**Score**: `OptimizationRegistry.Score` conta quantas recomendações **aplicáveis
a este PC** já estão configuradas, e o texto sempre diz, por escrito, que **não
é benchmark nem nota de desempenho**. Teste garante a frase.

---

## 7. Invariantes (`OptimizationRegistry.Validate()`)

Roda na suíte e falha o build se qualquer um quebrar:

1. IDs não duplicam.
2. Toda otimização tem título.
3. **Reversível sem estratégia de backup é proibido.**
4. Toda otimização declara efeito **e** contrapartida.
5. Nível de evidência, confiança, estágio e escopo de reinício são valores válidos.
6. Tudo que não é mera conveniência tem fonte `https://`.
7. **Experimental ou não recomendada nunca é automática.**
8. Grupo do perfil Ultra sem opção elegível **não pode vir marcado**.
9. Nenhum tweak placebo conhecido é executável.

O invariante 9 merece destaque: `LargeSystemCache`, `DisablePagingExecutive`,
`IoPageLockLimit`, `TcpAckFrequency`, `TCPNoDelay`, `SvcHostSplitThresholdInKB`,
`HwSchMode`, `Win32PrioritySeparation` e `SystemResponsiveness` existem no
catálogo **apenas como arquivo catalogado para leitura**. Nenhum aparece nas
operações de uma opção executável. Antes isso era verdade por acaso; agora é
verdade por teste. `NetworkThrottlingIndex` é o único executável da lista — e o
teste exige que ele carregue o rótulo experimental.

---

## 8. Onde a fundação aparece para o usuário

Fundação que ninguém chama também é código morto. Foi ligada em três pontos:

1. **Ficha "Detalhes"** de cada opção agora vem de `OptimizationRegistry.Details`:
   o que muda, desvantagem, evidência, confiança, risco, estágio, **compatibilidade
   para este PC com o motivo**, administrador, reinício, como será restaurado,
   fonte e ID. Enquanto não houver análise, ela diz para clicar em Analisar este PC.
2. **Página Diagnóstico** mostra o perfil detectado, incluindo a lista do que
   **não** foi possível ler e por quê — e o score de recomendações.
3. **Grupos do Ultra** passaram a declarar a ausência em vez de prometer.

---

## 9. Verificação executada

```
mcs @linux.rsp -target:winexe            → OK
mcs @linux.rsp -define:TESTS ... EngineTests
mono LukeOptimizer-Testes.exe            → 1.114 verificações, 6 puladas
xvfb-run mono UiSmoke.exe                → suíte de interface completa, 1 bloco pulado
```

**+95 verificações** nesta fase (1.019 → 1.114). Todos os invariantes passaram
contra o catálogo real na primeira execução.

Os 6 pulados e o bloco pulado da interface são premissas exclusivas do Windows
(trava obrigatória de arquivo, caminhos com letra de unidade, `psapi.dll`, job
object, `FILE_DISPOSITION_INFO`, `PostMessage`), cada um marcado com o motivo.
Ver `RELATORIO-TESTES-EXECUTADOS.md`.

**Não testado em Windows real**: elevação UAC, DPI de monitor, gravação em HKLM
sob política de domínio, e as leituras WMI novas do `MachineProfile`
(`MSFT_PhysicalDisk`, `Win32_SystemEnclosure`, `Win32_Tpm`, SecureBoot). Elas
são tolerantes a falha por construção — cada uma vira nota em vez de exceção —
mas **os valores que retornam num PC real ainda não foram conferidos por mim**.

---

## 10. Próxima fase

**FASE 3 — UX**, que é o que transforma a fundação em algo visível: dashboard
com o estado do PC, busca global por tags, filtros por risco/evidência/estágio,
prévia universal antes de aplicar um perfil (`Recurso | Atual | Novo | Risco |
Rollback | Reinício`) e separação entre modo básico e avançado.

A fundação já entrega os dados de que a FASE 3 precisa: cada descritor tem tags,
risco, evidência, estágio, compatibilidade e escopo de reinício.
