# FASE 3 — Nomes para humanos

A regra do pedido: **o usuário nunca vê nome de arquivo, chave de Registro ou
comando como nome de uma função.** Detalhe técnico continua disponível, em
Detalhes.

Antes de aplicar a regra, medi o tamanho do problema. O resultado mudou a
resposta, então começo por ele.

---

## 1. O que a medição mostrou

Rodei uma sonda sobre o catálogo que o programa monta em memória, procurando
`.reg`, `.bat`, `.cmd`, `.ps1`, `.exe`, `HKLM`, `HKCU`, `reg add`, `netsh`,
`ipconfig`, `powercfg`, `DISM` nos nomes exibidos:

| | Resultado |
|---|---|
| Entradas com nome de arquivo no título | **307 de 673** |
| — contendo `.bat` | 173 |
| — contendo `.reg` | 70 |
| — contendo `.exe` | 63 |
| **Funções EXECUTÁVEIS com nome ruim** | **0 de 127** |

Essa última linha é o ponto.

**Nenhuma das 127 otimizações que o LukeOptimizer realmente aplica tem nome
técnico.** Elas já se chamam "Desativar aceleração do ponteiro", "Windows 11:
alinhar barra à esquerda", "Explorer: mostrar extensões de arquivos".

As 307 com nome de arquivo são **outra coisa**: são as 546 entradas de
inventário — os arquivos que você mesmo enviou (`.bat`, `.reg`) e instaladores
de terceiros que estavam no pacote:

```
Optimizer-16.7.exe          netlimiter-5.3.26.0.exe     ISLC v1.0.3.7.exe
memreduct-3.5.2-setup.exe   Firemin_8520_Setup.exe      processlassosetup64.exe
RazerCortexInstaller.exe     Dism++10.1.1002.1B.zip      Autoruns.exe
NET200%_OTIMIZADA(1).BAT     DEBLOAT(1).BAT              LIMPA_RAM(1).BAT
```

O LukeOptimizer **não executa nenhum deles**. Ele leu o conteúdo, explicou o que
fazem e os deixou como referência.

### Por que eu não renomeei esses 307

Porque para inventário **o nome do arquivo é a identidade**.

Se eu rebatizar `Optimizer-16.7.exe` como "Otimizador de terceiros", você perde
a capacidade de achar esse arquivo na sua própria pasta, comparar com o que o
programa diz sobre ele, ou decidir apagá-lo. Trocar o nome não deixaria a
interface mais honesta — deixaria menos.

O problema real não era o nome do arquivo aparecer. Era ele aparecer **no mesmo
lugar e com a mesma aparência de uma função**, sem distinção. Isso é o que o
item 182 do pedido está apontando quando proíbe uma aba "BATs".

---

## 2. Onde o vazamento realmente estava: as categorias

O seletor de categorias da tela Catálogo mostrava as 35 categorias **cruas**:

```
2 - .Bats                              99 - Seus 14 BATs revisados
3 - Regedits                           11 - Tweaks Avançados (Modo Agressivo)
5 - Apps essenciais                    -4 - Avançado: perde recursos
10 - Otimizar Driver de Vídeo (AMD & NVIDIA)
```

Prefixos numéricos, tipos de arquivo como nome de seção, e **quatro pares
duplicados** sob prefixos diferentes: `-3 - Personalização` e
`-4 - Personalização` eram a mesma coisa listada duas vezes, o mesmo para
`Windows leve`, `Privacidade opcional` e `Rede e downloads`.

### Depois

**35 categorias cruas → 24 na tela**, com nome limpo:

| Na tela | Cobre |
|---|---|
| Jogos | 3 categorias brutas |
| Rede | 3 |
| Personalizar Windows | 3 |
| **Arquivos que você enviou** | 3 (`.Bats`, `Regedits`, `Seus 14 BATs revisados`) |
| Windows leve · Privacidade · Reparar Windows | 2 cada |
| Programas de terceiros · Ferramentas do pacote · Energia · GPU e drivers · Processos · Mouse e teclado · Navegadores · Remover aplicativos · Diagnóstico · Backup e restauração · Reverter alterações · Laboratório experimental · Guias e referência · … | 1 cada |

O inventário agora se chama **"Arquivos que você enviou"** e **"Programas de
terceiros"** — honesto sobre o que é, sem fingir que é função e sem apagar o
nome do arquivo lá dentro.

A ordem também mudou: funções primeiro, inventário e referência por último.

---

## 3. O que garante que isso não volte

`tests/NamingTests.cs` — **254 verificações novas**:

| Invariante | O que impede |
|---|---|
| `FeatureNaming.Violation()` rejeita 14 formas de nome técnico | a regra virar parágrafo esquecido |
| Toda uma das **127 executáveis** passa pela regra | um cadastro novo entrar com nome de arquivo |
| Detalhes continua citando fonte técnica e ID | "simplificar" virar "esconder" |
| Nenhuma categoria tem extensão, chave ou prefixo numérico | `.Bats` voltar ao seletor |
| Categorias duplicadas colapsam em uma | a tela listar a mesma coisa duas vezes |
| **Toda** categoria bruta cai em algum grupo | um item sumir da tela por falta de tradução |
| Nenhum item de inventário entra em perfil automático | arquivo de terceiro ser aplicado por engano |
| Inventário **mantém** o nome real do arquivo | a "limpeza de nomes" destruir a rastreabilidade |

---

## 4. Nenhum ID foi alterado

Como manda o item 4 do pedido. Perfis, histórico, backup e desfazer gravados no
seu PC referenciam IDs; renomear identidade para arrumar texto quebraria a
restauração de backups antigos.

`FeatureNaming` é camada de **apresentação**: mapeia categoria bruta → nome
limpo, guarda as categorias brutas no filtro, e não encosta em `Item.Id`.

### Bug corrigido de quebra

A ordenação do catálogo fazia `Int32.Parse(categoria.Split(' ')[0])`. Uma
categoria nova sem prefixo numérico derrubaria a lista inteira com
`FormatException`. Agora a ordem vem de `FeatureNaming.Rank`.

---

## 5. Verificação

```
Engine:     1368 verificações, 6 puladas   (era 1114)
Interface:  suíte completa, 1 bloco pulado
Executável: compila
```

**Não testado em Windows real:** a aparência final do seletor de categorias.
A lógica de agrupamento é testada; a renderização do ComboBox no Windows, não.

---

## 6. O que deste pedido NÃO foi feito

O pedido tem 220 itens. Esta fase cobre os itens 1–4, 134–135, 140–142 e
181–184 (regra de nomes, unificação de categorias, separação inventário/função,
preservação de IDs, testes de nomenclatura).

Continua pendente, por ordem do que eu faria em seguida:

| Item | O que é |
|---|---|
| 15, 170, 179, 180 | Dashboard com cards, sidebar de 20 seções, badges |
| 16–26 | Central de rede: LIMPAR IP / INTERNET, DNS, testes |
| 27–41 | Central de limpeza com analisador |
| 42–45 | Reparar Windows (SFC/DISM pelo ExecutionHub) |
| 54–65 | Energia, hibernação, inicialização rápida, CompactOS |
| 94–111 | Personalização (Taskbar, Start, Explorer, aparência) |
| 130–131 | Busca global e Command Palette (Ctrl+K) |
| 145 | Assistente de primeira execução |
| 146 | Os 11 guias de documentação |
| 163 | Laboratório experimental (a categoria já existe; falta a tela) |

Isso não cabe numa rodada. A fundação da FASE 2 (contrato, evidência,
compatibilidade) e a camada de nomes desta fase são justamente o que permite
adicionar cada um deles sem tocar em dezenas de arquivos.
