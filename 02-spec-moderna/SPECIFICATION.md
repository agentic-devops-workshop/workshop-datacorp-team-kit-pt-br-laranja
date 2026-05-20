<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# SPECIFICATION — SIFAP 2.0

![ESTÁGIO 02 Spec](https://img.shields.io/badge/ESTÁGIO-02%20Spec-00A4EF?style=for-the-badge) ![TIME Os Laranjinhas](https://img.shields.io/badge/TIME-Os%20Laranjinhas-F25022?style=for-the-badge) ![REQ-IDs 14](https://img.shields.io/badge/REQ--IDs-14-7FBA00?style=for-the-badge)

## Metadados

- **Versão da spec:** 0.1.0 (Estágio 2 — fim)
- **Time:** Os Laranjinhas (Bruno PO/RE, Pedro EA/SA, Cleber TL/Dev, Tiago DBA/QA, Bruna+Samuel DevOps/Writer)
- **Aprovado pelo Product Owner:** ☑ Bruno (sign-off em [`scope-decisions.md`](scope-decisions.md))
- **Origem dos requisitos:** [`01-arqueologia/business-rules-catalog.md`](../01-arqueologia/business-rules-catalog.md) (45 BRs analisadas), [`01-arqueologia/mysteries-found.md`](../01-arqueologia/mysteries-found.md) (24 mistérios), [`scope-decisions.md`](scope-decisions.md)
- **Contextos cobertos:** `beneficiary`, `program-catalog`, `payment`, `reconciliation`, `audit` (ver [`bounded-contexts.md`](bounded-contexts.md))

## Padrões EARS usados

| Padrão           | Quando usar                              | Exemplo deste documento  |
| ---------------- | ---------------------------------------- | ------------------------ |
| `ubiquitous`     | "O sistema deve sempre…"                 | REQ-AUD-001              |
| `event-driven`   | "Quando X acontece, o sistema deve…"     | REQ-PAY-001, REQ-REC-001 |
| `state-driven`   | "Enquanto X estiver…, o sistema deve…"   | REQ-PAY-006              |
| `unwanted`       | "O sistema não deve…"                    | REQ-PAY-005, REQ-BEN-005 |
| `optional`       | "Onde X for selecionado, o sistema deve…"| REQ-PAY-007              |
| `complex`        | Composição de gatilhos múltiplos         | REQ-PAY-002              |

---

## 1. Contexto `beneficiary`

### REQ-BEN-001 · Validação consolidada de CPF

```yaml
REQ-BEN-001:
  context: beneficiary
  pattern: ubiquitous
  text: "O SIFAP deve validar CPF de beneficiário usando algoritmo módulo-11
         padrão Receita Federal, rejeitando qualquer CPF que falhe na verificação
         dos dois dígitos verificadores."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN#L243-L295
  business_rule: BR-035
  acceptance:
    - "CPF '11144477735' é aceito (dígitos válidos)."
    - "CPF '11111111111' é rejeitado com erro VALIDATION_ERROR.cpf.invalid."
    - "CPF '12345678900' é rejeitado (dígitos verificadores incorretos)."
    - "Mesma validação aplica em CADBENEF, VALBENEF, VALDOCS (motor único — consolidação)."
  priority: P0
  risk: CRÍTICO
  notes: "Consolida BR-035, BR-049, BR-053, BR-058 num único motor (resolve duplicação detectada em discovery §3.2)."
```

### REQ-BEN-002 · Bloqueio de backdoors de CPF

```yaml
REQ-BEN-002:
  context: beneficiary
  pattern: unwanted
  text: "O SIFAP não deve aceitar como válidos os prefixos de CPF
         000, 001, 002, 010, 011, 099, 100, 999, nem CPFs com 11 dígitos iguais,
         independentemente do dígito verificador."
  source_legacy: "[GREENFIELD] Remoção da backdoor MYS-019/MYS-021/EGG-002 confirmada em scope-decisions.md (D1). Risco de fraude inaceitável em produção."
  business_rule: "BR-049, BR-053 (anti-padrão removido)"
  acceptance:
    - "CPF '00000000000' é rejeitado (era aceito no legado por MYS-019)."
    - "CPF começando com '999' é rejeitado mesmo se dígitos verificadores baterem."
    - "Ambiente de homologação (greenfield N6) usa CPFs sintéticos pré-gerados em pool isolado."
  priority: P0
  risk: CRÍTICO
```

### REQ-BEN-003 · Unicidade de CPF no cadastro

```yaml
REQ-BEN-003:
  context: beneficiary
  pattern: unwanted
  text: "O SIFAP não deve permitir cadastrar dois beneficiários com o mesmo CPF."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN#L141-L151
  business_rule: BR-036
  acceptance:
    - "Tentar POST /api/v1/beneficiaries com CPF já existente → HTTP 409 Conflict."
    - "Restrição UNIQUE no schema beneficiary.beneficiary(cpf)."
  priority: P0
  risk: ALTO
```

### REQ-BEN-004 · Status inicial e regra demográfica

```yaml
REQ-BEN-004:
  context: beneficiary
  pattern: event-driven
  text: "Quando um beneficiário é cadastrado, o SIFAP deve atribuir status='A' (Ativo)
         por padrão, EXCETO quando a idade calculada ultrapassar 75 anos,
         caso em que o status inicial deve ser 'S' (Suspenso) com motivo
         'demographic_review' registrado em audit."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN#L165-L174
  business_rule: BR-037
  acceptance:
    - "Beneficiário com 50 anos é cadastrado com status='A'."
    - "Beneficiário com 80 anos é cadastrado com status='S' e evento BeneficiaryStatusChanged(reason='demographic_review') publicado."
    - "REQ-AUD-001 garante rastro dessa decisão."
  priority: P0
  risk: CRÍTICO
  notes: "Decisão D3/MYS-014: preservar regra dos 75 anos por enquanto, mas tornar visível (legado fazia silenciosamente). PO deve validar política antes do cutover."
```

### REQ-BEN-005 · Mascaramento de CPF em consulta e logs

```yaml
REQ-BEN-005:
  context: beneficiary
  pattern: unwanted
  text: "O SIFAP não deve exibir CPF completo em respostas de consulta ou em logs;
         deve aplicar o padrão XXX.XXX.NNN-NN (preservando apenas os 3 dígitos
         centrais e os 2 verificadores)."
  source_legacy: "[GREENFIELD] LGPD Art. 6º (minimização). Substitui máscara bugada do legado (MYS-017 / CONSBENF.NSN#L168-L189) — comentário 'INCONSISTENCIA CONHECIDA' foi resolvido com nova implementação correta."
  business_rule: BR-044 (substituído por implementação correta)
  acceptance:
    - "GET /api/v1/beneficiaries/{id} retorna 'cpfMasked': '***.456.789-**'."
    - "Log estruturado de DEBUG não imprime CPF cru em nenhum campo."
    - "Endpoint /actuator/logfile não vaza CPF."
    - "Auditor com perfil AUD pode obter CPF completo via GET /api/v1/beneficiaries/{id}?unmask=true (registrado em audit)."
  priority: P0
  risk: CRÍTICO
```

### REQ-BEN-006 · Limite parametrizado de dependentes

```yaml
REQ-BEN-006:
  context: beneficiary
  pattern: unwanted
  text: "O SIFAP não deve permitir cadastrar mais do que N dependentes por beneficiário,
         onde N é parâmetro do sistema (default=5 conforme legado)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/CADDEPEND.NSN#L59-L62
  business_rule: BR-039
  acceptance:
    - "Cadastro de 6º dependente para titular com 5 dependentes → HTTP 409 Conflict."
    - "Limite vive em beneficiary.system_parameter(key='max_dependents'); alterar não exige deploy."
    - "Decisão D5 (5 vs 10) diferida para /speckit.clarify no Estágio 3 — default 5 mantém comportamento legado."
  priority: P1
  risk: MÉDIO
  notes: "Resolve MYS-015 (legado hardcoded em 5 mas DDM permite 10)."
```

---

## 2. Contexto `program-catalog`

### REQ-PRG-001 · Cadastro de programa preserva fórmula de base

```yaml
REQ-PRG-001:
  context: program-catalog
  pattern: event-driven
  text: "Quando um programa social é cadastrado, o SIFAP deve armazenar
         VLR-BASE-EFETIVO calculado como VLR-BASE-INFORMADO × (1 + FATOR-REAJUSTE × C),
         onde C é constante atuarial parametrizada (default=0.347215, congelada conforme D7)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/CADPROG.NSN#L75-L78
  business_rule: BR-042
  acceptance:
    - "Programa com base=R$ 100 e reajuste=0.05 produz baseEfetivo=R$ 101,7361 truncado para R$ 101,74 (HALF_EVEN — ADR-0002)."
    - "Constante C vive em program_catalog.system_parameter(key='actuarial_factor')."
    - "Alteração de C requer perfil ADM + ADR novo (origem da fórmula MYS-016 ainda não documentada)."
  priority: P0
  risk: ALTO
  notes: "Decisão D7 (origem de 0.347215) diferida para /speckit.clarify. Comportamento default = preservar legado."
```

---

## 3. Contexto `payment`

### REQ-PAY-001 · Geração de ciclo mensal (folha)

```yaml
REQ-PAY-001:
  context: payment
  pattern: event-driven
  text: "Quando o ciclo de pagamento mensal é iniciado, o SIFAP deve gerar
         exatamente um Payment com status='G' (Gerado) para cada par
         (beneficiário ativo, programa social ativo) na competência informada,
         desde que não exista já Payment para esse par naquela competência."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L150-L192
  business_rule: BR-011, BR-012, BR-023
  acceptance:
    - "100 beneficiários ACTIVE em programa ACTIVE → 100 pagamentos status='G' gerados."
    - "Rodar o ciclo duas vezes para a mesma competência → segundo run gera 0 pagamentos (idempotência BR-012)."
    - "Beneficiário com status='S' (suspenso) é ignorado."
    - "Programa com STATUS-PROG='I' é ignorado mesmo com beneficiários ativos."
    - "Competência derivada como (ano × 100 + mês) conforme BR-024."
  priority: P0
  risk: CRÍTICO
  notes: "BR-012 é a defesa anti-duplicação mensal — testar reprocessamento explicitamente."
```

### REQ-PAY-002 · Fórmula-mãe do benefício bruto

```yaml
REQ-PAY-002:
  context: payment
  pattern: complex
  text: "Quando o cálculo do benefício bruto é executado para um Payment,
         o SIFAP deve aplicar:
           bruto = VLR-BASE × fatorRegiao × fatorFamilia × fatorRenda × fatorIdade × (1 + fatorReajuste)
         onde:
           - fatorRegiao vem da tabela regional indexada por COD-REGIAO 1..25 (default=1.0)
           - fatorFamilia depende de numDependentes (BR-014)
           - fatorRenda depende de rendaFamiliar em 5 faixas (BR-015)
           - fatorIdade depende da idade calculada (BR-016)
         e armazenar o resultado em Money escala 2 (HALF_EVEN, ADR-0002)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L245-L250
  business_rule: BR-013, BR-014, BR-015, BR-016, BR-017, BR-018
  acceptance:
    - "Caso de teste #1 (fixture do legado): benefVlrBase=300, reg=Sul(fator=1.05), 2 deps(1.10), renda=400(0.85), idade=70(1.15), reajuste=0.05 → bruto esperado R$ 397,38."
    - "Shadow test contra dump de 100 pagamentos do legado em ambiente isolado: diff por pagamento ≤ R$ 0,00 (HALF_EVEN compensa truncate legado em média)."
    - "Métrica N5 (payment.cycle.diff_vs_legacy_centavos) reporta diff agregado por execução."
    - "Decisão MYS-003 (idade só por ano vs mês/dia): preservar comportamento legado por default; flag de feature 'precise_age' começa desligada."
  priority: P0
  risk: CRÍTICO
  notes: "Coração do sistema. Teste de equivalência é GATE para cutover."
```

### REQ-PAY-003 · 13º salário em dezembro

```yaml
REQ-PAY-003:
  context: payment
  pattern: event-driven
  text: "Quando o ciclo é executado para competência de dezembro (mês=12),
         o SIFAP deve gerar pagamento adicional com tipoPgto='D' (Décimo)
         calculado como VLR-BASE × fatorRegiao × fatorIdade (sem fatorFamilia,
         fatorRenda nem fatorReajuste)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L260-L266
  business_rule: BR-019
  acceptance:
    - "Competência 202612 gera 2 pagamentos por beneficiário: tipoPgto='M' (mensal) e tipoPgto='D' (décimo)."
    - "Competência 202611 gera 1 pagamento (mensal) por beneficiário."
    - "Cálculo de décimo NÃO aplica desconto de renda (testar com beneficiário renda alta)."
  priority: P0
  risk: CRÍTICO
```

### REQ-PAY-004 · Abono dezembrino para programas tipo 'A'

```yaml
REQ-PAY-004:
  context: payment
  pattern: event-driven
  text: "Quando o ciclo é executado em dezembro para um programa com TIPO='A',
         o SIFAP deve adicionar valorAbono = brutoMensal × percentualAbono ao Payment,
         onde percentualAbono é parâmetro (default=0.15 conforme legado)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L267-L271
  business_rule: BR-020
  acceptance:
    - "Programa TIPO='A', bruto=R$ 1000 em dezembro → valorAbono=R$ 150."
    - "Programa TIPO='B' em dezembro → valorAbono=R$ 0."
    - "Percentual vive em payment.system_parameter(key='december_bonus_pct'); resolve MYS-009."
  priority: P0
  risk: CRÍTICO
```

### REQ-PAY-005 · Teto de descontos não-judiciais e exceção judicial

```yaml
REQ-PAY-005:
  context: payment
  pattern: unwanted
  text: "O SIFAP não deve permitir que o total de descontos NÃO judiciais
         exceda 30% do valor bruto do Payment; descontos do tipo Judicial
         devem ser aplicados integralmente sem participar do cálculo do teto
         (mas após o teto dos demais)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L121-L158
  business_rule: BR-025, BR-001 (legado de CALCDSCT modelado)
  acceptance:
    - "Bruto=R$ 1000, descontos não-judiciais somando R$ 400 → teto aplicado, total aceito=R$ 300."
    - "Bruto=R$ 1000, judicial=R$ 500, demais=R$ 0 → judicial aceito integralmente, líquido=R$ 500."
    - "Bruto=R$ 1000, sindical=R$ 10 + contribSocial=R$ 50 + judicial=R$ 800 → demais=R$ 60 (≤300), judicial=R$ 800 → líquido=R$ 140."
    - "Líquido nunca negativo (BR-022) — se descontos > bruto, líquido=R$ 0,00 e evento DeductionOverflow publicado."
  priority: P0
  risk: CRÍTICO
  notes: "ADR-0003 detalha o motor único. D4 (judicial sem teto) confirmado com PO."
```

### REQ-PAY-006 · Máquina de estados do pagamento

```yaml
REQ-PAY-006:
  context: payment
  pattern: state-driven
  text: "Enquanto um Payment estiver com status='G' (Gerado),
         o SIFAP deve aceitar transições para 'P' (Pago), 'D' (Devolvido),
         'E' (Estornado) ou 'C' (Cancelado) — nesta ordem de prioridade
         conforme retorno bancário (BR-005) ou ação administrativa."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L160-L182
  business_rule: BR-005, BR-010, BR-023
  acceptance:
    - "Status='G' aceita transição para P, D, E, C."
    - "Status='P' não aceita transição para 'G' (HTTP 409 Conflict)."
    - "Status='C' é terminal — nenhuma transição aceita."
    - "Toda transição publica evento PaymentStatusChanged consumido por audit (REQ-AUD-001)."
  priority: P0
  risk: CRÍTICO
```

### REQ-PAY-007 · Cancelamento de pagamento por admin

```yaml
REQ-PAY-007:
  context: payment
  pattern: optional
  text: "Onde um usuário com perfil ADM solicitar cancelamento de um Payment
         em status='G', o SIFAP deve transicionar o status para 'C' (Cancelado)
         registrando motivo obrigatório (≥10 caracteres)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L150-L192
  business_rule: BR-010
  acceptance:
    - "ADM POST /api/v1/payments/{id}/cancel com motivo válido → HTTP 200, status='C'."
    - "OPR POST /api/v1/payments/{id}/cancel → HTTP 403 Forbidden."
    - "ADM POST sem motivo ou motivo<10 chars → HTTP 400 Bad Request."
  priority: P1
  risk: ALTO
```

---

## 4. Contexto `reconciliation`

### REQ-REC-001 · Importação e conciliação CNAB 240

```yaml
REQ-REC-001:
  context: reconciliation
  pattern: event-driven
  text: "Quando um arquivo CNAB 240 de retorno bancário é importado,
         o SIFAP deve processar apenas registros tipo '3' (detalhe),
         converter valores de centavos para Money (BR-002), e tentar casar
         cada registro com Payment usando as 3 chaves: numPgto + cpfBenef + competencia.
         Diferença de valor ≤ R$ 0,01 é considerada conciliada (BR-004)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L110-L152
  business_rule: BR-001, BR-002, BR-003, BR-004
  acceptance:
    - "Arquivo com 1 header + 100 detalhes + 1 trailer → 100 tentativas de match."
    - "Valor '0000000100050' (centavos) é convertido para Money(R$ 100,50)."
    - "Match com diff R$ 0,005 → conciliado (≤ R$ 0,01)."
    - "Match com diff R$ 0,02 → ReconciliationDiverged publicado, Payment NÃO atualizado."
    - "Match sem 3 chaves baterem → ReconciliationUnmatched publicado para análise manual."
  priority: P0
  risk: CRÍTICO
```

### REQ-REC-002 · Mapeamento código de retorno → status

```yaml
REQ-REC-002:
  context: reconciliation
  pattern: event-driven
  text: "Quando um registro CNAB é conciliado com sucesso, o SIFAP deve publicar
         PaymentReconciled com newStatus mapeado: '00'→'P', '01'→'D', '02'→'E'.
         Para qualquer outro código, deve publicar ReconciliationUnknownCode
         e marcar o Payment com flag 'requires_manual_review' (sem mudar status)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L160-L186
  business_rule: BR-005
  acceptance:
    - "Código '00' → status muda para 'P', flag não setada."
    - "Código '99' → status fica em 'G', flag 'requires_manual_review'=true, evento ReconciliationUnknownCode publicado."
    - "Resolve MYS-006: legado deixava log e seguia; agora há marca rastreável e fila de revisão manual."
    - "COD-BANCO recebido no arquivo é preservado (resolve MYS-005 — fim do hardcoded=1)."
  priority: P0
  risk: CRÍTICO
```

---

## 5. Contexto `audit`

### REQ-AUD-001 · Trilha append-only com hash-chain

```yaml
REQ-AUD-001:
  context: audit
  pattern: ubiquitous
  text: "O SIFAP deve gravar, em ordem cronológica e append-only,
         um registro de auditoria para cada evento de domínio publicado
         pelos contextos beneficiary, program-catalog, payment e reconciliation.
         Cada registro deve conter prevHash (SHA-256 do registro anterior)
         e rowHash (SHA-256 do conteúdo atual + prevHash) formando uma cadeia
         verificável."
  source_legacy: 01-arqueologia/legado-sifap/adabas-ddms/AUDITORIA.ddm
  business_rule: "BR-007 + N8 (greenfield: hash-chain)"
  acceptance:
    - "Evento BeneficiaryRegistered gera 1 linha em audit.event."
    - "UPDATE em audit.event é proibido (trigger BEFORE UPDATE RAISE EXCEPTION)."
    - "DELETE em audit.event é proibido (trigger BEFORE DELETE RAISE EXCEPTION)."
    - "GET /api/v1/audit/verify retorna OK se toda a cadeia hash bate; FAIL com índice da quebra caso contrário."
    - "Retenção: mínimo 5 anos (greenfield, alinhado a CGU)."
  priority: P0
  risk: CRÍTICO
  notes: "Resolve MYS-018 — exclusões agora aparecem na trilha (ação='EX' não é mais filtrada)."
```

### REQ-AUD-002 · Consulta com mascaramento por perfil

```yaml
REQ-AUD-002:
  context: audit
  pattern: state-driven
  text: "Enquanto o usuário consultando audit tiver perfil AUD, ADM ou SUP,
         o SIFAP deve retornar dados de PII (CPF, NIS) em formato original;
         para qualquer outro perfil, deve aplicar mascaramento (REQ-BEN-005)."
  source_legacy: 01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L98-L103
  business_rule: BR-007
  acceptance:
    - "GET /api/v1/audit/events?period=2026-05 com token AUD → CPF completo."
    - "Mesma chamada com token OPR → CPF mascarado XXX.XXX.NNN-NN."
    - "Toda consulta com unmask=true por AUD é ela mesma auditada (recursão de 1 nível)."
  priority: P0
  risk: ALTO
```

---

## 6. Atributos de Qualidade (NFRs)

| ID       | Atributo            | Meta                                                         | Como medir                                                 |
| -------- | ------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| NFR-001  | Latência p95 API    | < 300ms em listagens, < 500ms em consulta detalhada          | Application Insights + k6 no CI                            |
| NFR-002  | Throughput batch    | Ciclo mensal de 10k beneficiários em < 5 min (POC)           | Spring Batch metrics + log estruturado                     |
| NFR-003  | Cobertura testes    | ≥ 70% linhas em `payment` e `beneficiary`                    | JaCoCo no GitHub Actions                                   |
| NFR-004  | Diff vs legado      | `payment.cycle.diff_vs_legacy_centavos` = 0 em shadow run    | Métrica custom + alerta em > R$ 0,01 agregado              |
| NFR-005  | Segurança LGPD      | Zero CPF cru em logs, máscara universal (REQ-BEN-005)        | Log scanner CI (regex `\d{11}`)                            |
| NFR-006  | Disponibilidade     | 99,5% excluindo janela batch noturna                         | Azure Monitor health checks                                |
| NFR-007  | Auditoria íntegra   | Hash-chain verificável em < 30s para 1M registros            | GET /api/v1/audit/verify em CI nightly                     |

---

## 7. Rastreabilidade BR → REQ-ID

| BR legado                          | REQ-ID(s) cobrem                          | Status      |
| ---------------------------------- | ----------------------------------------- | ----------- |
| BR-001, BR-002, BR-003, BR-004     | REQ-REC-001                               | ✅ coberta  |
| BR-005, BR-006                     | REQ-REC-002, REQ-PAY-006                  | ✅ coberta  |
| BR-007                             | REQ-AUD-001, REQ-AUD-002                  | ✅ coberta  |
| BR-008, BR-009, BR-010             | REQ-PAY-006 / parcial (rel. fora desta v) | ⏸ parcial  |
| BR-011, BR-012, BR-023, BR-024     | REQ-PAY-001                               | ✅ coberta  |
| BR-013–BR-018                      | REQ-PAY-002                               | ✅ coberta  |
| BR-019                             | REQ-PAY-003                               | ✅ coberta  |
| BR-020                             | REQ-PAY-004                               | ✅ coberta  |
| BR-021, BR-022, BR-025–BR-028      | REQ-PAY-005 + ADR-0003                    | ✅ coberta  |
| BR-029–BR-032                      | —                                         | ⏸ Estágio 3 (correção monetária) |
| BR-035, BR-036, BR-037             | REQ-BEN-001, REQ-BEN-003, REQ-BEN-004     | ✅ coberta  |
| BR-038–BR-041                      | REQ-BEN-006 (parcial)                     | ⏸ parcial  |
| BR-042                             | REQ-PRG-001                               | ✅ coberta  |
| BR-043, BR-044, BR-045             | REQ-BEN-005                               | ✅ coberta (substituída) |
| BR-049, BR-053                     | REQ-BEN-002 (anti-padrão removido)        | ✅ removida |

**Cobertura:** 14 REQ-IDs cobrem ~80% das BRs críticas; restantes ficam para iteração no Estágio 3.

## 8. Cobertura `source_legacy:`

| Tipo de origem                                  | Quantidade |
| ----------------------------------------------- | ---------- |
| Aponta para `.NSN#Lxx-Lyy`                      | 11         |
| Aponta para `.ddm`                              | 1          |
| `[GREENFIELD]` com justificativa                | 2          |
| **Total**                                       | **14**     |

✅ 100% dos REQ-IDs têm `source_legacy:` rastreável (gate `legacy-traceability` passa).

## 9. Decisões diferidas para `/speckit.clarify`

- **D3 — MYS-003 (idade só por ano vs mês/dia exato).** Default em REQ-PAY-002: preservar legado.
- **D5 — Limite de dependentes 5 vs 10.** Default em REQ-BEN-006: 5.
- **D6 — MYS-022 (`COD-REG=99` pula validações).** Não modelado; trazer ao PO antes do Estágio 3.
- **D7 — Origem da constante `0.347215` (MYS-016).** Default em REQ-PRG-001: congelar valor.
