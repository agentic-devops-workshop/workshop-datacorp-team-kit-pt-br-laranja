<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# ADR-0002: Política Financeira Única — `Money` em BigDecimal, escala 2, HALF_EVEN

| Campo     | Valor                                                       |
| --------- | ----------------------------------------------------------- |
| Status    | **aceito**                                                  |
| Data      | 2026-05-20                                                  |
| Autores   | Pedro (SA), Tiago (DBA/QA), Bruno (PO) — Os Laranjinhas     |
| Substitui | N/A                                                         |

## Contexto

O legado SIFAP tem **inconsistência financeira plantada** entre seus programas — documentada em [`01-arqueologia/mysteries-found.md`](../../01-arqueologia/mysteries-found.md):

- **MYS-001 / BR-009:** `BATCHREL.NSN#L118-L121` **arredonda** valor bruto (`+ 0.005` e depois `INT(x×100)/100`).
- **BR-018:** `BATCHPGT.NSN#L251-L253` **trunca** o mesmo valor (`INT(x×100)/100` sem o `+0.005`).
- **MYS-012:** truncagem `*100/100` se espalhou por `CALCBENF`, `CALCDSCT`, `CALCCORR`.

Resultado: o relatório mensal **não bate** com o somatório real dos pagamentos, em centavos × milhões de registros. Auditoria contábil acusa diferença recorrente desde pelo menos 2006 (alteração de Roberto Mendes).

Sem decisão única na modernização, a divergência se replica ou — pior — vira nova divergência se cada desenvolvedor escolher política diferente.

Restrições adicionais:

- **Valores em centavos no CNAB 240** (BR-002) precisam ser convertidos para reais sem perda.
- **R$ 0,01 de tolerância de conciliação** (BR-004) é regra contábil real e fica preservada.
- **PostgreSQL `NUMERIC(15,2)`** é o tipo de armazenamento — qualquer arredondamento aconteceria na camada de aplicação.

## Decisão

**Adotamos `Money` como value object único do `shared-kernel`**, com as seguintes invariantes:

1. **Implementação interna:** `java.math.BigDecimal` com escala fixa = **2** e `RoundingMode.HALF_EVEN` (arredondamento bancário ISO 31-0, evita viés sistemático).
2. **Construção:**
   - `Money.ofReais(BigDecimal)` — entrada já em reais.
   - `Money.ofCentavos(long)` — entrada em centavos (uso obrigatório no parser CNAB, resolve BR-002).
   - Construtor privado; nunca aceita `double` ou `float`.
3. **Operações:** `add`, `subtract`, `multiply(BigDecimal factor)`, `divide(BigDecimal divisor)`. Toda operação que reduz escala usa `HALF_EVEN`. Comparação por `compareTo`, nunca `equals` para evitar pegadinha de escala.
4. **Política de relatório:** relatórios **não re-arredondam**. Somatório é `Σ Money.add(...)` sobre os valores já persistidos (que são `NUMERIC(15,2)`). Isso resolve MYS-001 estruturalmente.
5. **Persistência JPA:** `@Convert(converter = MoneyConverter.class)` mapeando para `NUMERIC(15,2)` em todos os schemas.
6. **Logging/serialização:** `toString()` produz `"R$ 1.234,56"` (locale pt-BR); JSON serializa como número decimal (`1234.56`), nunca string.

**Migração de dados legados:** durante o cutover, todo `VLR-BRUTO`/`VLR-LIQUIDO` do Adabas é lido como BigDecimal escala 2 sem reaplicar arredondamento — o valor **já gravado** é a fonte da verdade, não a fórmula.

## Alternativas consideradas

| Alternativa                                              | Por que foi rejeitada                                                                                                                                                          |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **A. `double` ou `float`**                               | Inaceitável para moeda — IEEE 754 introduz erro em soma simples (`0.1 + 0.2 ≠ 0.3`). Causa de incidentes em sistemas financeiros há décadas.                                   |
| **B. `BigDecimal` cru, sem wrapper**                     | Cada chamador escolhe escala e modo — replica o problema do legado. Sem encapsulamento, é fácil errar.                                                                         |
| **C. Joda-Money ou JSR-354 (Moneta)**                    | Considerado; adiciona dependência externa para problema pequeno (sempre BRL). Custos: 1 lib a manter + curva de aprendizado. Não traz vantagem real para escopo mono-moeda.    |
| **D. Preservar truncate (`HALF_DOWN`) como no legado**   | Replica viés sistemático (sempre a favor do governo, contra o beneficiário). Soma de 180M registros gera diferença significativa. HALF_EVEN é neutro estatisticamente.         |
| **E. Preservar round (`HALF_UP`) como em BATCHREL**      | Resolve MYS-001 escolhendo o lado oposto, mas reintroduz viés (favorece beneficiário em metade dos casos e governo em nenhum). HALF_EVEN é mais defensável em auditoria.       |
| **F. Armazenar em centavos como `long`**                 | Elimina arredondamento mas viola compatibilidade com o schema PostgreSQL planejado (`NUMERIC(15,2)`) e dificulta queries ad-hoc por DBAs. Toda saída SQL exige conversão manual. |

## Consequências

- **Mais fácil:**
  - Reconciliação contábil resolvida no design (BR-009 vs BR-018 deixam de divergir).
  - Code review: regra única, simples de auditar.
  - Testes de equivalência com legado: diff esperado em centavos é **conhecido** (HALF_EVEN vs truncate); cada caso documenta-se em fixture.
- **Mais difícil:**
  - Migração: pagamentos antigos foram gravados com truncate. Recalcular = mudar histórico (proibido). **Mitigação:** congelar política para registros novos; histórico fica como está, com nota de migração no `audit`.
  - Treinar time para nunca usar `BigDecimal` diretamente. **Mitigação:** ArchUnit regra `noFieldOfType(BigDecimal.class).inPackages("..application..", "..domain..")`.
- **Riscos:**
  - **R-FIN1:** algum desenvolvedor usa `double` em hotfix de produção. **Mitigação:** regra ArchUnit + revisão de PR.
  - **R-FIN2:** shadow test contra legado mostra diferença > R$ 0,01 (limite BR-004). **Mitigação:** N5 (métrica `payment.cycle.diff_vs_legacy_centavos`) bloqueia cutover.
- **Critérios de envelhecimento:**
  - Necessidade de multi-moeda (improvável no SIFAP, mas possível em integração SIAFI).
  - Mudança de norma contábil federal sobre arredondamento.

## Relacionado

- REQ-IDs: REQ-PAY-001, REQ-PAY-003, REQ-PAY-005, REQ-RPT-001, REQ-REC-001
- BRs: BR-002, BR-004, BR-009, BR-017, BR-018, BR-022
- Mistérios resolvidos: MYS-001, MYS-012
- ADRs: ADR-0001 (estrutura), ADR-0003 (descontos consomem `Money`)
- Arquivos legado: [`BATCHPGT.NSN#L251-L253`](../../01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN), [`BATCHREL.NSN#L118-L121`](../../01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN)

## Referências

- ISO 31-0 (HALF_EVEN — Banker's Rounding)
- Brian Goetz, *Java Concurrency in Practice* — value objects imutáveis
- Martin Fowler, *Patterns of Enterprise Application Architecture* — Money pattern
