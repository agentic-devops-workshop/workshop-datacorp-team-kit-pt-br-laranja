<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Relatório de Descoberta — Estágio 1: Arqueologia Digital

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **discovery-report**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Este documento consolida todas as descobertas do Estágio 1.
> Preencha cada seção com as conclusões do time. **Este é o input principal do Estágio 2** — sem ele, a especificação vira chute.

**Time**: Os Laranjinhas
**Data**: 20/05/2026
**Edição**: Bruna
**Participantes**: 
Personas:
PAR1_VISÃO – Bruno (Product Owner + Requiremente Engineer)
Define o que precisa existir. Escreve specs EARS. Valida Critérios.
 
PAR2_ARQUITETURA – Pedro (Enterprise Architect + Software Architect)
Função: Desenha Modular Monolith, contratos entre módulos, ADRs.
 
PAR3_IMPLEMENTAÇÃO - Cleber(Technical Lead + Developer)
Função: Conduz Translation Agent. Revisa código gerado. Refatora onde precisa.
 
PAR4_QUALIDADE - Tiago (Qualidade – DBA + QA Engineer)
Função: Modela dados, escreve cenários BDD, garante cobertura e contratos.
 
PAR5_OPERAÇÃO – Bruna e Samuel (Tech Writer, DevOps Engineer)
Função: CI/CD, Terraform, ADRs, runbooks. Tudo que vai sustentar a evolução.

---

## 1. Sumário Executivo

> Em 3 a 5 frases, resuma o que o time descobriu sobre o SIFAP legado.
> O que é este sistema? Qual sua criticidade? Qual o estado do código?

O SIFAP é um sistema centralizado em mainframe para gestão de cadastro, processamento mensal de pagamentos de benefícios sociais, consultas e auditoria/fiscalização. Ele nasceu para substituir o SIPAG/DOS descentralizado e resolver consolidação nacional, duplicidade cadastral e integração financeira federal.
O sistema é bastante crítico, tem impacto social direto de escala nacional e processa dados de milhões de beneficiários.
O código parece ter sido bem estruturado para a tecnologia do legado, porém apresenta fragilidades técnicas como a divergência entre a a documentação e a implementação.
---

## 2. Visão Geral do Sistema

### 2.1 Propósito do SIFAP

O propósito do SIFAP é centralizar e controlar, em âmbito nacional, o cadastro e o pagamento de benefícios sociais federais, com processamento mensal em larga escala, conciliação financeira e trilha de auditoria/fiscalização.

### 2.2 Arquitetura Legada

A arquitetura do legado SIFAP é uma arquitetura mainframe monolítica, orientada a programas Natural, com processamento híbrido online + batch e persistência em Adabas.

Principais módulos:

Cadastro: manutenção de beneficiários, dependentes e programas.
Processamento: geração da folha mensal e consolidações batch.
Consulta: telas online de consulta operacional.
Auditoria: trilhas e relatórios de fiscalização.

DDMs principais: BENEFICIARIO (FNR 150), PROGRAMA-SOCIAL (FNR 151), PAGAMENTO (FNR 152), com inclusão posterior de AUDITORIA (FNR 153)

Fluxo operacional:

Online (3270): cadastro/consulta com validações e atualizações de DDMs.
Batch mensal: processamento da folha, geração de remessa, retorno bancário e conciliação.
Integrações externas: CNAB 240 (Banco do Brasil) e SIAFI,

### 2.3 Usuários e Perfis

Os usuários do SIFAP no legado são operadores em terminal 3270, analistas de sistemas, operação de mainframe, analistas de negócio e equipe DBA.
Os perfis de acesso identificados no legado usam os códigos ADM, OPR, CON, AUD e SUP (campo `COD-PERFIL` em `AUDITORIA`), com regras explícitas para ADMIN e SUPERVISOR em operações sensíveis.

| Grupo de Usuário | Perfil legado | Capacidades principais | Evidência |
| --- | --- | --- | --- |
| Operadores de atendimento/execução | OPR | Operação diária em telas 3270, consulta e execução de rotinas operacionais | `legacy-docs/MANUAL-TECNICO-SIFAP-2008.md` (diagrama com Operadores) |
| Analistas de sistemas SUPDE/DESIF | CON, OPR, SUP (conforme função) | Manutenção funcional/técnica e suporte evolutivo | `legacy-docs/MANUAL-TECNICO-SIFAP-2008.md` (público-alvo técnico) |
| Operação de mainframe | OPR, ADM | Execução e monitoramento de jobs batch, operação de ambiente | `legacy-docs/MANUAL-TECNICO-SIFAP-2008.md` |
| Analistas de negócio SENARC/CGPB | CON, AUD | Consulta técnica, auditoria funcional e validação de regras | `legacy-docs/MANUAL-TECNICO-SIFAP-2008.md` e `legacy-docs/ARQUITETURA-ORIGINAL-1997.md` |
| DBA Adabas | ADM, AUD | Administração de DDM/FDT, performance, integridade e suporte a auditoria | `legacy-docs/MANUAL-TECNICO-SIFAP-2008.md` |
| Administradores do sistema | ADM | Acesso restrito a funções críticas (ex.: administração de programas) | `legacy-docs/MANUAL-TECNICO-SIFAP-2008.md` |
| Supervisores | SUP | Autorização de operações sensíveis (ex.: alteração de CPF) | `legacy-docs/REGRAS-NEGOCIO-2012.md` |
| Auditoria/fiscalização | AUD | Consulta de trilhas e relatórios de auditoria por período/usuário | `legacy-docs/ARQUITETURA-ORIGINAL-1997.md` e `adabas-ddms/AUDITORIA.ddm` |

Perfis técnicos identificados no dado de auditoria: **ADM / OPR / CON / AUD / SUP** (campo `COD-PERFIL`).

---

## 3. Principais Descobertas

### 3.1 Regras de Negócio Críticas

> Liste as 5 regras de negócio mais importantes encontradas.

1. [Regra + referência ao catálogo BR-XXX]
2.
3.
4.
5.

### 3.2 Dependências Complexas

> Quais programas estão mais acoplados? Onde há risco de efeito cascata?

[Descreva]

### 3.3 Dívida Técnica Identificada

> Que problemas no código legado vão complicar a migração?

- [ ] [Problema 1]
- [ ] [Problema 2]
- [ ] [Problema 3]

### 3.4 Gaps de Documentação

> O que a documentação existente NÃO cobre?

[Descreva]

---

## 4. Mistérios e Riscos

### 4.1 Mistérios Não Resolvidos

> Resuma os mistérios do arquivo `mysteries-found.md` que permanecem sem explicação.

| ID  | Descrição | Risco para Migração |
| --- | --------- | ------------------- |
|     |           |                     |

### 4.2 Riscos para o Estágio 2

> O que o time de especificação precisa saber antes de começar?

1. [Risco 1]
2. [Risco 2]
3. [Risco 3]

---

## 5. Recomendações

### 5.1 O que migrar primeiro

> Com base na priorização do Par 1 (Product Owner), quais funcionalidades devem ser migradas primeiro?

| Prioridade | Funcionalidade | Justificativa |
| ---------- | -------------- | ------------- |
| 1          |                |               |
| 2          |                |               |
| 3          |                |               |

### 5.2 O que descartar

> Funcionalidades que existem por causa da **tecnologia** (Adabas/3270/JCL) e não da política pública. Não migrar — reescrever do zero na stack moderna.

- **Telas 3270 / MAPs do Com\*plete** (CONSBENF, CADBENEF, CADPROG): UX obsoleta, sem acessibilidade nem mobile. Substituir por frontend Next.js 15 + shadcn/ui.
- **Relatórios impressos em spool JES2 formato 132 colunas** (RELPGT, RELAUDIT): saída para impressora matricial sem valor hoje. A *lógica* de agregação migra; o *formato* não. Substituir por endpoints REST + export PDF/CSV sob demanda.
- **Campos `MU` (multi-value) e `PE` (periodic group) dos DDMs** (PAGAMENTO, BENEFICIARIO): anti-padrão relacional herdado do Adabas. Não cabe em PostgreSQL. Substituir por tabelas filhas normalizadas (`pagamento_parcela`, `beneficiario_endereco_historico`).
- **ISN / FNR (150-153) como identificadores físicos** (todos os DDMs): conceito Adabas-específico que não deve vazar para o domínio moderno. Substituir por UUID/BIGSERIAL no PostgreSQL.
- **Mascaramento de CPF feito no `DISPLAY` da tela** (CONSBENF): workaround de apresentação, não privacidade. LGPD-by-design exige masking no service layer + RBAC, não na renderização.
- **Orquestração batch via JCL/JES2** (BATCHCON, BATCHPGT, BATCHREL): a *lógica* batch migra; o *orquestrador* não. Substituir por Spring Batch + cron K8s / Azure Container Apps Jobs.
- **"Autenticação" implícita por USER-ID do terminal Natural**: não é autenticação — é só identificação de sessão TP. Sem MFA, sem token, sem expiração. Substituir por OAuth2/OIDC + JWT (Spring Security).
- **Sub-rotinas `PERFORM` de validação duplicadas** (VALDOCS / VALBENEF — evidência em BR-027 e BR-031): consolidar em um único `BeneficiarioValidator` no domain layer. Modernizar copiando duplicação é desperdício.
- **Cálculo de correção monetária com índices legados** (CALCCORR — TR/IGP-M congelados de planos econômicos antigos): validar com PO se ainda há aplicação retroativa válida; caso negativo, descartar.
- **Códigos de programas sociais hardcoded em `IF/ELSE` Natural** (CADPROG, VALELEG): regras engessadas em código exigem deploy para qualquer mudança. Substituir por tabela de configuração parametrizável — as BR-021 a BR-035 viram **dados**, não código.
- **Mistérios catalogados em `mysteries-found.md` sem origem conhecida**: regras cuja razão ninguém mais lembra não devem ser migradas até alguém explicar para que servem. Migrar mistério = perpetuar dívida técnica.

### 5.3 O que evoluir

> Funcionalidades essenciais da política pública que **devem ser migradas E melhoradas** na modernização.

- **Cálculo de benefício** (CALCBENF, BR-021/BR-022): migrar a fórmula, mas externalizar parâmetros (faixas, valores, multiplicadores) em tabela de configuração versionada. Validar com SENARC se a regra vigente em 2026 ainda corresponde à do legado.
- **Desconto em folha** (CALCDSCT): migrar lógica, mas mover a tabela de descontos para configuração parametrizável e adicionar histórico de alterações.
- **Cadastro de beneficiários e dependentes** (CADBENEF, CADDEPEND): migrar modelo, normalizar dados (sem MU/PE), adicionar validação por CPF na Receita, LGPD-by-design (masking + consent log) e limites configuráveis (não hardcoded).
- **Validação de elegibilidade** (VALELEG, BR-024/BR-025): consolidar com VALBENEF/VALDOCS num único motor de regras parametrizável, com explainability (qual regra reprovou e por quê).
- **Geração de ciclo de pagamento** (BATCHPGT, BR-029/BR-030): migrar a lógica como job Spring Batch, mas com idempotência, retry automático, dead-letter queue, observabilidade (OpenTelemetry) e dry-run obrigatório antes de execução produtiva.
- **Trilha de auditoria** (RELAUDIT, BR-032 a BR-035, DDM AUDITORIA adicionado em 2005): migrar o conceito, mas evoluir para audit log estruturado (JSON) com contexto completo de request, hash de integridade encadeado, retenção configurável e integração nativa com SIEM.
- **Consulta de beneficiário** (CONSBENF): migrar regras de negócio (BR-023, BR-026), mas reimplementar como API REST + frontend moderno, com paginação, filtros server-side e cache de leitura.
- **Relatórios gerenciais** (RELPGT, BR-027/BR-028/BR-031): preservar agregações e regras de exibição, mas evoluir para dashboards interativos (drill-down, export sob demanda) em vez de spool batch fixo.
- **Autorização de operações sensíveis** (perfil SUPERVISOR para alteração de CPF, etc.): manter a regra (dupla aprovação), mas evoluir para workflow auditável com notificação, justificativa obrigatória e expiração da autorização.

---

## 6. Métricas do Estágio

| Métrica                       | Valor        |
| ----------------------------- | ------------ |
| Programas analisados          | 15 / 15  |
| DDMs mapeados                 | 4 / 4   |
| Regras de negócio encontradas | \_\_\_       |
| Regras escondidas encontradas | \_\_\_ / 10  |
| Easter eggs encontrados       | \_\_\_ / 3   |
| Termos no glossário           | 70       |
| Mistérios catalogados         | \_\_\_       |
| Tempo total gasto             | \_\_\_ horas |

---

## 7. Notas para o Próximo Estágio

> Deixe aqui mensagens para o time no Estágio 2 (Especificação Moderna):

[Escreva aqui]

---

## Definição de Pronto deste relatório

- [ ] Todas as seções acima preenchidas (sem placeholders).
- [ ] Pelo menos 5 regras críticas listadas em §3.1, cada uma referenciando uma `BR-XXX` do catálogo.
- [ ] Decisões de migrar/descartar/evoluir em §5 cobrem as 8+ funcionalidades principais.
- [ ] Métricas de §6 conferem com os outros artefatos (glossary.md, business-rules-catalog.md, mysteries-found.md).

— Paula


---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="mysteries-found.md"><strong>mysteries-found.md</strong></a><br/>
<sub>Lista de mistérios.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="../02-spec-moderna/GUIDE.md"><strong>Estágio 2 — Spec</strong></a><br/>
<sub>Próximo estágio: spec moderna.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="../README.md">Voltar ao Kit PT-BR</a></sub>

