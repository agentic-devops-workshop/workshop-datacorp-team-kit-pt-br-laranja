# Catálogo de Campos — Adabas DDMs do SIFAP

Data: 20/05/2026  
Escopo: `01-arqueologia/legado-sifap/adabas-ddms/`  
Objetivo: Mapeamento estrutural de campos para modernização

---

## 1. AUDITORIA.ddm

**Arquivo:** 153 | **Registros:** ~25M | **Tamanho médio:** ~1.2 KB

Log de auditoria — trilha de alterações (registro imutável).

| Level | Nome do Campo | Tipo | Tamanho | OCC | Marcador | Descrição |
|-------|---------------|------|---------|-----|----------|-----------|
| 1 | AA - NUM-AUDITORIA | N | 15 | — | **DE** | Sequencial único |
| 1 | AB - DT-EVENTO | N | 8 | — | **DE** | Data AAAAMMDD |
| 1 | AC - HR-EVENTO | N | 6 | — | — | Hora HHMMSS |
| 1 | AD - TS-EVENTO | N | 14 | — | — | Timestamp AAAAMMDDHHMMSS |
| 1 | BA - COD-ACAO | A | 2 | — | **DE** | Tipo ação (IN/AL/EX/CO/LG/LO/BT/ER/AU/RE) |
| 1 | BB - COD-MODULO | A | 8 | — | — | Nome programa Natural |
| 1 | BC - DES-ACAO | A | 80 | — | — | Descrição livre |
| 1 | CA - TIPO-ENTIDADE | A | 4 | — | — | BENF/PGTO/PROG/ADMN/SIST |
| 1 | CB - ID-ENTIDADE | A | 15 | — | **DE** | Chave entidade |
| 1 | CC - NUM-CPF-AFETADO | A | 11 | — | **DE** | CPF (se aplicável) |
| 1 | DA - GRP-ANTES | — | — | — | **Grupo** | Estado anterior |
| 2 | DB - CAMPO-ALTERADO-ANT | A | 30 | 20 | **MU** | Nome campo (máx 20 valores) |
| 2 | DC - VALOR-ANTERIOR | A | 80 | 20 | **MU** | Valor anterior (máx 20 valores) |
| 1 | DD - GRP-DEPOIS | — | — | — | **Grupo** | Estado posterior |
| 2 | DE - CAMPO-ALTERADO-DEP | A | 30 | 20 | **MU** | Nome campo (máx 20 valores) |
| 2 | DF - VALOR-POSTERIOR | A | 80 | 20 | **MU** | Valor posterior (máx 20 valores) |
| 1 | EA - USR-EVENTO | A | 8 | — | **DE** | Login Natural |
| 1 | EB - NOME-USUARIO | A | 40 | — | — | Nome completo |
| 1 | EC - COD-PERFIL | A | 3 | — | — | ADM/OPR/CON/AUD/SUP |
| 1 | ED - COD-LOTACAO | A | 10 | — | — | Unidade organizacional |
| 1 | EE - IP-ORIGEM | A | 15 | — | — | IP terminal (2012+) |
| 1 | EF - ID-SESSAO | A | 20 | — | — | Identificador sessão |
| 1 | FA - NUM-CICLO-BATCH | N | 6 | — | — | Ciclo processamento |
| 1 | FB - NUM-SEQ-BATCH | N | 10 | — | — | Sequencial ciclo |
| 1 | FC - NOM-JOB-BATCH | A | 16 | — | — | Nome job JES2/JCL |
| 1 | FD - SIT-BATCH | A | 1 | — | — | S=Sucesso E=Erro W=Warning |
| 1 | FE - DES-ERRO-BATCH | A | 120 | — | — | Mensagem erro (se aplicável) |
| 1 | GA - ID-CORRELACAO | A | 36 | — | — | UUID operação composta |
| 1 | GB - NUM-SEQ-CORRELACAO | N | 3 | — | — | Sequencial operação |

**Super-Descritores:**
- S1 = AB(1-8) + BA(1-2) — Data + Ação
- S2 = CA(1-4) + CB(1-15) + AB(1-8) — Entidade + Data
- S3 = EA(1-8) + AB(1-8) — Usuário + Data

**Resumo:** 34 campos | 22 simples + 4 MU + 8 controle

---

## 2. BENEFICIARIO.ddm

**Arquivo:** 150 | **Registros:** ~4.2M | **Tamanho médio:** ~850 B

Cadastro de beneficiários — base principal do SIFAP.

| Level | Nome do Campo | Tipo | Tamanho | OCC | Marcador | Descrição |
|-------|---------------|------|---------|-----|----------|-----------|
| 1 | AA - NUM-INSCRICAO | N | 11 | — | — | ISN alternativo / matrícula |
| 1 | AB - NUM-CPF | A | 11 | — | **DE** | CPF sem formatação |
| 1 | AC - NOME-COMPLETO | A | 60 | — | — | Nome civil oficial |
| 1 | AD - NOME-MAE | A | 60 | — | — | Nome da mãe (obrigatório) |
| 1 | AE - NOME-PAI | A | 60 | — | — | Nome do pai (opcional) |
| 1 | AF - DT-NASCIMENTO | N | 8 | — | — | Data AAAAMMDD |
| 1 | AG - SEXO | A | 1 | — | — | M/F/I (I=Indefinido) |
| 1 | AH - EST-CIVIL | A | 1 | — | — | S/C/D/V/U |
| 1 | AI - RG-NUMERO | A | 15 | — | — | Número RG |
| 1 | AJ - RG-ORGAO | A | 10 | — | — | Órgão expedidor |
| 1 | AK - RG-UF | A | 2 | — | — | UF expedição |
| 1 | AL - RG-DT-EXPEDICAO | N | 8 | — | — | Data expedição AAAAMMDD |
| 1 | BA - GRP-ENDERECO | — | — | — | **Grupo** | Grupo de endereço |
| 2 | BB - LOGRADOURO | A | 60 | — | — | Rua/Avenida/Travessa |
| 2 | BC - NUMERO | A | 10 | — | — | Número (alfanumérico) |
| 2 | BD - COMPLEMENTO | A | 30 | — | — | Apto/Bloco/Sala |
| 2 | BE - BAIRRO | A | 40 | — | — | Bairro |
| 2 | BF - MUNICIPIO | A | 40 | — | — | Município |
| 2 | BG - UF | A | 2 | — | **DE** | Sigla UF |
| 2 | BH - CEP | N | 8 | — | — | CEP sem hífen |
| 2 | BI - COD-IBGE | N | 7 | — | — | Código município IBGE |
| 2 | BJ - COD-REGIAO | A | 2 | — | — | 01-05 ou 99 (especial) |
| 1 | CA - COD-PROGRAMA | A | 4 | — | — | Código programa social |
| 1 | CB - DT-CADASTRO | N | 8 | — | **DE** | Data AAAAMMDD |
| 1 | CC - DT-INICIO-BENEF | N | 8 | — | — | Data início benefício |
| 1 | CD - DT-FIM-BENEF | N | 8 | — | — | Data fim benefício (0=sem prazo) |
| 1 | CE - SIT-BENEFICIARIO | A | 1 | — | **DE** | A/S/C/I/D |
| 1 | CF - MOT-SITUACAO | A | 3 | — | — | Código motivo |
| 1 | CG - DT-ULT-SITUACAO | N | 8 | — | — | Data última situação |
| 1 | CH - VLR-RENDA-FAMILIAR | N | 9.2 | — | — | Renda declarada |
| 1 | CI - QTD-MEMBROS-FAMILIA | N | 2 | — | — | Membros no domicílio |
| 1 | CJ - IND-RENDA-PERCAP | N | 7.2 | — | — | Renda per capita calculada |
| 1 | DA - GRP-DEPENDENTE | — | — | **10** | **PE** | Grupo periódico (máx 10) |
| 2 | DB - CPF-DEPENDENTE | A | 11 | — | — | CPF ou 00000000000 |
| 2 | DC - NOME-DEPENDENTE | A | 60 | — | — | Nome dependente |
| 2 | DD - DT-NASC-DEPEND | N | 8 | — | — | Data nascimento AAAAMMDD |
| 2 | DE - PARENTESCO | A | 2 | — | — | FI/CJ/NT/TU |
| 2 | DF - SIT-DEPENDENTE | A | 1 | — | — | A/I/D |
| 2 | DG - IND-DEFICIENCIA | A | 1 | — | — | S/N |
| 1 | EA - TEL-FIXO | A | 14 | — | — | (DD) NNNN-NNNN |
| 1 | EB - TEL-CELULAR | A | 15 | — | — | (DD) NNNNN-NNNN |
| 1 | EC - EMAIL | A | 80 | — | — | Email notificação |
| 1 | FA - IND-BIOMETRIA | A | 1 | — | — | S/N/P |
| 1 | FB - DT-COLETA-BIO | N | 8 | — | — | Data coleta AAAAMMDD |
| 1 | FC - COD-POSTO-BIO | A | 6 | — | — | Código posto coleta |
| 1 | FD - HASH-DIGITAL | A | 64 | — | — | SHA-256 template (não implementado) |
| 1 | GA - DT-INCLUSAO | N | 8 | — | **DE** | Data AAAAMMDD |
| 1 | GB - HR-INCLUSAO | N | 6 | — | — | Hora HHMMSS |
| 1 | GC - USR-INCLUSAO | A | 8 | — | — | Login Natural |
| 1 | GD - DT-ULT-ALTERACAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | GE - HR-ULT-ALTERACAO | N | 6 | — | — | Hora HHMMSS |
| 1 | GF - USR-ULT-ALTERACAO | A | 8 | — | — | Login Natural |
| 1 | GG - NUM-VERSAO | N | 5 | — | — | Controle concorrência |

**Super-Descritores:**
- S1 = AB(1-11) — CPF completo
- S2 = BG(1-2) + CE(1-1) — UF + Situação
- S3 = CA(1-4) + CE(1-1) — Programa + Situação

**Resumo:** 52 campos | 37 simples + 1 PE (6 campos × 10 ocorrências) + 14 controle/auditoria

---

## 3. PAGAMENTO.ddm

**Arquivo:** 152 | **Registros:** ~180M | **Tamanho médio:** ~720 B

Histórico de pagamentos — tabela transacional crítica.

| Level | Nome do Campo | Tipo | Tamanho | OCC | Marcador | Descrição |
|-------|---------------|------|---------|-----|----------|-----------|
| 1 | AA - NUM-PAGAMENTO | N | 15 | — | **DE** | Sequencial único |
| 1 | AB - NUM-CPF | A | 11 | — | **DE** | CPF beneficiário |
| 1 | AC - NUM-INSCRICAO | N | 11 | — | — | Matrícula beneficiário |
| 1 | AD - COD-PROGRAMA | A | 4 | — | **DE** | Programa social |
| 1 | AE - ANO-MES-REF | N | 6 | — | **DE** | Competência AAAAMM |
| 1 | AF - NUM-CICLO | N | 6 | — | — | Ciclo processamento |
| 1 | BA - VLR-BRUTO | N | 9.2 | — | — | Valor bruto calculado |
| 1 | BB - VLR-LIQUIDO | N | 9.2 | — | — | Valor líquido (bruto - descontos) |
| 1 | BC - VLR-DESCONTO-TOTAL | N | 7.2 | — | — | Soma descontos |
| 1 | CA - GRP-DESCONTO | — | — | **8** | **PE** | Descontos aplicados (máx 8) |
| 2 | CB - TIPO-DESCONTO | A | 3 | — | — | IR/JD/CS/PA/EM/TX/OU/EX |
| 2 | CC - VLR-DESCONTO | N | 7.2 | — | — | Valor desconto |
| 2 | CD - PCT-DESCONTO | N | 3.2 | — | — | Percentual aplicado |
| 2 | CE - NUM-PROCESSO | A | 20 | — | — | Número processo judicial (se JD) |
| 2 | CF - DT-INICIO-DSCT | N | 8 | — | — | Data início AAAAMMDD |
| 2 | CG - DT-FIM-DSCT | N | 8 | — | — | Data fim AAAAMMDD (0=indefinido) |
| 1 | DA - SIT-PAGAMENTO | A | 1 | — | — | P/G/E/C/D/X/R |
| 1 | DB - DT-GERACAO | N | 8 | — | **DE** | Data geração AAAAMMDD |
| 1 | DC - HR-GERACAO | N | 6 | — | — | Hora HHMMSS |
| 1 | DD - DT-EMISSAO | N | 8 | — | — | Data envio banco AAAAMMDD |
| 1 | DE - DT-CONFIRMACAO | N | 8 | — | — | Data retorno banco AAAAMMDD |
| 1 | DF - DT-CANCELAMENTO | N | 8 | — | — | Data cancelamento AAAAMMDD (se aplicável) |
| 1 | DG - MOT-CANCELAMENTO | A | 3 | — | — | Código motivo |
| 1 | EA - COD-BANCO | A | 3 | — | — | Código FEBRABAN |
| 1 | EB - COD-AGENCIA | A | 6 | — | — | Número agência |
| 1 | EC - NUM-CONTA | A | 13 | — | — | Número conta |
| 1 | ED - TIPO-CONTA | A | 1 | — | — | C/P |
| 1 | EE - COD-OPERACAO | A | 3 | — | — | Operação caixa (se aplicável) |
| 1 | FA - NUM-OB-SIAFI | A | 12 | — | — | Ordem bancária SIAFI |
| 1 | FB - NUM-NE-SIAFI | A | 12 | — | — | Nota empenho SIAFI |
| 1 | FC - COD-UG-EMITENTE | A | 6 | — | — | Unidade gestora |
| 1 | FD - COD-GESTAO | A | 5 | — | — | Código gestão SIAFI |
| 1 | FE - SIT-INTEG-SIAFI | A | 1 | — | — | I/P/E |
| 1 | GA - DT-CONCILIACAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | GB - SIT-CONCILIACAO | A | 1 | — | — | C/D/P/N |
| 1 | GC - VLR-CONCILIADO | N | 9.2 | — | — | Valor confirmado banco |
| 1 | GD - COD-RETORNO-BANCO | A | 2 | — | — | Código retorno CNAB 240 |
| 1 | GE - DES-RETORNO-BANCO | A | 40 | — | — | Descrição retorno |
| 1 | HA - HASH-ARQ-REMESSA | A | 64 | — | — | SHA-256 arquivo remessa |
| 1 | HB - HASH-ARQ-RETORNO | A | 64 | — | — | SHA-256 arquivo retorno |
| 1 | IA - DT-INCLUSAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | IB - HR-INCLUSAO | N | 6 | — | — | Hora HHMMSS |
| 1 | IC - USR-INCLUSAO | A | 8 | — | — | Login (geralmente 'BATCH') |
| 1 | ID - DT-ULT-ALTERACAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | IE - HR-ULT-ALTERACAO | N | 6 | — | — | Hora HHMMSS |
| 1 | IF - USR-ULT-ALTERACAO | A | 8 | — | — | Login usuário |

**Super-Descritores:**
- S1 = AB(1-11) + AE(1-6) — CPF + Competência
- S2 = AD(1-4) + AE(1-6) + DA(1-1) — Programa + Competência + Situação
- S3 = AF(1-6) + DA(1-1) — Ciclo + Situação

**Resumo:** 50 campos | 31 simples + 1 PE (7 campos × 8 ocorrências) + 18 controle/auditoria

---

## 4. PROGRAMA-SOCIAL.ddm

**Arquivo:** 151 | **Registros:** ~45 | **Tamanho médio:** ~420 B

Cadastro de programas sociais — tabela paramétrica.

| Level | Nome do Campo | Tipo | Tamanho | OCC | Marcador | Descrição |
|-------|---------------|------|---------|-----|----------|-----------|
| 1 | AA - COD-PROGRAMA | A | 4 | — | **DE** | Chave primária |
| 1 | AB - NOME-PROGRAMA | A | 60 | — | — | Nome oficial |
| 1 | AC - SIGLA-PROGRAMA | A | 10 | — | — | Sigla (PBF, BPC, PETI) |
| 1 | AD - TIPO-PROGRAMA | A | 1 | — | — | A/T/P |
| 1 | AE - ORGAO-RESPONSAVEL | A | 10 | — | — | Código órgão MDS/MDAS |
| 1 | AF - LEI-CRIACAO | A | 20 | — | — | Número lei ou decreto |
| 1 | AG - DT-CRIACAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | AH - DT-ENCERRAMENTO | N | 8 | — | — | Data AAAAMMDD (0=vigente) |
| 1 | AI - SIT-PROGRAMA | A | 1 | — | — | A/I/E |
| 1 | BA - VLR-BASE-INDIVIDUAL | N | 7.2 | — | — | Valor mensal base/pessoa |
| 1 | BB - VLR-BASE-FAMILIAR | N | 7.2 | — | — | Valor mensal base/família |
| 1 | BC - VLR-TETO-BENEF | N | 9.2 | — | — | Valor máximo benefício |
| 1 | BD - VLR-PISO-BENEF | N | 7.2 | — | — | Valor mínimo benefício |
| 1 | BE - PCT-REAJUSTE-ANUAL | N | 3.2 | — | — | Percentual reajuste anual |
| 1 | BF - DT-ULT-REAJUSTE | N | 8 | — | — | Data último reajuste AAAAMMDD |
| 1 | BG - FATOR-K | N | 5.4 | — | — | Fator correção especial (não documentado) |
| 1 | CA - RENDA-MAX-PERCAP | N | 7.2 | — | — | Renda per capita máxima |
| 1 | CB - IDADE-MIN | N | 3 | — | — | Idade mínima beneficiário (0=sem) |
| 1 | CC - IDADE-MAX | N | 3 | — | — | Idade máxima beneficiário (0=sem) |
| 1 | CD - IND-EXIGE-FILHOS | A | 1 | — | — | S/N |
| 1 | CE - QTD-MIN-FILHOS | N | 2 | — | — | Mínimo filhos (se CD='S') |
| 1 | CF - IND-EXIGE-ESCOLA | A | 1 | — | — | S/N (frequência escolar) |
| 1 | CG - IND-EXIGE-VACINA | A | 1 | — | — | S/N (carteira vacina) |
| 1 | CH - IND-EXIGE-PRENATAL | A | 1 | — | — | S/N (prenatal) |
| 1 | CI - IND-EXIGE-BIOMETRIA | A | 1 | — | — | S/N (obrigatório a partir 2005) |
| 1 | DA - GRP-FAIXA-CALCULO | — | — | **5** | **PE** | Faixas cálculo (máx 5) |
| 2 | DB - RENDA-INICIO | N | 7.2 | — | — | Início faixa renda |
| 2 | DC - RENDA-FIM | N | 7.2 | — | — | Fim faixa renda |
| 2 | DD - FATOR-MULTIPLICADOR | N | 3.4 | — | — | Fator sobre valor base |
| 2 | DE - VLR-ADICIONAL | N | 7.2 | — | — | Valor fixo adicional |
| 2 | DF - IND-ACUMULATIVO | A | 1 | — | — | S (acumula com faixa anterior) |
| 1 | EA - TIPO-DSCT-APLIC | A | 3 | — | **MU** | Tipos desconto válidos (máx 8) |
| 1 | FA - GRP-PARAM-REGIONAL | — | — | **6** | **PE** | Parâmetros regionais (máx 6: 5 regiões + 1 especial) |
| 2 | FB - COD-REGIAO | A | 2 | — | — | 01-05 ou 99 |
| 2 | FC - FATOR-REGIONAL | N | 3.4 | — | — | Multiplicador regional |
| 2 | FD - VLR-COMPLEMENTO-REG | N | 7.2 | — | — | Complemento fixo regional |
| 2 | FE - IND-ATIVO-REGIAO | A | 1 | — | — | S/N |
| 1 | GA - DT-INCLUSAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | GB - USR-INCLUSAO | A | 8 | — | — | Login usuário |
| 1 | GC - DT-ULT-ALTERACAO | N | 8 | — | — | Data AAAAMMDD |
| 1 | GD - USR-ULT-ALTERACAO | A | 8 | — | — | Login usuário |

**Super-Descritores:**
- S1 = AA(1-4) — Código programa
- S2 = AD(1-1) + AI(1-1) — Tipo + Situação

**Resumo:** 42 campos | 19 simples + 1 MU (máx 8) + 2 PE (5 campos × 5 + 4 campos × 6) + 4 controle

---

## Resumo Consolidado

| DDM | Registros | Campos | Simples | MU | PE | Desc. | Obs |
|-----|-----------|--------|---------|----|----|-------|-----|
| **AUDITORIA** | 25M | 34 | 22 | 4 | — | 3 | Log imutável, retenção 10 anos |
| **BENEFICIARIO** | 4.2M | 52 | 37 | — | 1 | 3 | Base principal, PE=dependentes (máx 10) |
| **PAGAMENTO** | 180M | 50 | 31 | — | 1 | 3 | Crítico, PE=descontos (máx 8) |
| **PROGRAMA-SOCIAL** | 45 | 42 | 19 | 1 | 2 | 2 | Paramétrica, PE=cálculo + regional |
| **TOTAL** | 209M+ | 178 | 109 | 5 | 4 | — | — |

---

## Mapeamento PostgreSQL — DDM por DDM

### AUDITORIA.ddm → Tabela `audit_event`

**Campos principais:**
- `num_auditoria` BIGINT PRIMARY KEY
- `dt_evento` DATE NOT NULL
- `hr_evento` TIME NOT NULL
- `ts_evento` TIMESTAMP(0) NOT NULL
- `cod_acao` CHAR(2) NOT NULL
- `cod_modulo` VARCHAR(8)
- `des_acao` VARCHAR(80)
- `tipo_entidade` VARCHAR(4)
- `id_entidade` VARCHAR(15)
- `num_cpf_afetado` CHAR(11)
- ...

**Campos MU:**
- `campo_alterado_ant` TEXT[] (máx 20) — tabela auxiliar recomendada para normalizar
- `valor_anterior` TEXT[] (máx 20)
- `campo_alterado_dep` TEXT[] (máx 20)
- `valor_posterior` TEXT[] (máx 20)

**Nulabilidade:**
- NOT NULL para chaves e datas principais; demais campos podem ser NULL

**Anti-padrão:**
- Uso de arrays (MU) para campos alterados/valores — dificulta queries SQL; ideal seria tabela `audit_event_change` (1:N)

---

### BENEFICIARIO.ddm → Tabela `beneficiary`

**Campos principais:**
- `num_inscricao` BIGINT PRIMARY KEY
- `num_cpf` CHAR(11) UNIQUE NOT NULL
- `nome_completo` VARCHAR(60) NOT NULL
- `dt_nascimento` DATE NOT NULL
- `status` CHAR(1) NOT NULL
- ...

**Grupo PE:**
- `dependentes` — tabela `beneficiary_dependent` (1:N, máx 10)
    - `cpf_dependente` CHAR(11)
    - `nome_dependente` VARCHAR(60)
    - `dt_nasc_depend` DATE
    - `parentesco` CHAR(2)
    - `sit_dependente` CHAR(1)
    - `ind_deficiencia` CHAR(1)

**Nulabilidade:**
- NOT NULL para chaves, nomes, datas essenciais; campos opcionais podem ser NULL

**Anti-padrão:**
- Grupo PE fixo (máx 10) — pode limitar famílias maiores; ideal seria sem limite

---

### PAGAMENTO.ddm → Tabela `payment`

**Campos principais:**
- `num_pagamento` BIGINT PRIMARY KEY
- `num_cpf` CHAR(11) NOT NULL
- `cod_programa` CHAR(4) NOT NULL
- `ano_mes_ref` CHAR(6) NOT NULL
- `vlr_bruto` NUMERIC(11,2) NOT NULL
- `vlr_liquido` NUMERIC(11,2)
- ...

**Grupo PE:**
- `descontos` — tabela `payment_discount` (1:N, máx 8)
    - `tipo_desconto` CHAR(3)
    - `vlr_desconto` NUMERIC(9,2)
    - `pct_desconto` NUMERIC(5,2)
    - `num_processo` VARCHAR(20)
    - `dt_inicio_dsct` DATE
    - `dt_fim_dsct` DATE

**Nulabilidade:**
- NOT NULL para chaves, datas, valores principais; descontos podem ser NULL

**Anti-padrão:**
- PE de descontos limitado a 8 — pode não cobrir casos futuros; recomendável remover limite

---

### PROGRAMA-SOCIAL.ddm → Tabela `social_program`

**Campos principais:**
- `cod_programa` CHAR(4) PRIMARY KEY
- `nome_programa` VARCHAR(60) NOT NULL
- `sigla_programa` VARCHAR(10)
- `tipo_programa` CHAR(1) NOT NULL
- ...

**Campos MU:**
- `tipo_dsct_aplic` TEXT[] (máx 8) — recomendável tabela auxiliar `social_program_discount_type`

**Grupos PE:**
- `faixa_calculo` — tabela `social_program_calc_band` (1:N, máx 5)
    - `renda_inicio` NUMERIC(9,2)
    - `renda_fim` NUMERIC(9,2)
    - `fator_multiplicador` NUMERIC(7,4)
    - ...
- `param_regional` — tabela `social_program_region_param` (1:N, máx 6)
    - `cod_regiao` CHAR(2)
    - `fator_regional` NUMERIC(7,4)
    - ...

**Nulabilidade:**
- NOT NULL para chaves, nomes, tipos; demais campos podem ser NULL

**Anti-padrão:**
- MU e PE com limite fixo — dificulta expansão futura; ideal seria tabelas auxiliares sem limite

---

## Anti-padrões identificados

- Arrays (MU) e grupos periódicos (PE) com tamanho fixo dificultam normalização e expansão futura
- Campos de controle e auditoria misturados com dados de negócio
- Uso de tipos genéricos (A/N) sem validação de domínio
- Falta de datas/hora com precisão (ex: só AAAAMMDD)

---

## Sugestão de revisão

- Para cada campo MU ou PE, criar tabela auxiliar 1:N
- Usar tipos PostgreSQL adequados: `NUMERIC`, `VARCHAR`, `DATE`, `TIME`, `BOOLEAN` (para flags)
- Definir NOT NULL apenas onde obrigatório por negócio
- Adicionar constraints e índices conforme uso real

---

## Legendas

- **Type:** A=Alpha | N=Numeric | P=Packed Decimal
- **DE:** Descriptor (campo indexado para busca rápida)
- **MU:** Multiple-Value (array, máximo de ocorrências listado)
- **PE:** Periodic Group (grupo repetitivo, máximo de ocorrências listado)
- **OCC:** Ocorrências (— = uma única, número = múltiplas)
- **Desc.:** Número de super-descritores definidos

---

## Mapeamento para Modernização

### Correspondências Adabas → Moderno

| Conceito Adabas | Mapeamento Moderno |
|-----------------|-------------------|
| **Descriptor (DE)** | `@Column` + `@Index` em JPA |
| **Super-Descriptor** | `@Index(columnList = "col1, col2")` |
| **MU (Multiple-Value)** | `@ElementCollection` com `@CollectionTable` |
| **PE (Periodic Group)** | `@OneToMany` com entidade embedded ou `@ElementCollection` |
| **Packed Decimal** | `@Column(precision=N, scale=2)` + `BigDecimal` em Java |

### Dependências Entre DDMs

```
PROGRAMA-SOCIAL (base: 45)
    ↓ referenced by
BENEFICIARIO (CA → AA) — beneficiários de programas
    ↓ referenced by
PAGAMENTO (AD → CA, AB → AB) — pagamentos de beneficiários
    ↑
    ↓ audited by
AUDITORIA (CA, CB, CC)
```

---

**Documento gerado:** 20/05/2026  
**Escopo:** Arqueologia SIFAP · Estágio 1  
**Próximo passo:** Mapear programas Natural que leem/escrevem estes DDMs