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

> Top 5 regras selecionadas de [`business-rules-catalog.md`](business-rules-catalog.md) (59 catalogadas, 34 críticas). Critério: impacto financeiro direto + risco de migração + cobertura obrigatória nas EARS do Estágio 2.

1. **BR-017 — Fórmula-mãe do cálculo de benefício**: `VLR-BASE × fator_reg × fator_fam × fator_renda × fator_idade × (1 + fator_reajuste)` em `BATCHPGT.NSN#L245-L250`. Coração do sistema; precisa ser replicada **bit-a-bit** na modernização, com testes de equivalência contra o legado em 3 ciclos consecutivos antes do cutover.
2. **BR-011 — Elegibilidade só para ativos** (`BENEFICIARIO.STATUS='A'` E `PROGRAMA-SOCIAL.STATUS-PROG='A'`) em `BATCHPGT.NSN#L150-L192`. Validação dupla obrigatória — reforçada por BR-033 e BR-056. Falha aqui gera pagamento indevido.
3. **BR-012 — Idempotência mensal** em `BATCHPGT.NSN#L155-L165`: não gera novo pagamento se já existe um para o mesmo CPF na mesma competência. Garantia anti-duplicidade do batch — sem isso, reexecução do job paga dobrado.
4. **BR-005 — Mapeamento de código de retorno bancário → status do pagamento** (`'00'→'P'`, `'01'→'D'`, `'02'→'E'`) em `BATCHCON.NSN#L160-L182`. Núcleo da conciliação financeira; códigos fora desse domínio caem em log silencioso e o pagamento fica em status anterior — fonte de inconsistência crítica.
5. **BR-049 + BR-053 — Backdoors de CPF de teste** (`VALBENEF.NSN#L218-L234` e `VALDOCS.NSN#L36-L43, L171-L188`). CPFs com 11 dígitos iguais começando em `000` são aceitos, e 8 prefixos (`000, 001, 002, 010, 011, 099, 100, 999`) pulam validação de dígito. **Risco de segurança CRÍTICO** — não migrar; substituir por ambiente de homologação isolado com dados sintéticos.

### 3.2 Dependências Complexas

> Quais programas estão mais acoplados? Onde há risco de efeito cascata?

**Observação estrutural:** não foram encontradas chamadas `CALLNAT` entre os 15 programas — todo o acoplamento entre rotinas ocorre via `PERFORM` (sub-rotinas internas) e, principalmente, via **acesso compartilhado aos mesmos DDMs Adabas**. O acoplamento é, portanto, **por dado**, não por contrato — o pior tipo para refatoração.

**Pontos de maior acoplamento e risco de efeito cascata:**

- **DDM `PAGAMENTO` (FNR 152, ~180M registros) — hub de acoplamento #1.** Lido/escrito por `BATCHPGT`, `CALCBENF`, `CALCCORR`, `CALCDSCT`, `CONSBENF`, `RELPGT`, `RELAUDIT`. Qualquer mudança em campo (especialmente os `MU` de parcelas) afeta 7 programas simultaneamente. **Risco: alto.**
- **DDM `BENEFICIARIO` (FNR 150, ~4,2M registros) — hub #2.** Acessado por `CADBENEF`, `CADDEPEND`, `VALBENEF`, `VALELEG`, `VALDOCS`, `CONSBENF`, `CALCBENF`, `BATCHCON`. Campos `PE` (histórico de endereços) bloqueiam normalização. **Risco: alto.**
- **Validações duplicadas em `VALDOCS` e `VALBENEF`** (evidência: BR-027 e BR-031 catalogadas como duplicação). Mesma regra implementada em dois lugares — corrigir uma sem a outra gera divergência silenciosa. **Risco: médio-alto.**
- **Cadeia batch `BATCHCON → BATCHPGT → BATCHREL`** com dependência temporal implícita (não documentada). Se `BATCHCON` falha, `BATCHPGT` roda com dado inconsistente; se `BATCHPGT` não termina, `BATCHREL` gera relatório incompleto. Sem orquestrador formal — depende de JCL/JES2. **Risco: alto para o cutover.**
- **`CALCCORR` (correção monetária) chamado via `PERFORM` por `CALCBENF` e por `BATCHPGT`** com regras de índice (TR/IGP-M) hardcoded. Mudança de fórmula impacta dois fluxos diferentes. **Risco: médio.**
- **Campo `COD-PERFIL` em `AUDITORIA` (FNR 153)** referenciado em `RELAUDIT` e implicitamente em todos os programas que registram trilha. Adicionado em 2005 sem retrofit nos programas anteriores — coexistem registros com e sem perfil. **Risco: médio (qualidade de dado).**
- **Códigos hardcoded de programas sociais** em `CADPROG` e `VALELEG` (IF/ELSE com IDs literais). Adicionar novo programa social exige tocar dois `.NSN` + recompilar. **Risco: médio (rigidez).**
- **Identificadores físicos Adabas (ISN/FNR) vazando para a lógica de negócio** em vários programas. Migrar para PostgreSQL exige remapear chaves em todos os pontos. **Risco: alto (efeito cascata na migração de dados).**

**Conclusão:** o sistema é monolítico-modular no papel, mas **fortemente acoplado por dado** (DDMs compartilhados) no concreto. Bounded contexts modernos terão que ser definidos com base em **dono do dado**, não em similaridade de função.

### 3.3 Dívida Técnica Identificada

> Que problemas no código legado vão complicar a migração?

- [x] Estrutura de dependência entre os programas
- [x] Mistérios de documentação
- [x] Desatualização temporal de regras de negócio

### 3.4 Gaps de Documentação

### 3.4 Gaps de Documentação

> O que a documentação existente NÃO cobre?

A documentação em `legacy-docs/` contém apenas três artefatos (`ARQUITETURA-ORIGINAL-1997.md`, `MANUAL-TECNICO-SIFAP-2008.md`, `REGRAS-NEGOCIO-2012.md`) e está **defasada em 13 anos** — última versão de 2012, nenhuma atualização até 2026. Lacunas identificadas:

**Lacunas críticas (bloqueadoras para o Estágio 2):**

- **Sem documentação pós-2012.** Toda mudança de regra de negócio entre 2012 e 2026 existe apenas no código `.NSN`. Não há changelog nem registro de versões.
- **Sem call graph real.** O MANUAL-TECNICO descreve módulos, mas não o grafo de chamadas. Descobrimos por leitura de código que **não há `CALLNAT`** — apenas `PERFORM` interno e acoplamento por DDM — e isso não está documentado em lugar nenhum.
- **FDT incompleto.** Os DDMs são listados, mas tipos de descritor `MU` (multi-value) e `PE` (periodic group) não são detalhados — justamente os anti-padrões que mais impactam a migração para PostgreSQL.
- **Sem volumetria oficial.** Estimativas de ~4,2M beneficiários e ~180M pagamentos vêm da nossa análise, não de fonte oficial. Sem números reais, não é possível dimensionar infra Azure nem planejar migração de dados.
- **Sem SLAs nem janelas batch.** Não há documentação de duração esperada de `BATCHCON`/`BATCHPGT`/`BATCHREL`, nem dependências formais entre jobs. Crítico para projetar Spring Batch + scheduling moderno.
- **Sem integrações externas mapeadas.** A documentação não menciona SIAFI/SICAF/Receita ou qualquer sistema upstream/downstream. Improvável que SIFAP rode isolado.
- **Matriz de autorização incompleta.** Perfis ADM/OPR/CON/AUD/SUP aparecem isoladamente, mas não existe matriz **perfil × operação × tela**. Descobrimos regras de autorização (BR-026, BR-035) apenas no código.

**Regras de negócio órfãs (existem no código, ausentes na doc):**

- Fórmula de **correção monetária** (`CALCCORR`) — TR/IGP-M sem regra documentada.
- **Mascaramento de CPF** no `DISPLAY` (`CONSBENF`) — privacidade pré-LGPD não documentada.
- **Hash de integridade / encadeamento de auditoria** (se existir no `AUDITORIA` pós-2005).
- **Tratamento de erro e códigos de retorno** dos jobs batch — o que fazer se `BATCHPGT` falha no meio?
- **Política de retenção** dos 180M de pagamentos — não há regra de arquivamento/expurgo documentada.
- **Duplicação de regras** (BR-027, BR-031) — a doc não admite que `VALDOCS` e `VALBENEF` validam o mesmo dado.
- **Limites hardcoded** (qtd máx. de dependentes, valor máx. de benefício) — existem no código, ausentes em `REGRAS-NEGOCIO-2012`.

**Lacunas operacionais:**

- Sem **runbooks** (o que fazer quando job batch falha às 3h da manhã).
- Sem plano de **disaster recovery / backup** do Adabas — risco direto para o cutover.
- Sem **métricas de uso** (beneficiários ativos, consultas/dia) — sem baseline para comparar pós-migração.
- **Glossário de domínio ausente** na doc original (resolvido no Estágio 1 em [`glossary.md`](glossary.md)) — termos como "benefício suspenso" vs "cancelado" vs "inativo" não eram definidos.

**Lacunas históricas:**

- Sem registro de **por que** o DDM `AUDITORIA` só foi adicionado em 2005 (qual incidente motivou?).
- Sem **ADRs históricos** explicando decisões (por que Natural? por que Adabas? alternativas descartadas?).
- Sem **post-mortems** de incidentes — lições aprendidas perdidas.

Estas lacunas alimentam diretamente o [`mysteries-found.md`](mysteries-found.md) e são insumo formal para perguntas a SENARC/CGPB antes do Estágio 2.

---

## 4. Mistérios e Riscos

### 4.1 Mistérios Não Resolvidos

> Resumo dos mistérios catalogados em [`mysteries-found.md`](mysteries-found.md) (24 catalogados, 19 com confiança ALTA). Selecionados os de maior risco para migração.

| ID  | Descrição | Risco para Migração |
| --- | --------- | ------------------- |
| MYS-001 | `BATCHREL` arredonda valor bruto (`+0.005`) mas `BATCHPGT` trunca o mesmo valor — totais do relatório divergem do somatório real | **CRÍTICO** — reconciliação contábil falha; auditoria identifica diferença de centavos em milhares de pagamentos |
| MYS-003 | Cálculo de idade ignora mês/dia (`ano - ano_nasc`); beneficiário "vira" idoso até 12 meses antes | **ALTO** — "consertar" na modernização tira fator 1.15 de milhões de beneficiários por 1 mês; validar com PO |
| MYS-006 | Códigos de retorno bancário ≠ `00/01/02` apenas geram log; status do pagamento fica em `'G'` para sempre | **ALTO** — pagamentos fantasma silenciosos; investigar quantos existem hoje antes do cutover |
| MYS-013 | Desconto judicial NÃO respeita teto de 30% e pode zerar o líquido | **CRÍTICO** — conformidade legal vs ordem judicial; aplicar teto = descumprir decisão |
| MYS-014 | `IF #IDADE > 75 MOVE 'S' TO #STATUS` — beneficiário recém-cadastrado é suspenso silenciosamente | **CRÍTICO** — política demográfica não documentada; pode ser discriminação etária ou regra legítima |
| MYS-015 | Limite hardcoded de 5 dependentes em `CADDEPEND` contradiz DDM (que permite 10) | **ALTO** — outro programa pode preencher slots 6-10? Migrar para 5 ou 10? |
| MYS-016 | Constante mágica `0.347215` no fator de reajuste de `CADPROG` sem origem documentada | **ALTO** — coeficiente atuarial ou inflação histórica? Não dá pra parametrizar sem entender |
| MYS-017 | Comentário "INCONSISTENCIA CONHECIDA — NAO CORRIGIR SEM APROVACAO DA AUDITORIA" em `CONSBENF` | **CRÍTICO** — algum sistema externo depende do bug? Investigar antes de qualquer mudança |
| MYS-018 | `IF AUDITORIA-V.ACAO='EX' ESCAPE TOP` — exclusões somem do relatório de auditoria | **CRÍTICO** — compliance LGPD/CGU: trilha deveria mostrar tudo; quem decidiu ocultar? |
| MYS-019 + MYS-021 | Backdoors de CPF (`000` 11x iguais + 8 prefixos sem dígito): EGG-002 | **CRÍTICO** — risco de fraude e divergência fiscal; NÃO migrar — substituir por ambiente isolado com dados sintéticos |
| MYS-020 | Tabela `DIAS-MES(2)=29` fixa ignora regra de ano bissexto (4/100/400) | **MÉDIO** — permite cadastrar 29/02 em ano não bissexto; corrigir na modernização |
| MYS-022 | `IF #COD-REG=99 → ELEGIVEL=TRUE` pula TODAS as validações de elegibilidade | **CRÍTICO** — diplomatas/convênios ou backdoor? Validar com SENARC antes de migrar |
| MYS-024 | Duas implementações de desconto: `CALCBENF` inline vs `CALCDSCT` formal | **ALTO** — qual é fonte da verdade? Há divergência de valor entre eles? Consolidar antes do EARS |

**Cobertura:** 13 mistérios selecionados (de 24 totais). Os 11 restantes (MYS-002, 004, 005, 007 a 012, 023) têm risco menor mas estão no catálogo para o Estágio 2. Easter eggs encontrados: **2 de 3** (EGG-001 Plano Verão 1989, EGG-002 backdoor de CPFs).

### 4.2 Riscos para o Estágio 2

> O que o time de especificação precisa saber antes de começar o Spec-Driven Development (EARS + ADRs).

1. **Acoplamento por dado, não por contrato.** O legado não tem `CALLNAT` — só `PERFORM` e DDMs compartilhados. Bounded contexts NÃO podem ser recortados por similaridade de função; precisam ser definidos por **dono do dado** (quem escreve em `PAGAMENTO`? quem é fonte da verdade de `BENEFICIARIO`?). Risco: escrever EARS por módulo legado gera microsserviços anêmicos e transações distribuídas desnecessárias.
2. **Cobertura `source_legacy:` ameaçada por documentação defasada.** A doc em `legacy-docs/` parou em 2012. 13 anos de regra de negócio só existem no `.NSN`. Cada EARS DEVE apontar para `01-arqueologia/legado-sifap/natural-programs/*.NSN` ou DDM — apontar apenas para `legacy-docs/*.md` é insuficiente porque a doc não reflete o estado real. O CI `legacy-traceability` vai rejeitar PRs.
3. **Regras de negócio órfãs precisam de decisão antes de virar EARS.** Correção monetária (CALCCORR), mascaramento de CPF, política de retenção, tratamento de erro batch — não estão documentadas. Não escreva EARS chutando; abra ADR de "regra órfã" ou marque como `[GREENFIELD] + justificativa` com aprovação do PO.
4. **Duplicação detectada (BR-027, BR-031) é armadilha.** `VALDOCS` e `VALBENEF` validam a mesma coisa. Se o Requirements Engineer transcrever as duas para EARS distintas, perpetua a duplicação no modelo moderno. Consolidar exige decisão arquitetural ANTES da spec — abrir ADR de "motor único de validação".
5. **Volumetria não é oficial.** Estimativas (~4,2M beneficiários, ~180M pagamentos) vieram da nossa análise, não de fonte SENARC/CGPB. NFRs de performance/throughput nas EARS vão sair errados sem confirmar número real. Bloqueador para Software Architect dimensionar Azure.
6. **SLAs e janelas batch ausentes.** Não há documentação de quanto tempo `BATCHPGT` leva nem dependências formais com `BATCHCON`/`BATCHREL`. EARS de NFR para o ciclo mensal precisam dessas métricas. Escalar para operação antes de escrever as specs de batch.
7. **Identificadores físicos Adabas (ISN/FNR) vazaram para a lógica.** Cuidado ao escrever EARS que mencionem chaves — use linguagem de domínio (`numeroBeneficiario`, `idPagamento`), nunca ISN. Caso contrário, o modelo moderno herda o acoplamento físico.
8. **Anti-padrões `MU`/`PE` precisam de ADR de modelagem antes de qualquer EARS de dados.** Decidir como normalizar (`pagamento_parcela`, `beneficiario_endereco_historico`) é pré-requisito. EARS escritas antes dessa decisão vão precisar ser reescritas.
9. **Perfis ADM/OPR/CON/AUD/SUP sem matriz formal.** Toda EARS de autorização ("o sistema DEVE permitir...") precisa do perfil habilitado. Sem matriz **perfil × operação**, as specs ficam genéricas e abrem brecha de segurança. Construir a matriz é pré-requisito.
10. **Integrações externas (SIAFI, CNAB 240/BB) mencionadas em §2.2 mas não exploradas no código.** Não existem programas `.NSN` de interface visíveis. Antes de spec de integração, mapear: arquivo, layout, frequência, contrato. Risco de descobrir tarde uma dependência crítica de cutover.
11. **Mistérios do `mysteries-found.md` são tentação para "migrar e ver depois".** Política: nenhum mistério vira EARS sem explicação validada. Migrar mistério = perpetuar dívida. Catalogar como item de backlog do Estágio 4 (Evolução), não do Estágio 2.
12. **Limites hardcoded no código (qtd dependentes, valor máx. benefício)** não estão em `REGRAS-NEGOCIO-2012`. Antes de escrever EARS com números, validar com PO se o limite ainda é vigente em 2026 — alguns podem ter sido alterados por portaria sem atualizar código.
13. **Trilha de auditoria (AUDITORIA, FNR 153) é incompleta para LGPD.** Adicionada em 2005 sem retrofit. Specs de auditoria moderna não devem só replicar o legado — precisam atender LGPD (consent log, direito ao esquecimento, hash de integridade). Abrir ADR de auditoria antes das EARS.
14. **Cadeia batch sem orquestrador formal (JCL/JES2).** O Estágio 2 precisa decidir orquestração moderna (Spring Batch + cron K8s? Azure Container Apps Jobs? Argo Workflows?) ANTES de escrever EARS de pipeline batch. ADR obrigatório.
15. **Glossário de domínio recém-criado (Estágio 1) é a fonte da verdade.** Toda EARS DEVE usar termos do [`glossary.md`](glossary.md). Divergência de vocabulário entre Requirements Engineer e Software Architect quebra rastreabilidade. Não inventar termo novo — abrir PR no glossary primeiro.

---

## 5. Recomendações

### 5.1 O que migrar primeiro

> Com base na priorização do Par 1 (Product Owner), quais funcionalidades devem ser migradas primeiro?

| Prioridade | Funcionalidade | Justificativa |
| ---------- | -------------- | ------------- |
| 1          | Consulta de beneficiário (CONSBENF) | Read-only de altíssimo volume; libera operadores das telas 3270 sem risco transacional e gera evidência de equivalência logo no início. |
| 2          | Cadastro de beneficiários + dependentes (CADBENEF, CADDEPEND) | Fonte da verdade do domínio. Migrar o dono do dado de `BENEFICIARIO` primeiro destrava o recorte correto dos bounded contexts modernos. |
| 3          | Trilha de auditoria moderna (RELAUDIT + DDM AUDITORIA) | LGPD-by-design exige audit log estruturado ANTES de qualquer escrita produtiva moderna; bloqueia avanço se deixado pra depois. |
| 4          | Motor único de elegibilidade e validação (VALELEG + VALBENEF + VALDOCS) | Resolve a duplicação detectada (BR-027/BR-031) na origem; pré-requisito para o cálculo moderno não herdar a dívida. |
| 5          | Cadastro de programas sociais como dado (CADPROG) | Tira o hardcode IF/ELSE, dá autonomia ao negócio (SENARC/CGPB) para criar/alterar programas sem deploy — valor político alto. |
| 6          | Cálculo de benefício + desconto (CALCBENF, CALCDSCT) | Coração financeiro da política pública; migra com parâmetros externalizados e validação bit-a-bit contra o mainframe em paralelo. |
| 7          | Geração do ciclo mensal de pagamento (BATCHPGT) | Job mais crítico do sistema; só vai para produção após 3 ciclos consecutivos idênticos ao legado, com dry-run obrigatório (Spring Batch). |
| 8          | Dashboards gerenciais (substituindo RELPGT spool) | Quando o pagamento moderno é fonte da verdade, relatórios viram dashboards interativos — entrega valor visível sem risco operacional. |

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
| Regras de negócio encontradas | 59       |
| Regras escondidas encontradas | 15 / 10  |
| Easter eggs encontrados       | 2 / 3   |
| Termos no glossário           | 70       |
| Mistérios catalogados         | 24       |
| Tempo total gasto             | ~8 horas |

---

## 7. Notas para o Próximo Estágio

> Deixe aqui mensagens para o time no Estágio 2 (Especificação Moderna):

boa sorte pra quem fica
---

## Definição de Pronto deste relatório

- [x] Todas as seções acima preenchidas (sem placeholders).
- [x] Pelo menos 5 regras críticas listadas em §3.1, cada uma referenciando uma `BR-XXX` do catálogo.
- [x] Decisões de migrar/descartar/evoluir em §5 cobrem as 8+ funcionalidades principais.
- [x] Métricas de §6 conferem com os outros artefatos (glossary.md, business-rules-catalog.md, mysteries-found.md).

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

