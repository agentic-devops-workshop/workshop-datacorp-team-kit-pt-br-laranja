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

| ID      | Descrição                                                                          | Onde Encontrado                                | Impacto Potencial                                              | Confiança |
| ------- | ---------------------------------------------------------------------------------- | ---------------------------------------------- | -------------------------------------------------------------- | --------- |
| MYS-002 | Tabela `#TAB-REG` tem 27 posições mas `IF` só permite índices 1–25                 | `CALCBENF.NSN` L100 + L162–166                 | Migração pode tratar 27 regiões; código real ignora 26 e 27    | ALTA      |
| MYS-003 | Constante mágica `#TAB-REG(15) = 1.0000` rotulada apenas como `/* REF */`          | `CALCBENF.NSN` L121                            | Significado de "REF" perdido; provável fator de referência (SP/Sudeste). Confundir = recalibrar todo o País | MÉDIA |
| MYS-004 | Em dezembro (`#MES = 12`) o cálculo muda: soma 13º + abono natalino 15% (tipo 'A') | `CALCBENF.NSN` L226–247                        | Se não preservado, beneficiários perdem 13º e abono em dezembro | ALTA      |
| MYS-005 | Truncagem sistemática via `*100 / 100` (sem `ROUND`) — perda de centavos          | `CALCBENF.NSN` L213, L233, L242, L262; `CALCDSCT.NSN` L62, L177; `CALCCORR.NSN` L113 | Implementação moderna com `ROUND` ou `BigDecimal` HALF_UP causa divergência sistemática vs. legado | ALTA |
| MYS-006 | Desconto `J` (Judicial) ignora teto de 30% que se aplica a todos os demais tipos  | `CALCDSCT.NSN` L139–144 (comentário "JUDICIAL NAO TEM TETO") | Ordem judicial pode levar líquido a zero ou negativo; migração ingênua aplicaria teto e violaria decisão judicial | ALTA |
| INC-004 | Cálculo de descontos coexiste em duas versões divergentes                          | `CALCBENF.NSN` L307–315 (subrotina simples 3%) vs. `CALCDSCT.NSN` (lógica completa) | Qual é a verdade? Depende do caminho de invocação. Pagamentos podem ser gerados com desconto errado | ALTA |
| EGG-001 | Bloco comentado mantido "para histórico" — correção do Plano Verão (1989–1991)    | `CALCCORR.NSN` L62–72                          | Código morto; pode ser removido. Sinaliza que houve correções monetárias compostas no Cruzado→Cruzeiro | ALTA |

## Detalhamento dos Mistérios

### MYS-002: Tabela regional com slots fantasmas (26 e 27 "RESERVA")

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L100-L166`
- **Trecho de código**:

```natural
1 #TAB-REG  (N3.4/27)
...
MOVE 1.0000 TO #TAB-REG(26)  /* RESERVA */
MOVE 1.0000 TO #TAB-REG(27)  /* RESERVA */
...
IF #COD-REG >= 1 AND #COD-REG <= 25
  MOVE #TAB-REG(#COD-REG) TO #FATOR-REG
ELSE
  MOVE 1.0000 TO #FATOR-REG
END-IF
```

- **O que esperávamos**: tabela com 27 posições significa 27 regiões válidas.
- **O que o código faz**: dimensiona 27 mas só aceita 1–25. Índices 26 e 27 são código morto. Qualquer `COD-REG` fora desse range cai no `ELSE` e recebe fator 1.0000 (regra silenciosa).
- **Hipótese do time**: alguém previu expansão futura ("RESERVA") que nunca aconteceu; o `IF` ficou amarrado em 25.
- **Risco se ignorarmos**: na migração, manter `27` no schema enquanto o domínio real é `25` perpetua confusão. Pior: o fallback `1.0000` mascara dados ruins (UF inválida) sem alertar.

---

### MYS-003: Constante "REF" — o que é a Região 15?

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L121`
- **Trecho de código**:

```natural
MOVE 1.0000 TO #TAB-REG(15)  /* REF */
```

- **O que esperávamos**: cada índice 1–25 mapeia uma UF (ex.: 11=SP, 12=RJ…).
- **O que o código faz**: índice 15 está rotulado apenas como "REF" — todas as outras 24 posições têm sigla de UF. Não há documentação em `legacy-docs/` explicando.
- **Hipótese do time**: "REF" = REFERÊNCIA — fator-base 1.0000 usado para normalizar os demais (provavelmente sigla DF ou um pseudo-estado). Pode ser também marcador para beneficiários sem UF cadastrada.
- **Risco se ignorarmos**: confundir "REF" com uma UF real causa erro de cálculo silencioso. Modelagem moderna precisa decidir se vira `enum` separado, `null` ou registro especial.

---

### MYS-004: Dezembro muda tudo — 13º salário e abono natalino

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L226-L247`
- **Trecho de código**:

```natural
IF #MES = 12
  MOVE 'D' TO #TIPO-PGTO
  COMPUTE #VLR-13 = #VLR-BASE * #FATOR-REG * #FATOR-IDADE
  ...
  COMPUTE #VLR-BRUTO = #VLR-BENF + #VLR-13
*
* ABONO NATALINO - 15% ADICIONAL PARA PROGRAMAS TIPO 'A'
  IF #TIPO-PROG = 'A'
    COMPUTE #VLR-ABONO = #VLR-BENF * 0.15
    ...
    COMPUTE #VLR-BRUTO = #VLR-BRUTO + #VLR-ABONO
  END-IF
END-IF
```

- **O que esperávamos**: cálculo uniforme nos 12 meses.
- **O que o código faz**: em dezembro adiciona um 13º calculado com fórmula **diferente** (sem `FATOR-FAM` nem `FATOR-RND` nem reajuste) e, se o programa for tipo 'A', soma 15% de abono natalino sobre o benefício mensal.
- **Hipótese do time**: regra trazida por "ALTERADO 30/11/2001 - INC 13O SALARIO" e "ALTERADO 22/12/2009 - ABONO NATALINO". O 13º propositalmente ignora família/renda — equivale a "salário-base regional ajustado por idade".
- **Risco se ignorarmos**: beneficiários perdem 13º e abono em dezembro → impacto financeiro direto. Specs EARS PRECISAM ter um REQ separado para dezembro.

---

### MYS-005: Truncagem sistemática causa perda de centavos

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L213-L262` (e múltiplos outros pontos)
- **Trecho de código** (padrão repetido):

```natural
* TRUNCAR P/ 2 CASAS DECIMAIS - PADRAO MAINFRAME
COMPUTE #VLR-TEMP = #VLR-BENF * 100
COMPUTE #VLR-BENF = #VLR-TEMP / 100
```

Ocorrências adicionais:
- `CALCBENF.NSN` L213, L233, L242, L262
- `CALCDSCT.NSN` L62-65, L177-178
- `CALCCORR.NSN` L113-114

- **O que esperávamos**: arredondamento HALF_UP (banker's rounding) como em sistemas financeiros modernos.
- **O que o código faz**: `#VLR-TEMP` é `N11` (inteiro) — multiplica por 100 e divide por 100 **truncando** a parte fracionária além da segunda casa. R$ 123,4567 vira R$ 123,45 (sempre arredonda para baixo).
- **Hipótese do time**: convenção mainframe ("PADRAO MAINFRAME" no comentário). Acumulado em milhões de pagamentos representa receita "perdida" para o beneficiário.
- **Risco se ignorarmos**: usar `BigDecimal.setScale(2, HALF_UP)` em Java diverge do legado. Testes de equivalência vão falhar em centavos. Decisão necessária: replicar `RoundingMode.DOWN` ou modernizar (e documentar o ganho do beneficiário em ADR).

---

### MYS-006: Desconto Judicial sem teto — exceção silenciosa

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L121-L158`
- **Trecho de código**:

```natural
VALUE 'J'
* DESCONTO JUDICIAL - VALOR FIXO OU PERCENTUAL
  ...
* JUDICIAL NAO TEM TETO
  ADD #VLR-DSCT-ITEM TO #VLR-TOTAL-DSCT
...
* APLICAR TETO 30% - EXCETO JUDICIAL
IF #TIPO-DSCT NE 'J'
  IF #VLR-TOTAL-DSCT > #VLR-MAX-DSCT
    MOVE #VLR-MAX-DSCT TO #VLR-TOTAL-DSCT
  END-IF
END-IF
```

- **O que esperávamos**: teto de 30% do bruto se aplica a todos os tipos de desconto.
- **O que o código faz**: tipo `J` (Judicial) é deliberadamente excluído do teto. Líquido pode ser zero ou negativo.
- **Hipótese do time**: ordens judiciais (penhora, dívida tributária) têm precedência legal — o legislador/jurisdição obriga retenção integral, mesmo que zere o benefício.
- **Risco se ignorarmos**: aplicar teto 30% em desconto judicial = descumprimento de ordem judicial → responsabilidade legal do órgão. EARS REQ deve explicitar a exceção.

---

### INC-004: Duas implementações divergentes de cálculo de desconto

- **Arquivo A**: `CALCBENF.NSN#L307-L315` (subrotina interna `CALC-DESCONTOS`)
- **Arquivo B**: `CALCDSCT.NSN` (programa completo)

- **Trecho de código A** (CALCBENF — simplificado):

```natural
DEFINE SUBROUTINE CALC-DESCONTOS
  MOVE 0 TO #VLR-DESC
* DESCONTO BASICO - 3% CONTRIB SOCIAL
  IF #VLR-BRUTO > 500.00
    COMPUTE #VLR-DESC = #VLR-BRUTO * 0.03
    ...
  END-IF
END-SUBROUTINE
```

- **O que esperávamos**: um único algoritmo de desconto, reutilizado.
- **O que o código faz**: `CALCBENF` aplica uma versão simplificada hardcoded (apenas 3% fixo acima de R$ 500), enquanto `CALCDSCT` aplica a tabela completa de 4 faixas + PE de descontos personalizados + tipos J/P/I/S/A.
- **Hipótese do time**: `CALCBENF` chama internamente sua versão "simplificada" só para gerar o `VLR-LIQUIDO` inicial; depois `CALCDSCT` é executado em batch e sobrescreve. Mas se `CALCDSCT` não rodar, fica o cálculo errado.
- **Risco se ignorarmos**: na migração, replicar só uma das duas lógicas gera pagamentos com valores divergentes do legado. Pior: a sequência de execução (online → batch) precisa ser preservada ou unificada.

---

## Easter Eggs

> Dica: existem **3 easter eggs** escondidos no código legado. Registre aqui os que encontrar:

1. [x] **EGG-001 — Plano Verão (1989–1991):** bloco comentado em `CALCCORR.NSN#L62-L72` mantém código de correção monetária da transição Cruzado→Cruzeiro, com fator `2.7500` e ajuste adicional `1.4289` para competências antes de jul/1989. Comentário: "NAO REMOVER (HISTORICO) - RESPONSAVEL: JOAO BATISTA - 15/03/2003".

```natural
* CORRECAO PLANO VERAO - PERIODO 01/1989 A 01/1991
* UTILIZADO DURANTE TRANSICAO MOEDA CRUZADO->CRUZEIRO
*  IF #COMP-INI >= 198901 AND #COMP-INI <= 199101
*    COMPUTE #IND-ACUM = #IND-ACUM * 2.7500
*    IF #COMP-INI < 198907
*      COMPUTE #IND-ACUM = #IND-ACUM * 1.4289
*    END-IF
*    MOVE 'V' TO PAGAMENTO-V.IND-CORRIGIDO
*  END-IF
```

2. [ ] Easter Egg 2: _(buscar nos demais programas — VAL*, BATCH*, CAD*, CONS*, REL*)_
3. [ ] Easter Egg 3: _(buscar nos demais programas)_

## Resumo

- Total de mistérios encontrados: **7** (6 mistérios + 1 inconsistência catalogada)
- Confiança alta: **6** (MYS-002, MYS-004, MYS-005, MYS-006, INC-004, EGG-001)
- Confiança média: **1** (MYS-003)
- Confiança baixa: 0
- Easter eggs encontrados: **1** / 3

> Mistérios ainda em aberto (não visíveis nos 3 programas CALC* — investigar nos demais):
> MYS-001 (alteração silenciosa de status), MYS-007 (CPFs sem validação), MYS-008 (região que pula elegibilidade), MYS-009 (ordem batch ilógica), MYS-010 (evento de auditoria ocultado), EGG-002 (backdoor de validação), EGG-003 (integração com empresa morta).

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

