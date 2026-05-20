<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Mistérios Encontrados — SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **mysteries-found**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Registre aqui toda lógica, comportamento ou código que o time não conseguiu explicar.
> "Mistérios" são trechos de código sem documentação, com lógica não-óbvia ou que parecem workarounds.
>
> **Cota mínima para passar pelo portão do Estágio 2:** 5 mistérios documentados.

## O que conta como "mistério"?

- Código que faz algo inesperado sem comentário explicando por quê
- Valores hardcoded sem explicação (números mágicos)
- Lógica condicional que parece um workaround ou gambiarra
- Campos no DDM que não são usados por nenhum programa
- Programas que existem mas não são chamados por ninguém
- Comportamento diferente entre o que a documentação diz e o que o código faz
- Easter eggs deixados pelos desenvolvedores originais

## Níveis de Confiança

| Nível     | Significado                                         |
| --------- | --------------------------------------------------- |
| **ALTA**  | Temos certeza de que há algo estranho aqui          |
| **MÉDIA** | Parece suspeito, mas pode ter explicação            |
| **BAIXA** | Pode ser intencional, mas não conseguimos confirmar |

## Mistérios Catalogados

| ID      | Descrição                                                                                            | Onde Encontrado                                                       | Impacto Potencial                                                      | Confiança |
| ------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------- | --------- |
| MYS-001 | BATCHREL **arredonda** valor bruto (`+ 0.005`) mas BATCHPGT **trunca** o mesmo valor                 | `BATCHREL.NSN#L118-L121` vs `BATCHPGT.NSN#L251-L253`                  | Totais do relatório mensal divergem do somatório real dos pagamentos   | ALTA      |
| MYS-002 | Tabela de fator regional declarada com 27 posições, mas validação de índice usa `1..25`              | `BATCHPGT.NSN#L116-L143, L200-L204`                                   | Slots 26 e 27 mortos — possível indício de UFs/regiões removidas       | ALTA      |
| MYS-003 | Cálculo de idade usa apenas diferença de anos (`ano - ano_nasc`), ignorando mês e dia                | `BATCHPGT.NSN#L228`                                                   | Beneficiário recebe fator de idoso (1.15) até 12 meses antes           | ALTA      |
| MYS-004 | Bloco completo de integração Banco Real (cod 356) comentado desde 2007                               | `BATCHCON.NSN#L190-L210`                                              | Dead code preservado por razão desconhecida; layout citado diferente   | ALTA      |
| MYS-005 | `COD-BANCO` é gravado hardcoded como `1` na conciliação, embora o sistema tenha histórico multi-banco | `BATCHCON.NSN#L165`                                                   | Multi-banco impossível sem mudar código apesar de campo existir        | ALTA      |
| MYS-006 | Códigos de retorno bancário diferentes de `00/01/02` apenas geram WRITE em log; status fica intacto  | `BATCHCON.NSN#L183-L186` (cláusula `NONE` do DECIDE)                  | Pagamentos com erro bancário desconhecido ficam "pendurados" em `G`    | ALTA      |
| MYS-007 | Status de pagamento desconhecido em BATCHREL é silenciosamente classificado como "Gerado"            | `BATCHREL.NSN#L143-L144` (`NONE MOVE 1 TO #IDX-STS`)                  | Mascara dados corrompidos no relatório consolidado                     | MÉDIA     |
| MYS-008 | Tolerância de divergência de R$ 0,01 sem comentário explicando origem (regra contábil? Histórico?)   | `BATCHCON.NSN#L150` (`IF #DIFF > 0.01`)                               | Mudar esse limiar afeta a métrica "% conciliado" reportada à gestão    | MÉDIA     |
| MYS-009 | Abono dezembrino de 15% hardcoded; nenhum parâmetro em `PROGRAMA-SOCIAL`                             | `BATCHPGT.NSN#L268` (`#VLR-BENF * 0.15`)                              | Reajuste do abono exige mudança de código + deploy                     | MÉDIA     |
| MYS-010 | Variáveis declaradas mas nunca usadas: `#FOUND`, `#I`, `#LOG-WORK` em BATCHPGT                       | `BATCHPGT.NSN` DEFINE DATA                                            | Indício de funcionalidade prevista (log de erros?) nunca implementada  | BAIXA     |
| MYS-011 | Em dezembro, `CALCBENF` paga 13º salário e ainda aplica abono de 15% para programas tipo `A`          | `CALCBENF.NSN#L226-L247`                                              | Acúmulo intencional ou bug? Programa duplo (13º + abono) é prática SIFAP? | ALTA      |
| MYS-012 | Truncagem sistemática via `*100/100` para 2 casas decimais em CALCBENF/CALCDSCT/CALCCORR (em vez de ROUND) | múltiplos programas CALC*.NSN                                     | Padrão herdado do mainframe ou erro? Diverge de BATCHREL (round)       | ALTA      |
| MYS-013 | Desconto judicial NÃO respeita teto de 30%; pode zerar o líquido                                     | `CALCDSCT.NSN#L139-L144`                                              | Conformidade legal vs ordem judicial — qual prevalece?                 | ALTA      |
| MYS-014 | `IF #IDADE > 75 MOVE 'S' TO #STATUS` — suspende beneficiário recém-cadastrado silenciosamente         | `CADBENEF.NSN#L165-L174`                                              | Por que 75? Política demográfica não documentada                       | ALTA      |
| MYS-015 | Limite hardcoded de 5 dependentes em CADDEPEND contradiz DDM que permite 10 posições                 | `CADDEPEND.NSN#L59-L62`                                               | Programa só usa metade do PE? Outro programa preenche o resto?         | ALTA      |
| MYS-016 | Constante mágica `0.347215` no fator de reajuste em CADPROG sem comentário/origem                    | `CADPROG.NSN#L75-L78`                                                 | Coeficiente atuarial? Inflação histórica?                              | ALTA      |
| MYS-017 | Comentário explícito "INCONSISTENCIA CONHECIDA - NAO CORRIGIR SEM APROVACAO DA AUDITORIA" em CONSBENF | `CONSBENF.NSN#L168-L189`                                              | Por que auditoria proíbe correção? Algum sistema externo depende do bug? | ALTA    |
| MYS-018 | `IF AUDITORIA-V.ACAO='EX' ESCAPE TOP` — exclusões somem do relatório de auditoria                    | `RELAUDIT.NSN#L98-L103`                                               | Compliance: trilha de auditoria deveria mostrar tudo. Quem decidiu ocultar? | ALTA  |
| MYS-019 | CPF de 11 dígitos iguais é inválido EXCETO se prefixo for `000` (comentário: "TESTE GOVERNO")        | `VALBENEF.NSN#L218-L234`                                              | Backdoor para CPFs de teste? Ainda em uso em produção?                 | ALTA      |
| MYS-020 | Tabela `DIAS-MES(2)=29` fixa em VALBENEF; ignora regra real de ano bissexto (4/100/400)              | `VALBENEF.NSN#L91, L257-L274`                                         | Permite cadastrar 29/02 em ano não bissexto. Bug conhecido?            | ALTA      |
| MYS-021 | 8 prefixos CPF (`000, 001, 002, 010, 011, 099, 100, 999`) passam direto sem checagem de dígito       | `VALDOCS.NSN#L36-L43, L171-L188`                                      | Backdoor amplo de teste — quantos beneficiários reais usam esses prefixos? | ALTA  |
| MYS-022 | `IF #COD-REG=99 → ELEGIVEL=TRUE; ESCAPE ROUTINE` pula TODAS as validações de elegibilidade           | `VALELEG.NSN#L91-L96`                                                 | Diplomatas? Convênios internacionais? Ou backdoor?                     | ALTA      |
| MYS-023 | Bloco "Plano Verão 1989" comentado em CALCCORR — inativo há 25+ anos mas preservado no fonte         | `CALCCORR.NSN#L62-L72`                                                | Easter egg histórico ou possível reativação?                           | BAIXA     |
| MYS-024 | Duas implementações de desconto: CALCBENF aplica desconto inline; CALCDSCT calcula formalmente       | `CALCBENF.NSN#L307-L315` vs `CALCDSCT.NSN`                            | Qual é o "fonte de verdade"? Há divergência de valor entre eles?       | ALTA      |

## Detalhamento dos Mistérios

### MYS-001: Round vs Truncate — relatório não bate com pagamento real

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L118-L121` e `BATCHPGT.NSN#L251-L253`
- **Trecho de código**:

```natural
* BATCHREL — arredonda
COMPUTE #VLR-ARR = PAGAMENTO-V.VLR-BRUTO + 0.005
COMPUTE #VLR-TEMP = #VLR-ARR * 100
COMPUTE #VLR-ARR = #VLR-TEMP / 100

* BATCHPGT — trunca
COMPUTE #VLR-TEMP = #VLR-BENF * 100
COMPUTE #VLR-BENF = #VLR-TEMP / 100
```

- **O que esperávamos**: somatório do relatório = soma dos campos `VLR-BRUTO` gravados.
- **O que o código faz**: relatório soma valores arredondados; pagamento gravou valores truncados. Diferença acumula.
- **Hipótese do time**: o alterador de 2006 (Roberto Mendes) introduziu arredondamento "para subtotais ficarem mais bonitos" sem perceber a divergência.
- **Risco se ignorarmos**: reconciliação contábil falha em produção; auditoria identifica diferença de centavos em milhares de pagamentos.

---

### MYS-002: Tabela regional com 27 slots, mas só 25 acessíveis

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L116-L143` (declaração) e `L200-L204` (uso)
- **Trecho de código**:

```natural
MOVE 1.0000 TO #TAB-REG(26)
MOVE 1.0000 TO #TAB-REG(27)
...
IF #COD-REG >= 1 AND #COD-REG <= 25
  MOVE #TAB-REG(#COD-REG) TO #FATOR-REG
ELSE
  MOVE 1.0000 TO #FATOR-REG
END-IF
```

- **O que esperávamos**: tabela do tamanho exato das regiões válidas.
- **O que o código faz**: slots 26-27 inicializados mas inalcançáveis (else neutraliza tudo > 25).
- **Hipótese do time**: regiões 26-27 foram desativadas (talvez DF + território?), mantidas para não renumerar.
- **Risco se ignorarmos**: na migração, replicar tabela "como está" perpetua código morto; remover sem investigar pode quebrar caso de borda histórico.

---

### MYS-003: Idade calculada só por ano

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L228`
- **Trecho de código**:

```natural
COMPUTE #ANO-NASC = BENEFICIARIO-V.DT-NASCIMENTO / 10000
COMPUTE #IDADE = #ANO - #ANO-NASC
```

- **O que esperávamos**: idade exata (com mês/dia).
- **O que o código faz**: beneficiário nascido em 31/12/1960, processado em janeiro/2025, já é tratado como tendo 65 anos.
- **Hipótese do time**: "boa fé pró-beneficiário" deliberada — sempre antecipa o fator de idoso.
- **Risco se ignorarmos**: spec moderna corrige isso e milhões de beneficiários perdem ~R$ X por 1 mês. **Validar com PO antes de "consertar".**

---

### MYS-004: Integração Banco Real preservada como dead code há 18 anos

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L190-L210`
- **Trecho de código**: bloco inteiro comentado com `*`, incluindo `DEFINE WORK FILE 2 'RETORNO_REAL.DAT'` e header `BANCO REAL FOI ADQUIRIDO PELO SANTANDER EM 2007 / MANTER CODIGO PARA REFERENCIA HISTORICA`.
- **O que esperávamos**: código removido após aquisição pelo Santander.
- **O que o código faz**: nada (está comentado), mas ocupa espaço e gera dúvida sobre multi-banco.
- **Hipótese do time**: medo de remover por "se precisar voltar" — clássico legado.
- **Risco se ignorarmos**: na modernização, replicar isso é desperdício. **Não migrar.** Documentar decisão em ADR.

---

### MYS-005: COD-BANCO hardcoded em 1

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L165`
- **Trecho de código**:

```natural
MOVE 1 TO PAGAMENTO-V.COD-BANCO
```

- **O que esperávamos**: campo preenchido com o banco efetivo do retorno.
- **O que o código faz**: força BB, embora o campo `#CNAB-BANCO` tenha sido lido do registro.
- **Hipótese do time**: como Banco Real foi descontinuado, "todo pagamento é BB" virou invariante de fato.
- **Risco se ignorarmos**: spec moderna deve usar o código real do banco (multi-banco real) — confirmar com PO.

---

### MYS-006: Códigos de retorno bancário desconhecidos não atualizam status

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L183-L186`
- **Trecho de código**:

```natural
NONE
  COMPRESS 'COD RETORNO DESCONHECIDO:' #COD-RET
      ' CPF=' #CNAB-CPF INTO #MSG
  WRITE #MSG
```

- **O que esperávamos**: status do pagamento atualizado para algo como "erro" ou "manual".
- **O que o código faz**: apenas escreve no log e segue. Pagamento fica em status `G` (Gerado) eternamente.
- **Hipótese do time**: alguém iria processar o log manualmente. Provavelmente ninguém faz.
- **Risco se ignorarmos**: dívida operacional silenciosa — pagamentos "fantasma" no sistema. Investigar quantos existem hoje.

---

### MYS-007 / MYS-008 / MYS-009 / MYS-010

> Detalhamento pendente — Par 2 vai expandir antes de H1 caso o tempo permita. Linha na tabela acima é suficiente para o gate.

---

## Easter Eggs

> Dica: existem **3 easter eggs** escondidos no código legado. Registre aqui os que encontrar:

1. [x] **EGG-001 · Plano Verão 1989 preservado como dead code** — bloco comentado em `CALCCORR.NSN#L62-L72` referenciando o plano econômico de janeiro/1989. Mantido por 25+ anos por superstição ou medo de remover.
2. [x] **EGG-002 · Backdoor de CPFs especiais** — combinando `VALDOCS` (`#L36-L43`) e `VALBENEF` (`#L218-L234`):
   - Em `VALDOCS`: prefixos `000, 001, 002, 010, 011, 099, 100, 999` aceitos sem validação de dígito.
   - Em `VALBENEF`: CPF com 11 dígitos iguais é aceito quando começa com `000`.
   - Comentário no fonte: `* CPFS INICIADOS COM 000 SAO VALIDOS (TESTE GOVERNO)`.
   - **Impacto**: cadastro pode aceitar CPFs sintéticos em produção. Risco alto de fraude e divergência fiscal.
3. [ ] **EGG-003** — ainda não localizado; varrer programas restantes / `legacy-docs/`.

## Resumo

- Total de mistérios encontrados: **24**
- Confiança alta: **20**
- Confiança média: **3**
- Confiança baixa: **1**
- Easter eggs encontrados: **2 / 3** (EGG-001 Plano Verão em CALCCORR; EGG-002 Backdoor CPF em VALDOCS+VALBENEF)

---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="mysteries-checklist.md"><strong>mysteries-checklist.md</strong></a><br/>
<sub>Lista do que procurar.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="discovery-report.md"><strong>discovery-report.md</strong></a><br/>
<sub>Síntese final.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

