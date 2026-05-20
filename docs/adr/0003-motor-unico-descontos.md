<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# ADR-0003: Motor único de descontos — `CALCDSCT` como fonte da verdade

| Campo     | Valor                                                                |
| --------- | -------------------------------------------------------------------- |
| Status    | **aceito**                                                           |
| Data      | 2026-05-20                                                           |
| Autores   | Pedro (SA), Cleber (TL), Bruno (PO — confirma D4) — Os Laranjinhas    |
| Substitui | N/A                                                                  |

## Contexto

O legado tem **duas implementações de desconto** que coexistem há anos — [`01-arqueologia/mysteries-found.md`](../../01-arqueologia/mysteries-found.md) MYS-024:

1. **`CALCBENF.NSN#L307-L315`** — aplica desconto **inline** durante o cálculo do benefício (3% se bruto > R$ 500, BR-021).
2. **`CALCDSCT.NSN`** — programa formal de descontos com:
   - 4 faixas de contribuição social (BR-028: ≤500→3%, ≤1000→5%, ≤2000→7%, >2000→9%)
   - Desconto sindical fixo 1% (BR-026)
   - Desconto judicial **sem teto** (BR-025) — invariante legal confirmada com PO (D4)
   - Outros descontos com teto agregado de 30% do bruto
   - Vigência por período (BR-027: `DT-INICIO-DSCT ≤ hoje ≤ DT-FIM-DSCT`)

**Risco no legado:** o mesmo pagamento pode ter descontos calculados em dois lugares com regras diferentes. Não sabemos hoje qual é o valor "oficial" — depende de qual programa rodou por último para aquele registro.

Adicional: BR-026 (sindical 1% hardcoded) e BR-028 (faixas hardcoded) não podem ser parametrizadas hoje sem novo build.

## Decisão

1. **`CALCDSCT` é a única autoridade de desconto no SIFAP 2.0.** No contexto `payment`, apenas o serviço `DeductionEngine` (port no `application/`) emite valores de desconto. `CALCBENF` reimplementado (`BenefitCalculator`) **não aplica desconto** — só calcula bruto e delega.
2. **Tipos de desconto modelados como `sealed interface Deduction`** no domínio:

   ```text
   sealed interface Deduction permits SocialContribution, Union, Judicial, Other
   ```

   Cada implementação carrega sua própria regra (faixa, percentual fixo, valor absoluto). `Judicial` é a única que NÃO passa pelo teto agregado de 30% (BR-025/D4).

3. **Ordem de aplicação fixa e auditável:**
   1. `SocialContribution` (4 faixas por bruto — BR-028)
   2. `Union` (1% fixo — BR-026)
   3. `Other` (livre, com teto agregado de 30% — BR-001 do tipo "unwanted")
   4. `Judicial` (sem teto — BR-025)
4. **Piso zero (BR-022):** `valorLiquido = max(0, valorBruto - Σ deducoes)`. Nunca negativo.
5. **Parametrização:** as 4 faixas de BR-028 e o 1% sindical de BR-026 viram tabelas no schema `payment.deduction_rule` versionadas por `vigencia_inicio`/`vigencia_fim`. Mudança requer commit no schema + auditoria — não exige novo build.
6. **Auditoria obrigatória:** cada `DeductionApplied` vira evento de domínio consumido por `audit`, com `deductionType`, `rule.id`, `valorAplicado`, `motivoSeAjustadoAoTeto`.
7. **Vigência (BR-027):** descontos com `dataFim != null` e `hoje > dataFim` são ignorados silenciosamente; descontos com `dataInicio > hoje` também. `DeductionEngine` aplica filtro antes de calcular.

## Alternativas consideradas

| Alternativa                                                       | Por que foi rejeitada                                                                                                                                                  |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A. Manter as duas implementações (inline + formal)**            | Perpetua MYS-024 na arquitetura moderna. Diverge na primeira diferença de regra. Time legal hoje não consegue dizer qual valor é "correto".                            |
| **B. Eleger `CALCBENF` (inline) como fonte da verdade**           | Implementação mais simples mas **incompleta** — não cobre desconto judicial (BR-025) nem vigência (BR-027). Migrar para inline = perder regras críticas.               |
| **C. Motor único mas mantendo regras hardcoded em código Java**   | Resolve a duplicação mas não a rigidez. Cada alteração de alíquota (BR-028) ainda exige build + deploy. Decisão política exige resposta em dias, não em sprints.       |
| **D. Drools / engine de regras externo**                          | Overkill para 4 tipos de desconto. Adiciona dependência pesada, curva de aprendizado, segundo "lugar" para procurar a verdade. Tabela parametrizada já resolve.        |
| **E. Aplicar teto de 30% também sobre judicial**                  | **Ilegal.** D4 confirmado com PO/jurídico: descumprir ordem judicial é risco maior que zerar o líquido. Aplicar teto sobre judicial expõe a instituição.               |

## Consequências

- **Mais fácil:**
  - Auditoria responde em 1 query "qual desconto, qual regra, em qual vigência" para qualquer pagamento.
  - Mudança política (ex.: nova faixa de contribuição) = `INSERT` em `deduction_rule` com nova vigência, sem deploy.
  - Teste unitário foca em `DeductionEngine` isolado — alta cobertura possível.
- **Mais difícil:**
  - Migração de dados de descontos legados — alguns pagamentos antigos não têm `regra_id` rastreável. **Mitigação:** marcar como `legacy_rule` na migração; documentar como dívida técnica resolvida ao longo de 12 ciclos.
  - Time precisa entender o domínio (4 tipos, ordem, vigência) antes de tocar. **Mitigação:** `glossary.md` + diagrama sequencial no `data-model.md`.
- **Riscos:**
  - **R-DSC1:** alguém adiciona uma 5ª faixa de contribuição no domínio sem atualizar `deduction_rule`. **Mitigação:** repositório `DeductionRuleRepository` é a única forma de obter regra; sealed interface garante exaustividade no `switch` (Java 21).
  - **R-DSC2:** teto de 30% calcula sobre bruto pré- ou pós-judicial — ambiguidade. **Decisão explícita:** teto incide sobre **bruto cheio**, e judicial é aplicado **depois** do teto sem ser somado a ele. Documentado em REQ-PAY-005.
- **Critérios de envelhecimento:**
  - Surgir 5º tipo de desconto que não case com `sealed interface` atual → revisar modelo.
  - Mudança regulatória que crie teto sobre judicial → revogar D4 e este ADR.

## Relacionado

- REQ-IDs: REQ-PAY-004, REQ-PAY-005, REQ-PAY-006
- BRs: BR-021, BR-025, BR-026, BR-027, BR-028, BR-022
- Mistérios resolvidos: MYS-024
- ADRs: ADR-0001 (Modular Monolith), ADR-0002 (Money é insumo de cálculo)
- Decisão pendente confirmada: D4 (judicial sem teto) em [`02-spec-moderna/scope-decisions.md`](../../02-spec-moderna/scope-decisions.md)
- Arquivos legado: [`CALCDSCT.NSN`](../../01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN), [`CALCBENF.NSN#L307-L315`](../../01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN)

## Referências

- Java 21 — Sealed Classes & Pattern Matching for `switch` (JEP 441)
- DDD: Strategy/Specification para regras parametrizáveis
- Lei nº 10.820/2003 (consignação) e CPC art. 833 — descontos judiciais sobre benefícios sociais
