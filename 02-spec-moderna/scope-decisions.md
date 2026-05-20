<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Decisões de Escopo — SIFAP 2.0

![ESTÁGIO 02 Spec](https://img.shields.io/badge/ESTÁGIO-02%20Spec-00A4EF?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S2](https://img.shields.io/badge/PREENCHA-Durante%20S2-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 2](README.md) → **Scope Decisions**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 2 (Spec Moderna).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento preenchido para sua feature
> 2. Rastreabilidade `source_legacy:` para cada REQ-ID
> 3. Sign-off do Product Owner antes da passagem H2
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Para cada funcionalidade encontrada no Estágio 1, decida: **Migrar**, **Descartar** ou **Evoluir**.
>
> - **Migrar**: trazer para o SIFAP 2.0 como está (mesma lógica, nova tecnologia)
> - **Descartar**: não trazer — funcionalidade obsoleta ou desnecessária
> - **Evoluir**: trazer E melhorar (nova UX, novo fluxo, nova capacidade)

**Time**: Os Laranjinhas
**Data**: 20/05/2026
**Edição**: Bruna
**Par 1 (Product Owner) responsável**: Bruno

## Por que isso importa

O escopo é o que protege o time de chegar às 17h00 com 12 features pela metade. Se o Par 1 não cortar, o Estágio 3 não fecha. **Decisão difícil é tomada aqui, não no Estágio 3.**

## Como decidir

Pergunte de cada funcionalidade:

1. **Afeta o ciclo mensal de pagamento?** Sim → Migrar. Não → considere descartar.
2. **Tem uso documentado nos últimos 12 meses?** Não → descartar.
3. **Faz parte de um relatório regulatório obrigatório (TCU, CGU, BB)?** Sim → Migrar como está.
4. **Tem uma versão moderna mais barata de implementar?** Sim → Evoluir.

---

## Decisões por Funcionalidade

| #   | Funcionalidade legado                                              | Decisão        | Justificativa                                                                                                                  | Regra(s) de Negócio                       | Prioridade |
| --- | ------------------------------------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------- | ---------- |
| 1   | Cadastro de Beneficiários (CADBENEF + CADDEPEND)                   | **Evoluir**    | Migrar regras, mas reforçar LGPD (mascaramento, consentimento) e parametrizar limite de dependentes (MYS-015)                  | BR-035, BR-036, BR-037, BR-038, BR-041    | Alta       |
| 2   | Consulta de Beneficiários (CONSBENF)                               | **Evoluir**    | Substituir máscara de CPF bugada (MYS-017) por implementação LGPD-compliant; manter histórico de 12 pagamentos (BR-045)        | BR-043, BR-044, BR-045                    | Alta       |
| 3   | Validações de Documento e Elegibilidade (VALBENEF + VALDOCS + VALELEG) | **Evoluir**    | Consolidar em **motor único** de validação no contexto `beneficiary` — elimina duplicação BR-027/031 e backdoors de CPF (D1)   | BR-035, BR-049, BR-053, BR-058            | Alta       |
| 4   | Cálculo de Benefício (CALCBENF)                                    | **Migrar**     | Replicar **bit-a-bit** a fórmula-mãe BR-017 com shadow test contra legado por 3 ciclos consecutivos antes do cutover           | BR-011 a BR-024, BR-033                   | Alta       |
| 5   | Cálculo de Descontos (CALCDSCT)                                    | **Migrar**     | Eleger CALCDSCT como **fonte única de verdade** de desconto (resolve MYS-024); preservar exceção judicial sem teto (D4/BR-025) | BR-025, BR-026, BR-027, BR-028            | Alta       |
| 6   | Correção Monetária (CALCCORR)                                      | **Evoluir**    | Migrar lógica mas tirar tabela IPCA hardcoded até 2014 — passar para tabela externa parametrizável (MYS-023, dívida técnica)   | BR-029, BR-030, BR-031, BR-032            | Alta       |
| 7   | Conciliação Bancária (BATCHCON)                                    | **Evoluir**    | Multi-banco real (fim do `COD-BANCO=1` hardcoded MYS-005); tratar códigos de retorno desconhecidos sem deixar pagamento fantasma (MYS-006) | BR-001 a BR-007                           | Alta       |
| 8   | Folha Mensal Batch (BATCHPGT)                                      | **Migrar**     | Reimplementar como Spring Batch com idempotência reforçada (BR-012) e checkpoint resumível — orquestração Argo/Airflow opcional | BR-011, BR-012, BR-023, BR-024            | Alta       |
| 9   | Relatórios (BATCHREL + RELPGT)                                     | **Evoluir**    | Migrar mas **alinhar política financeira** com BATCHPGT (resolve MYS-001 — round vs truncate); exportar PDF + CSV               | BR-008, BR-009, BR-010                    | Média      |
| 10  | Auditoria (AUDITORIA + RELAUDIT)                                   | **Evoluir**    | Append-only com hash-chain; **incluir exclusões** no relatório (fim do MYS-018, conformidade LGPD/CGU); reter 5 anos             | BR-007 (e novas regras LGPD)              | Alta       |
| 11  | Cadastro de Programas Sociais (CADPROG)                            | **Evoluir**    | Parametrizar constante mágica `0.347215` (MYS-016) e eliminar IDs hardcoded; CRUD com versionamento de programa                | BR-042                                    | Média      |
| 12  | CPFs de teste em produção (backdoors EGG-002)                      | **Descartar**  | NÃO migrar (D1); substituir por ambiente de homologação isolado com dados sintéticos. Risco de fraude inaceitável               | BR-049, BR-053 (anti-padrão)              | Alta       |
| 13  | Plano Verão 1989 (CALCCORR dead code MYS-023 / EGG-001)            | **Descartar**  | Inativo há 25+ anos; ADR-004 (Strangler) documenta a remoção                                                                    | —                                         | Baixa      |
| 14  | Integração Banco Real comentada (BATCHCON MYS-004)                 | **Descartar**  | Banco Real adquirido pelo Santander em 2007; código morto                                                                       | —                                         | Baixa      |
| 15  | `IND-CORRIGIDO` flag de não-recálculo (BR-029)                     | **Migrar**     | Garantia de idempotência da correção retroativa — preservar                                                                     | BR-029                                    | Média      |

---

## Funcionalidades Novas (não existem no legado)

> Liste funcionalidades que o SIFAP 2.0 deveria ter e que não existem no sistema legado. Cada uma vira REQ-ID com `source_legacy: [GREENFIELD] <justificativa>`.

| #   | Funcionalidade Nova                                                                | Justificativa                                                                                              | Prioridade | Complexidade |
| --- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ---------- | ------------ |
| N1  | API REST `/api/v1/*` (substitui telas 3270 do Com-plete)                           | Habilita frontend Next.js e integrações modernas; legado usa terminal 3270 acoplado ao mainframe           | Alta       | Média        |
| N2  | Frontend Next.js 15 (App Router) para cadastro/consulta/aprovação                  | Substitui 3270; melhor UX, acessibilidade WCAG 2.1, suporte mobile para fiscais em campo                   | Alta       | Média        |
| N3  | OAuth2 + Azure AD com perfis ADM/OPR/CON/AUD/SUP em scope claim                    | Legado usa autorização interna do Com-plete; AAD permite SSO federado com SERPRO e auditoria centralizada  | Alta       | Média        |
| N4  | Observabilidade (OpenTelemetry, structured JSON logs, métricas Prometheus)         | Legado depende de SYSOUT/spool; impossível detectar latência por endpoint ou taxa de erro                  | Alta       | Baixa        |
| N5  | Métrica `payment.cycle.diff_vs_legacy_centavos` durante o Strangler                | Guard-rail de cutover — alerta se diferença vs legado ultrapassar R$ 0,01 por pagamento                    | Alta       | Média        |
| N6  | Ambiente de homologação com CPFs sintéticos (substitui EGG-002)                    | Mantém capacidade de teste em produção-like sem risco de fraude por backdoor                              | Alta       | Baixa        |
| N7  | Mascaramento de CPF em todos os logs (`XXX.XXX.NNN-NN`)                             | LGPD Art. 6º (minimização) — legado pré-data LGPD (1996 vs 2018)                                          | Alta       | Baixa        |
| N8  | Hash-chain de auditoria (append-only com SHA-256 encadeado)                        | Garantia criptográfica de não-adulteração — eleva trilha CGU/TCU                                          | Média      | Média        |
| N9  | Esteira CI/CD GitHub Actions + Terraform Azure                                     | Legado tem deploy manual via JCL; cutover só é viável com automação                                       | Alta       | Média        |
| N10 | Tabela externa de índices de correção monetária (TR, IPCA, IGP-M)                  | Resolve MYS-023 (tabela hardcoded até 2014); permite atualização sem novo deploy                          | Média      | Baixa        |

---

## Resumo de Escopo

| Decisão       | Quantidade legado | Greenfield novo | Percentual sobre legado |
| ------------- | ----------------- | --------------- | ----------------------- |
| Migrar        | 4                 | —               | 27%                     |
| Evoluir       | 8                 | —               | 53%                     |
| Descartar     | 3                 | —               | 20%                     |
| Greenfield N* | —                 | 10              | n/a                     |
| **Total**     | **15**            | **10**          | **100% do legado**      |

## Riscos de Escopo

> Riscos das decisões tomadas acima:

| #  | Risco                                                                                                  | Probabilidade | Impacto | Mitigação                                                                                            |
| -- | ------------------------------------------------------------------------------------------------------ | ------------- | ------- | ---------------------------------------------------------------------------------------------------- |
| R1 | Fórmula BR-017 reimplementada com divergência de centavos vs legado                                     | Alta          | Alto    | Shadow run de 3 ciclos mensais com métrica N5; cutover só com diff = R$ 0,00 em amostra de 100k     |
| R2 | Descartar backdoors CPF (D1) bloqueia testes de smoke em produção que dependem deles hoje              | Média         | Alto    | N6: ambiente de homologação com CPFs sintéticos pré-criados antes do cutover                         |
| R3 | Consolidar VAL* num motor único quebra fluxos que dependem de validações sutilmente diferentes         | Média         | Médio   | Mapear cada chamada em VALBENEF/VALDOCS/VALELEG e cobrir com teste de equivalência antes da fusão    |
| R4 | Parametrizar `0.347215` (MYS-016) sem entender origem causa cálculo errado em programas futuros        | Alta          | Alto    | Bloquear no Estágio 2 com pergunta ao PO/atuária; congelar valor se origem não confirmada            |
| R5 | Migrar BATCHPGT para Spring Batch ultrapassa janela de 3h do Estágio 3                                  | Alta          | Médio   | Fatiar: Estágio 3 entrega job de **1 programa social** com 1k beneficiários; escala vem no Estágio 4 |
| R6 | Decisão de "corrigir vs preservar" MYS-003 (idade) tira fator 1.15 de milhões em produção              | Baixa         | Alto    | Aprovação explícita do PO antes de qualquer mudança; default = preservar comportamento legado        |
| R7 | Tabela externa de índices (N10) não estar pronta a tempo do cutover de CALCCORR                        | Média         | Médio   | Carregar inicialmente com dump do hardcode; backlog separado para painel administrativo              |

## Decisões críticas pendentes para H2 (16:00)

> Estas 4 decisões precisam de sign-off explícito do PO antes de fechar o Estágio 2. Cada uma vira ADR ou nota no REQ-ID correspondente.

| ID | Decisão                                                                                       | Origem                  | Responsável  | Status   |
| -- | --------------------------------------------------------------------------------------------- | ----------------------- | ------------ | -------- |
| D1 | Descartar backdoors de CPF (MYS-019/021, EGG-002); criar ambiente de homologação isolado     | mysteries-found.md      | PO + Security| Proposto |
| D2 | Política financeira única: HALF_EVEN bancário, 2 casas decimais (resolve MYS-001)             | MYS-001, BR-009, BR-018 | PO + DBA     | Proposto |
| D4 | Desconto judicial sem teto confirmado como invariante legal (BR-025)                          | MYS-013                 | PO + Jurídico| Proposto |
| D8 | `CALCDSCT` é fonte única de desconto; `CALCBENF` deixa de aplicar inline (resolve MYS-024)    | MYS-024                 | SA           | Proposto |

**Decisões diferidas para `/speckit.clarify` no Estágio 3:** D3 (idade ano vs dia), D5 (limite 5 vs 10 dependentes), D6 (`COD-REG=99`), D7 (constante `0.347215`).

## Aprovação

- [x] Par 1 (Product Owner — Bruno) aprovou as decisões de escopo
- [x] Par 2 (Enterprise Architect — Pedro) validou a viabilidade técnica
- [x] Par 3 (Technical Lead — Cleber) confirmou que cabe nas 3 horas do Estágio 3 (com R5 mitigado por fatiamento)
- [x] Time concordou com as prioridades

> **Aprovação registrada na Passagem #2** (~16:00). Estágio 3 liberado.

— Os Laranjinhas (20/05/2026)


---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="GUIDE.md"><strong>GUIDE do Estágio 2</strong></a><br/>
<sub>Passo a passo do estágio.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="ADR-TEMPLATE.md"><strong>ADR-TEMPLATE</strong></a><br/>
<sub>Template de ADR.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="../README.md">Voltar ao Kit PT-BR</a></sub>

