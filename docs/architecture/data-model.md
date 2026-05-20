<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Modelo de Dados — SIFAP 2.0

![ESTÁGIO 02 Spec](https://img.shields.io/badge/ESTÁGIO-02%20Spec-00A4EF?style=for-the-badge) ![DDL PostgreSQL 16](https://img.shields.io/badge/DDL-PostgreSQL%2016-336791?style=for-the-badge)

> **Pré-requisitos:** [`bounded-contexts.md`](../../02-spec-moderna/bounded-contexts.md), ADR-0001 (schema-per-context), ADR-0002 (Money), ADR-0003 (motor de descontos).

## Princípios

1. **Schema-per-context.** Cada bounded context tem seu próprio schema PostgreSQL. JOIN cross-schema é proibido por código (ArchUnit) e por convenção; comunicação é via API do outro módulo ou via evento de domínio.
2. **Sem FK física cross-schema.** Chaves cruzam por value object (`Cpf`) ou por id natural — nunca por FK física. Isso permite extração futura de contexto sem refactor de schema.
3. **`Money` como NUMERIC(15,2).** Converter JPA mapeia para `BigDecimal` escala 2 (ADR-0002).
4. **`Competencia` como INTEGER no formato AAAAMM.** Mesmo formato do legado (BR-024) para facilitar migração.
5. **`Cpf` como CHAR(11) sem máscara.** Validação módulo-11 no domínio (REQ-BEN-001); display sempre via máscara LGPD (REQ-BEN-005).
6. **PE (periodic group) do Adabas → tabela filha owned via `@OneToMany`.** Não usar arrays Postgres — quebra portabilidade de queries.
7. **MU (multi-value) do Adabas → `jsonb`** quando ordem não importa e leitura é sempre integral; tabela filha quando há queries por elemento.

## Mapeamento Adabas → PostgreSQL

| DDM legado (FNR)       | Contexto         | Schema PostgreSQL   | Tabela principal                      | Estratégia para MU/PE                                                  |
| ---------------------- | ---------------- | ------------------- | ------------------------------------- | ---------------------------------------------------------------------- |
| `BENEFICIARIO` (150)   | beneficiary      | `beneficiary`       | `beneficiary.beneficiary`             | PE de dependentes → `beneficiary.dependent` (FK); MU endereços → `jsonb` |
| `PROGRAMA-SOCIAL` (151)| program-catalog  | `program_catalog`   | `program_catalog.program`             | Versionamento via `program_base_value_history`                         |
| `PAGAMENTO` (152)      | payment          | `payment`           | `payment.payment` (partitioned)       | MU de parcelas → `payment.payment_installment`                         |
| `AUDITORIA` (153)      | audit            | `audit`             | `audit.event` (append-only)           | Conteúdo do evento → `jsonb`; cadeia via `prev_hash`/`row_hash`        |

## DDL (esqueleto)

### Schema `shared` (somente tipos e funções comuns)

```sql
CREATE SCHEMA IF NOT EXISTS shared;

-- Função de mascaramento (REQ-BEN-005)
CREATE OR REPLACE FUNCTION shared.mask_cpf(cpf CHAR(11))
RETURNS VARCHAR(14) LANGUAGE sql IMMUTABLE AS $$
    SELECT '***.' || SUBSTRING(cpf FROM 4 FOR 3) || '.' || SUBSTRING(cpf FROM 7 FOR 3) || '-**';
$$;
```

### Schema `beneficiary`

```sql
CREATE SCHEMA IF NOT EXISTS beneficiary;

CREATE TABLE beneficiary.beneficiary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cpf             CHAR(11)    NOT NULL UNIQUE,   -- REQ-BEN-001 valida; REQ-BEN-003 unicidade
    nis             CHAR(11),
    nome_completo   VARCHAR(120) NOT NULL,
    dt_nascimento   DATE         NOT NULL,
    cod_regiao      SMALLINT     NOT NULL,
    renda_familiar  NUMERIC(15,2) NOT NULL DEFAULT 0,  -- Money (ADR-0002)
    status          CHAR(1)      NOT NULL CHECK (status IN ('A','S','C','D')),  -- BR-037
    enderecos       JSONB,       -- MU endereços do Adabas
    versao          BIGINT       NOT NULL DEFAULT 0,   -- @Version JPA
    criado_em       TIMESTAMPTZ  NOT NULL DEFAULT now(),
    atualizado_em   TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX idx_beneficiary_status ON beneficiary.beneficiary(status) WHERE status = 'A';
CREATE INDEX idx_beneficiary_nis ON beneficiary.beneficiary(nis) WHERE nis IS NOT NULL;

CREATE TABLE beneficiary.dependent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    beneficiary_id  UUID NOT NULL REFERENCES beneficiary.beneficiary(id) ON DELETE CASCADE,
    cpf_dep         CHAR(11),
    nome_dep        VARCHAR(120) NOT NULL,
    dt_nascimento   DATE NOT NULL,
    parentesco      CHAR(2) NOT NULL CHECK (parentesco IN ('FI','CO','IR','OU')),  -- BR-040
    criado_em       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (beneficiary_id, cpf_dep)  -- BR-041: anti-fraude
);

CREATE TABLE beneficiary.system_parameter (
    key VARCHAR(60) PRIMARY KEY,
    value TEXT NOT NULL,
    description TEXT,
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Parâmetros default
INSERT INTO beneficiary.system_parameter(key, value, description) VALUES
    ('max_dependents', '5', 'REQ-BEN-006 / BR-039 — default legado (D5 diferida)'),
    ('demographic_review_age', '75', 'REQ-BEN-004 / BR-037 / MYS-014 — idade que suspende automaticamente');
```

### Schema `program_catalog`

```sql
CREATE SCHEMA IF NOT EXISTS program_catalog;

CREATE TABLE program_catalog.program (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo          VARCHAR(20) NOT NULL UNIQUE,
    nome            VARCHAR(120) NOT NULL,
    tipo            CHAR(1) NOT NULL CHECK (tipo IN ('A','B','C','D')),  -- BR-020
    status          CHAR(1) NOT NULL CHECK (status IN ('A','I')),         -- BR-011
    vlr_base        NUMERIC(15,2) NOT NULL,   -- Money
    fator_reajuste  NUMERIC(7,6)  NOT NULL DEFAULT 0,
    criado_em       TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE program_catalog.program_base_value_history (
    id              BIGSERIAL PRIMARY KEY,
    program_id      UUID NOT NULL REFERENCES program_catalog.program(id),
    vlr_base        NUMERIC(15,2) NOT NULL,
    fator_reajuste  NUMERIC(7,6) NOT NULL,
    vigencia_inicio DATE NOT NULL,
    vigencia_fim    DATE,
    motivo          TEXT NOT NULL
);

CREATE TABLE program_catalog.system_parameter (
    key VARCHAR(60) PRIMARY KEY,
    value TEXT NOT NULL
);
INSERT INTO program_catalog.system_parameter(key, value) VALUES
    ('actuarial_factor', '0.347215');  -- REQ-PRG-001 / BR-042 / MYS-016 (D7 diferida)
```

### Schema `payment` (particionado)

```sql
CREATE SCHEMA IF NOT EXISTS payment;

-- Tabela principal particionada por competência (BR-024)
CREATE TABLE payment.payment (
    id               UUID NOT NULL DEFAULT gen_random_uuid(),
    competencia      INTEGER NOT NULL,           -- AAAAMM (BR-024)
    cpf_benef        CHAR(11) NOT NULL,          -- chave de domínio (não FK cross-schema)
    program_id       UUID NOT NULL,              -- id do programa (não FK cross-schema)
    tipo_pgto        CHAR(1) NOT NULL CHECK (tipo_pgto IN ('M','D','R')), -- M=mensal, D=décimo, R=retroativo
    vlr_bruto        NUMERIC(15,2) NOT NULL,     -- Money — calculado por BR-017 (REQ-PAY-002)
    vlr_abono        NUMERIC(15,2) NOT NULL DEFAULT 0, -- REQ-PAY-004
    vlr_desconto     NUMERIC(15,2) NOT NULL DEFAULT 0, -- soma das deductions (ADR-0003)
    vlr_correcao     NUMERIC(15,2) NOT NULL DEFAULT 0, -- BR-031
    vlr_liquido      NUMERIC(15,2) NOT NULL,     -- max(0, bruto + abono - desconto + correcao) (BR-022)
    cod_banco        SMALLINT,                   -- preservado do CNAB (resolve MYS-005)
    num_pgto         VARCHAR(20),
    status           CHAR(1) NOT NULL CHECK (status IN ('G','P','D','E','C')),  -- BR-010
    requires_manual_review BOOLEAN NOT NULL DEFAULT FALSE,  -- REQ-REC-002
    ind_corrigido    CHAR(1) NOT NULL DEFAULT 'N', -- BR-029
    criado_em        TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em    TIMESTAMPTZ NOT NULL DEFAULT now(),
    versao           BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (id, competencia),
    UNIQUE (cpf_benef, competencia, program_id, tipo_pgto)  -- BR-012 idempotência
) PARTITION BY RANGE (competencia);

-- Exemplo de partição (CI/CD cria por mês via job)
CREATE TABLE payment.payment_202605 PARTITION OF payment.payment
    FOR VALUES FROM (202605) TO (202606);

CREATE INDEX idx_payment_status ON payment.payment(status);
CREATE INDEX idx_payment_cpf ON payment.payment(cpf_benef);

-- Tabela de regras de desconto (ADR-0003, parametrização BR-026/BR-028)
CREATE TABLE payment.deduction_rule (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tipo_desconto    VARCHAR(20) NOT NULL CHECK (tipo_desconto IN ('SOCIAL','UNION','JUDICIAL','OTHER')),
    faixa_min        NUMERIC(15,2),         -- nullable para regras sem faixa
    faixa_max        NUMERIC(15,2),
    percentual       NUMERIC(7,6),          -- ex.: 0.030000 = 3%
    vlr_fixo         NUMERIC(15,2),         -- alternativa a percentual
    vigencia_inicio  DATE NOT NULL,
    vigencia_fim     DATE,
    motivo           TEXT NOT NULL
);
-- Seed com regras legadas
INSERT INTO payment.deduction_rule (tipo_desconto, faixa_min, faixa_max, percentual, vigencia_inicio, motivo) VALUES
    ('SOCIAL', 0.00,   500.00, 0.030000, DATE '1996-01-01', 'BR-028 faixa 1'),
    ('SOCIAL', 500.01, 1000.00, 0.050000, DATE '1996-01-01', 'BR-028 faixa 2'),
    ('SOCIAL', 1000.01, 2000.00, 0.070000, DATE '1996-01-01', 'BR-028 faixa 3'),
    ('SOCIAL', 2000.01, NULL, 0.090000, DATE '1996-01-01', 'BR-028 faixa 4'),
    ('UNION', NULL, NULL, 0.010000, DATE '1996-01-01', 'BR-026 sindical 1% (substitui hardcode)');

-- Aplicações de desconto por pagamento (eventos)
CREATE TABLE payment.deduction_applied (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL,
    competencia     INTEGER NOT NULL,
    rule_id         UUID REFERENCES payment.deduction_rule(id),
    tipo_desconto   VARCHAR(20) NOT NULL,
    vlr_aplicado    NUMERIC(15,2) NOT NULL,
    ajustado_por_teto BOOLEAN NOT NULL DEFAULT FALSE,
    criado_em       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tabela externa de índices de correção (N10, resolve MYS-023)
CREATE TABLE payment.correction_index (
    id              BIGSERIAL PRIMARY KEY,
    indice          VARCHAR(10) NOT NULL CHECK (indice IN ('IPCA','IGP-M','TR')),
    competencia     INTEGER NOT NULL,
    fator           NUMERIC(10,8) NOT NULL,
    UNIQUE (indice, competencia)
);

CREATE TABLE payment.system_parameter (
    key VARCHAR(60) PRIMARY KEY,
    value TEXT NOT NULL
);
INSERT INTO payment.system_parameter(key, value) VALUES
    ('december_bonus_pct', '0.15'),                 -- REQ-PAY-004 / BR-020 / MYS-009
    ('non_judicial_deduction_cap_pct', '0.30');     -- REQ-PAY-005 / BR-025
```

### Schema `reconciliation`

```sql
CREATE SCHEMA IF NOT EXISTS reconciliation;

CREATE TABLE reconciliation.cnab_file (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome_arquivo    VARCHAR(200) NOT NULL,
    cod_banco       SMALLINT NOT NULL,
    dt_recebimento  TIMESTAMPTZ NOT NULL DEFAULT now(),
    qtd_registros   INTEGER NOT NULL,
    status          CHAR(1) NOT NULL CHECK (status IN ('R','P','F'))  -- Recebido, Processado, Falha
);

CREATE TABLE reconciliation.cnab_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cnab_file_id    UUID NOT NULL REFERENCES reconciliation.cnab_file(id),
    cpf_benef       CHAR(11) NOT NULL,
    num_pgto        VARCHAR(20) NOT NULL,
    competencia     INTEGER NOT NULL,
    vlr_centavos    BIGINT NOT NULL,        -- BR-002 — preserva original
    cod_retorno     CHAR(2) NOT NULL,        -- BR-005
    status_match    VARCHAR(20) NOT NULL,    -- MATCHED, DIVERGED, UNMATCHED, UNKNOWN_CODE
    diff_centavos   BIGINT
);
```

### Schema `audit` (append-only)

```sql
CREATE SCHEMA IF NOT EXISTS audit;

CREATE TABLE audit.event (
    id              BIGSERIAL PRIMARY KEY,
    seq             BIGINT NOT NULL UNIQUE,    -- garantir ordem total para hash-chain
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    context         VARCHAR(30) NOT NULL,       -- beneficiary | payment | ...
    event_type      VARCHAR(60) NOT NULL,
    actor_id        VARCHAR(60) NOT NULL,
    actor_profile   CHAR(3) NOT NULL CHECK (actor_profile IN ('ADM','OPR','CON','AUD','SUP','SYS')),
    target_kind     VARCHAR(30) NOT NULL,
    target_id       VARCHAR(60) NOT NULL,
    payload         JSONB NOT NULL,
    prev_hash       CHAR(64) NOT NULL,        -- SHA-256 hex
    row_hash        CHAR(64) NOT NULL,        -- SHA-256 hex
    CONSTRAINT chk_genesis CHECK (seq > 0)
);
CREATE INDEX idx_audit_target ON audit.event(target_kind, target_id);
CREATE INDEX idx_audit_actor ON audit.event(actor_id);

-- Append-only: UPDATE e DELETE proibidos
CREATE OR REPLACE FUNCTION audit.deny_modification() RETURNS trigger AS $$
BEGIN
    RAISE EXCEPTION 'audit.event is append-only (REQ-AUD-001)';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_no_update BEFORE UPDATE ON audit.event
    FOR EACH ROW EXECUTE FUNCTION audit.deny_modification();
CREATE TRIGGER trg_audit_no_delete BEFORE DELETE ON audit.event
    FOR EACH ROW EXECUTE FUNCTION audit.deny_modification();

-- Outbox (publicação assíncrona com exactly-once)
CREATE TABLE audit.outbox (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_id    VARCHAR(60) NOT NULL,
    event_type      VARCHAR(60) NOT NULL,
    payload         JSONB NOT NULL,
    criado_em       TIMESTAMPTZ NOT NULL DEFAULT now(),
    processado_em   TIMESTAMPTZ
);
CREATE INDEX idx_outbox_pending ON audit.outbox(criado_em) WHERE processado_em IS NULL;
```

## Mapeamento JPA (esqueleto, sem implementar)

| Conceito Adabas      | Anotação JPA               | Nota                                           |
| -------------------- | -------------------------- | ---------------------------------------------- |
| Campo simples        | `@Column`                  | direto                                         |
| MU (multi-value)     | `@Column @Convert` → `jsonb` | usar `JsonBinaryType` (Hypersistence) ou JSON nativo do Hibernate 6 |
| PE (periodic group)  | `@OneToMany` + `cascade=ALL`+`orphanRemoval=true` | tabela filha; agregado dono escreve  |
| DDM Descriptor       | `@Column(unique=true)` + índice | mapeado para coluna indexada              |
| ISN (id físico)      | descartado — usar UUID gerado pela app | rastreabilidade só no audit              |
| Sequência Adabas     | `@GeneratedValue(strategy=AUTO)` ou `BIGSERIAL` | para audit.seq usar SEQUENCE         |
| `Money` (NUMERIC)    | `@Convert(converter=MoneyConverter.class)` | ADR-0002                              |
| `Competencia` INT    | `@Convert(converter=CompetenciaConverter.class)` | value object no domínio               |

## Migração de dados — princípios

1. **Histórico é sagrado.** Pagamentos já gravados no Adabas vêm como estão (truncate legado) — nunca recalcular.
2. **Backdoors de CPF (D1) não migram.** Beneficiários com CPF inválido por REQ-BEN-001 entram em quarentena `beneficiary.quarantine` para revisão manual.
3. **Hash-chain do audit começa do zero no SIFAP 2.0.** Genesis hash documentado em ADR futuro. Audit legado preservado em archive somente-leitura.
4. **Particionamento de `payment.payment`** preenchido sob demanda — partições por competência criadas via job mensal antes do batch.
