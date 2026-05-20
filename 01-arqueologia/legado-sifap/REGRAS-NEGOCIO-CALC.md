# Regras de Negócio Extraídas — Programas CALC*

Data: 20/05/2026  
Fonte: `CALCDSCT.NSN`, `CALCCORR.NSN`, `CALCBENF.NSN`  
Escopo: Cálculos de pagamentos, descontos, correção retroativa e benefícios

---

## 1. CALCDSCT.NSN — Cálculo de Descontos

### Regra RND-1: Tipos de Desconto e Suas Fórmulas

Cada tipo de desconto tem lógica de cálculo distinta:

| Tipo | Código | Fórmula | Tem Teto? | Nota |
|------|--------|---------|-----------|------|
| **Judicial** | `J` | Fixo OU Percentual (campo decide) | **NÃO** | Sem limite 30% |
| **Pensão Alimentícia** | `P` | Fixo OU Percentual (campo decide) | **SIM** | Teto 30% |
| **Imposto Retido** | `I` | **Sempre Percentual** | **SIM** | Teto 30% |
| **Desconto Sindical** | `S` | **Fixo 1% do bruto** | **SIM** | Teto 30% |
| **Administrativo** | `A` | Fixo OU Percentual (campo decide) | **SIM** | Teto 30% |

**Código:** [CALCDSCT.NSN](CALCDSCT.NSN#L80-L120)

```natural
DECIDE ON FIRST VALUE OF #TIPO-DSCT
  VALUE 'J'  /* Judicial: valor OU percentual */
    IF BENEFICIARIO-V.VLR-DSCT > 0
      MOVE BENEFICIARIO-V.VLR-DSCT TO #VLR-DSCT-ITEM
    ELSE
      COMPUTE #VLR-DSCT-ITEM = #VLR-BRUTO * (BENEFICIARIO-V.PCT-DSCT / 100)
    END-IF
    ADD #VLR-DSCT-ITEM TO #VLR-TOTAL-DSCT  /* SEM TETO */
  VALUE 'S'  /* Sindical: sempre 1% */
    COMPUTE #VLR-DSCT-ITEM = #VLR-BRUTO * 0.01
    ADD #VLR-DSCT-ITEM TO #VLR-TOTAL-DSCT
  ...
END-DECIDE
```

---

### Regra RND-2: Teto de Desconto — 30% do Bruto (Exceto Judicial)

```
IF tipo != 'J' AND total-descontos > bruto * 0.30
  THEN total-descontos := bruto * 0.30
END
```

**Significado:** Nenhum desconto pode ultrapassar 30% do valor bruto, EXCETO desconto judicial (que não tem teto).

**Código:** [CALCDSCT.NSN](CALCDSCT.NSN#L60-L65), [CALCDSCT.NSN](CALCDSCT.NSN#L110-L115)

```natural
MOVE 0 TO #VLR-TOTAL-DSCT
COMPUTE #VLR-MAX-DSCT = #VLR-BRUTO * 0.30
/* TRUNCAR */
COMPUTE #VLR-TEMP = #VLR-MAX-DSCT * 100
COMPUTE #VLR-MAX-DSCT = #VLR-TEMP / 100
...
IF #TIPO-DSCT NE 'J'
  IF #VLR-TOTAL-DSCT > #VLR-MAX-DSCT
    MOVE #VLR-MAX-DSCT TO #VLR-TOTAL-DSCT
  END-IF
END-IF
```

---

### Regra RND-3: Vigência de Desconto

Um desconto só é aplicável se estiver dentro do período de validade:

```
IF desconto.data-fim ≠ 0 AND desconto.data-fim < hoje
  THEN desconto expirou → SKIP
END

IF desconto.data-inicio > hoje
  THEN desconto ainda não começou → SKIP
END
```

**Código:** [CALCDSCT.NSN](CALCDSCT.NSN#L73-L80)

```natural
IF BENEFICIARIO-V.DT-FIM-DSCT(#IDX) NE 0
    AND BENEFICIARIO-V.DT-FIM-DSCT(#IDX) < #DT-HOJE
  ESCAPE TOP
END-IF
IF BENEFICIARIO-V.DT-INICIO-DSCT(#IDX) > #DT-HOJE
  ESCAPE TOP
END-IF
```

---

### Regra RND-4: Tabela de Alíquotas de Contribuição Social (4 Faixas)

Contribuição compulsória calculada por faixa de renda:

| Faixa de Renda | Alíquota |
|---|---|
| até R$ 500,00 | 3% |
| até R$ 1.000,00 | 5% |
| até R$ 2.000,00 | 7% |
| acima de R$ 2.000,00 | 9% |

**Código:** [CALCDSCT.NSN](CALCDSCT.NSN#L45-L52)

```natural
MOVE  500.00 TO #FAIXA-CONTRIB(1)
MOVE  0.03   TO #ALIQ-CONTRIB(1)
MOVE 1000.00 TO #FAIXA-CONTRIB(2)
MOVE  0.05   TO #ALIQ-CONTRIB(2)
MOVE 2000.00 TO #FAIXA-CONTRIB(3)
MOVE  0.07   TO #ALIQ-CONTRIB(3)
MOVE 9999.99 TO #FAIXA-CONTRIB(4)
MOVE  0.09   TO #ALIQ-CONTRIB(4)
```

**Nota histórica:** Alterado 30/09/2015 por Anderson Lima — novas alíquotas (então havia versão anterior).

---

## 2. CALCCORR.NSN — Correção Retroativa

### Regra RND-5: Não Corrigir Registro Já Corrigido

```
IF pagamento.ind-corrigido = 'S'
  THEN SKIP este registro
END
```

**Significado:** Um pagamento já corrigido não é reprocessado.

**Código:** [CALCCORR.NSN](CALCCORR.NSN#L96-L99)

```natural
IF PAGAMENTO-V.IND-CORRIGIDO = 'S'
  ESCAPE TOP
END-IF
```

---

### Regra RND-6: Validação de Período — Competência Inicial ≤ Competência Final

```
IF comp-inicial > comp-final
  THEN erro: "PERIODO INVALIDO"
END
```

**Código:** [CALCCORR.NSN](CALCCORR.NSN#L85-L89)

```natural
IF #COMP-INI > #COMP-FIM
  WRITE 'PERIODO INVALIDO - COMP INICIAL > FINAL'
  ESCAPE ROUTINE
END-IF
```

---

### Regra RND-7: Cálculo de Índice Acumulado por Competência

Para cada mês do período, acumular IPCA usando tabela de índices mensais:

```
indice-acumulado := 1.0
FOR cada mês de comp-inicio ATÉ comp-fim:
  indice-acumulado *= (1 + ipca-mes)
END
valor-corrigido := valor-original * indice-acumulado
```

**Código:** [CALCCORR.NSN](CALCCORR.NSN#L102-L110)

```natural
MOVE 1.000000 TO #IND-ACUM
MOVE PAGAMENTO-V.COMPETENCIA TO #COMP-ATUAL
PERFORM CALC-INDICE-ACUM
COMPUTE #VLR-CORR = #VLR-ORIG * #IND-ACUM
```

Subroutina: [CALCCORR.NSN](CALCCORR.NSN#L145-L160)

```natural
DEFINE SUBROUTINE CALC-INDICE-ACUM
  FOR #K = 1 TO 10
    IF #ANO-TAB(#K) = #ANO-C
      COMPUTE #IND-ACUM = #IND-ACUM * (1 + #IPCA-ANO(#K,#MES-C))
      ESCAPE BOTTOM
    END-IF
  END-FOR
END-SUBROUTINE
```

---

### Regra RND-8: Aplicar Correção Apenas Se Houver Diferença Positiva

```
diferenca := valor-corrigido - valor-original
IF diferenca > 0
  THEN gravar correção, marcar como corrigido
ELSE
  THEN pular este registro (sem correção)
END
```

**Código:** [CALCCORR.NSN](CALCCORR.NSN#L114-L125)

```natural
IF #VLR-DIFF > 0
  MOVE #VLR-CORR   TO PAGAMENTO-V.VLR-CORRECAO
  MOVE #DT-HOJE     TO PAGAMENTO-V.DT-CORRECAO
  MOVE 'S'          TO PAGAMENTO-V.IND-CORRIGIDO
  UPDATE PAGAMENTO-V
  END TRANSACTION
  ADD #VLR-DIFF TO #VLR-TOTAL-CORR
  ADD 1 TO #QTD-REG
END-IF
```

---

### Regra RND-9: Tabelas IPCA Mensais (2010-2012)

Índices de correção mensais, carregados em tabela. Exemplo:

| Mês | 2010 | 2011 | 2012 |
|-----|------|------|------|
| JAN | 0.75% | 0.83% | 0.56% |
| FEV | 0.78% | 0.80% | 0.45% |
| ... | ... | ... | ... |

**Código:** [CALCCORR.NSN](CALCCORR.NSN#L52-L108)

**Nota histórica:** "ULTIMA CARGA: 2014" — tabelas desatualizadas desde 2014. Bloco comentado sobre "Plano Verão" (1989-1991) sugere legado de múltiplos períodos inflacionários.

---

## 3. CALCBENF.NSN — Cálculo de Benefício Mensal

### Regra RND-10: Validação de Status do Beneficiário

```
IF beneficiario.status != 'A' (ATIVO)
  THEN erro: "BENEFICIARIO NAO ATIVO"
  THEN SKIP cálculo
END
```

**Significado:** Só calcula benefício para beneficiários com status ativo.

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L270-L273)

```natural
IF BENEFICIARIO-V.STATUS NE 'A'
  WRITE 'BENEFICIARIO NAO ATIVO - STATUS:' BENEFICIARIO-V.STATUS
  ESCAPE ROUTINE
END-IF
```

---

### Regra RND-11: Validação de Competência

```
competencia em formato AAAAMM
IF mes < 1 OR mes > 12
  THEN erro: "COMPETENCIA INVALIDA"
END
```

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L253-L258)

```natural
COMPUTE #ANO = #COMPETENCIA / 100
COMPUTE #MES = #COMPETENCIA - (#ANO * 100)
IF #MES < 1 OR #MES > 12
  WRITE 'COMPETENCIA INVALIDA'
  ESCAPE ROUTINE
END-IF
```

---

### Regra RND-12: Fator Regional por UF (27 Regiões)

Multiplicador regional varia conforme localização. Regiões: Norte(1-5), Nordeste(6-10), Sudeste(11-15), Sul(16-20), Centro-Oeste(21-25), Especial(26-27).

| Região | UF | Fator |
|--------|----|----|
| Norte | AC | 1.3500 |
| | AM | 1.3200 |
| | AP | 1.3000 |
| | PA | 1.2800 |
| | RO | 1.3100 |
| Nordeste | MA | 1.4000 |
| | PI | 1.3800 |
| | ... | ... |
| Sudeste | SP | 1.1000 |
| | RJ | 1.1200 |
| | MG | 1.0800 |
| | ES | 1.0500 |
| | **REF** | **1.0000** |
| ... | ... | ... |

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L120-L146)

```natural
MOVE 1.3500 TO #TAB-REG(1)   /* AC */
MOVE 1.3200 TO #TAB-REG(2)   /* AM */
...
MOVE 1.0000 TO #TAB-REG(15)  /* REF - REFERENCIA */
```

**Nota:** "REF" = Referência (São Paulo/região Sudeste normalizada).

**Código de aplicação:** [CALCBENF.NSN](CALCBENF.NSN#L300-L305)

```natural
IF #COD-REG >= 1 AND #COD-REG <= 25
  MOVE #TAB-REG(#COD-REG) TO #FATOR-REG
ELSE
  MOVE 1.0000 TO #FATOR-REG
END-IF
```

---

### Regra RND-13: Fator Familiar — Bônus por Dependentes

Multiplicador aumenta com número de dependentes:

| Dependentes | Fator |
|---|---|
| 0 | 1.0000 |
| 1-2 | 1.0000 + (dep × 0.0500) |
| 3-4 | 1.1000 + ((dep−2) × 0.0300) |
| 5+ | 1.1600 + ((dep−4) × 0.0200) |

**Exemplo:**
- 0 dependentes → 1.0000
- 1 dependente → 1.0500
- 2 dependentes → 1.1000
- 3 dependentes → 1.1300
- 5 dependentes → 1.2000

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L307-L321)

```natural
IF #NUM-DEP = 0
  MOVE 1.0000 TO #FATOR-FAM
ELSE
  IF #NUM-DEP <= 2
    COMPUTE #FATOR-FAM = 1.0000 + (#NUM-DEP * 0.0500)
  ELSE
    IF #NUM-DEP <= 4
      COMPUTE #FATOR-FAM = 1.1000 + ((#NUM-DEP - 2) * 0.0300)
    ELSE
      COMPUTE #FATOR-FAM = 1.1600 + ((#NUM-DEP - 4) * 0.0200)
    END-IF
  END-IF
END-IF
```

---

### Regra RND-14: Fator Renda — 5 Faixas Progressivas

Desconto por renda familiar (quanto maior a renda, menor o fator):

| Renda Familiar | Fator |
|---|---|
| até R$ 300 | 1.0000 |
| até R$ 600 | 0.8500 |
| até R$ 1.000 | 0.7000 |
| até R$ 1.500 | 0.5500 |
| acima de R$ 1.500 | 0.4000 |

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L151-L160)

```natural
MOVE 300.00  TO #FAIXA-RENDA(1)
MOVE 1.0000  TO #FATOR-FAIXA(1)
MOVE 600.00  TO #FAIXA-RENDA(2)
MOVE 0.8500  TO #FATOR-FAIXA(2)
MOVE 1000.00 TO #FAIXA-RENDA(3)
MOVE 0.7000  TO #FATOR-FAIXA(3)
MOVE 1500.00 TO #FAIXA-RENDA(4)
MOVE 0.5500  TO #FATOR-FAIXA(4)
MOVE 9999.99 TO #FAIXA-RENDA(5)
MOVE 0.4000  TO #FATOR-FAIXA(5)
```

**Nota histórica:** "ALTERADO 08/03/2013 - FERNANDA COSTA - NOVAS FAIXAS RENDA" → faixas foram mudadas nesta data.

---

### Regra RND-15: Fator Idade — Bônus para Idosos e Menores

| Faixa Etária | Fator |
|---|---|
| ≥ 65 anos | 1.1500 |
| 60-64 anos | 1.1000 |
| < 18 anos | 1.0500 |
| 18-59 anos | 1.0000 |

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L323-L337)

```natural
COMPUTE #ANO-NASC = BENEFICIARIO-V.DT-NASCIMENTO / 10000
COMPUTE #IDADE = #ANO - #ANO-NASC
IF #IDADE >= 65
  MOVE 1.1500 TO #FATOR-IDADE
ELSE
  IF #IDADE >= 60
    MOVE 1.1000 TO #FATOR-IDADE
  ELSE
    IF #IDADE < 18
      MOVE 1.0500 TO #FATOR-IDADE
    ELSE
      MOVE 1.0000 TO #FATOR-IDADE
    END-IF
  END-IF
END-IF
```

---

### Regra RND-16: Fórmula Principal — Cálculo do Benefício Mensal

```
BENEFICIO = BASE × FATOR_REGIONAL × FATOR_FAMILIAR × FATOR_RENDA × FATOR_IDADE
            × (1 + REAJUSTE_PROGRAMA)
```

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L340-L347)

```natural
COMPUTE #VLR-BENF = #VLR-BASE * #FATOR-REG * #FATOR-FAM
                     * #FATOR-RND * #FATOR-IDADE
* APLICAR REAJUSTE DO PROGRAMA
COMPUTE #VLR-BENF = #VLR-BENF * (1 + #FATOR-REAJ)
* TRUNCAR P/ 2 CASAS DECIMAIS
COMPUTE #VLR-TEMP = #VLR-BENF * 100
COMPUTE #VLR-BENF = #VLR-TEMP / 100
```

---

### Regra RND-17: 13º Salário — Dezembro Apenas, Fórmula Diferenciada

```
IF mes = 12 (DEZEMBRO)
  THEN tipo_pagamento := 'D' (DECIMO)
  THEN VLR_13 = BASE × FATOR_REGIONAL × FATOR_IDADE
       (nota: sem fator familiar, sem fator renda)
  THEN VLR_BRUTO_TOTAL = VLR_BENEFICIO + VLR_13
END
```

**Significado:** O 13º não recebe desconto por renda ou família, apenas correção regional e por idade.

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L349-L373)

```natural
IF #MES = 12
  MOVE 'D' TO #TIPO-PGTO
  COMPUTE #VLR-13 = #VLR-BASE * #FATOR-REG * #FATOR-IDADE
* TRUNCAR 13O
  COMPUTE #VLR-TEMP = #VLR-13 * 100
  COMPUTE #VLR-13 = #VLR-TEMP / 100
  COMPUTE #VLR-BRUTO = #VLR-BENF + #VLR-13
```

---

### Regra RND-18: Abono Natalino — 15% Adicional (Apenas Programas Tipo 'A')

```
IF mes = 12 AND tipo_programa = 'A' (ASSISTENCIAL)
  THEN VLR_ABONO = VLR_BENEFICIO × 0.15
  THEN VLR_BRUTO_TOTAL = VLR_BENEFICIO + VLR_13 + VLR_ABONO
ELSE
  THEN VLR_ABONO = 0
END
```

**Significado:** Programas assistenciais (tipo 'A') recebem 15% extra em dezembro para abono de fim de ano.

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L356-L365)

```natural
IF #TIPO-PROG = 'A'
  COMPUTE #VLR-ABONO = #VLR-BENF * 0.15
* TRUNCAR ABONO
  COMPUTE #VLR-TEMP = #VLR-ABONO * 100
  COMPUTE #VLR-ABONO = #VLR-TEMP / 100
  COMPUTE #VLR-BRUTO = #VLR-BRUTO + #VLR-ABONO
ELSE
  MOVE 0 TO #VLR-ABONO
END-IF
```

---

### Regra RND-19: Padrão Mainframe — Truncamento para 2 Casas Decimais

```
valor_truncado = INT(valor × 100) / 100
```

**Significado:** Não é arredondamento; é truncamento (corte sem arredondamento). Padrão histórico do Adabas/Mainframe.

**Código recorrente:** [CALCBENF.NSN](CALCBENF.NSN#L344-L347), [CALCBENF.NSN](CALCBENF.NSN#L356), [CALCBENF.NSN](CALCBENF.NSN#L368-L376), [CALCDSCT.NSN](CALCDSCT.NSN#L63-L65)

```natural
COMPUTE #VLR-TEMP = #VLR-BENF * 100
COMPUTE #VLR-BENF = #VLR-TEMP / 100
```

---

### Regra RND-20: Valor Líquido Nunca Negativo

```
IF VLR_BRUTO - DESCONTO < 0
  THEN VLR_LIQUIDO = 0
ELSE
  THEN VLR_LIQUIDO = VLR_BRUTO - DESCONTO
END
```

**Código:** [CALCBENF.NSN](CALCBENF.NSN#L378-L383)

```natural
COMPUTE #VLR-LIQ = #VLR-BRUTO - #VLR-DESC
IF #VLR-LIQ < 0
  MOVE 0 TO #VLR-LIQ
END-IF
```

---

## Resumo de Regras Críticas para Modernização

| ID | Título | Tipo | Impacto |
|----|----|----|----|
| RND-1 | Tipos de desconto | Lógica condicional | **ALTO** — core de cálculo |
| RND-2 | Teto 30% (exceto judicial) | Limite de negócio | **ALTO** — compliance |
| RND-3 | Vigência de desconto | Validação temporal | **MÉDIO** — controle |
| RND-4 | Alíquotas contrib. social | Tabela | **ALTO** — cálculo obrigatório |
| RND-5 | Não corrigir 2x | Idempotência | **MÉDIO** — evita duplicação |
| RND-6 | Validação período | Validação entrada | **BAIXO** — UX |
| RND-7 | IPCA acumulado | Fórmula financeira | **ALTO** — precisão |
| RND-8 | Corrigir só se > 0 | Lógica condicional | **MÉDIO** — regra negócios |
| RND-9 | Tabelas IPCA | Base histórica | **MÉDIO** — manutenção |
| RND-10 | Status 'A' obrigatório | Validação estado | **ALTO** — precondição |
| RND-11 | Mês 1-12 | Validação entrada | **BAIXO** — UX |
| RND-12 | Fator regional (27 UFs) | Tabela paramétrica | **ALTO** — desigualdade regional |
| RND-13 | Fator familiar | Fórmula progressiva | **ALTO** — inclusão social |
| RND-14 | Fator renda (5 faixas) | Fórmula progressiva | **ALTO** — focalização |
| RND-15 | Fator idade (4 faixas) | Fórmula | **MÉDIO** — proteção grupos |
| RND-16 | Fórmula benefício | Core matemático | **CRÍTICO** — tudo depende |
| RND-17 | 13º em dezembro | Lógica temporal | **ALTO** — benefício extra |
| RND-18 | Abono 15% (tipo A) | Lógica condicional | **MÉDIO** — programa específico |
| RND-19 | Truncamento 2 casas | Precisão | **MÉDIO** — compatibilidade |
| RND-20 | Líquido ≥ 0 | Constraint | **BAIXO** — proteção |

---

## Notas Históricas e Mudanças

1. **CALCDSCT (Descontos):**
   - 25/08/1999 — Versão original
   - 12/04/2007 — Adilson Batista: inclusão desconto judicial
   - 30/09/2015 — Anderson Lima: novas alíquotas

2. **CALCCORR (Correção):**
   - 12/07/2001 — Versão original (Patricia Gomes)
   - 20/01/2006 — Patricia: novos índices IPCA
   - 15/08/2014 — Anderson: ajuste período
   - **BLOCO COMENTADO:** "Plano Verão" (1989-1991), responsável Joao Batista — nunca remover

3. **CALCBENF (Benefício):**
   - 18/04/1997 — Versão original
   - 30/11/2001 — Inclusão 13º salário
   - 15/06/2004 — Ajuste fator regional
   - 22/12/2009 — Abono natalino
   - 08/03/2013 — **Novas faixas de renda** (regra crítica)

---

## Mapeamento para Testes de Equivalência

Cada regra RND-* deve ter **ao menos um teste automatizado** na modernização:

- **RND-1 a RND-4:** Testes de cálculo de desconto (matriz de tipos × valores)
- **RND-7, RND-8:** Testes de correção retroativa (período válido, período nulo, já corrigido)
- **RND-10 a RND-20:** Testes parametrizados de benefício (status, competência, UF, dependentes, renda, idade, tipo programa)

**Próximo passo:** Mapear estes para casos de teste BDD (Given-When-Then).
