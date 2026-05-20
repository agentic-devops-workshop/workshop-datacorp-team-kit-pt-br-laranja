<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# ADR-0001: Adotar Modular Monolith em vez de Microsserviços

| Campo     | Valor                                                       |
| --------- | ----------------------------------------------------------- |
| Status    | **aceito**                                                  |
| Data      | 2026-05-20                                                  |
| Autores   | Pedro (Enterprise + Software Architect), Os Laranjinhas     |
| Substitui | N/A                                                         |

## Contexto

Estamos modernizando o SIFAP (29 anos, Natural/Adabas em mainframe) para uma stack Java 21 + Spring Boot + PostgreSQL + Next.js. O Estágio 1 (arqueologia) identificou:

- **15 programas Natural, zero `CALLNAT` entre eles** — todo acoplamento é por DDM compartilhado (ver [`01-arqueologia/dependency-map.md`](../../01-arqueologia/dependency-map.md)). 4 DDMs (BENEFICIARIO 4.2M, PROGRAMA-SOCIAL, PAGAMENTO 180M, AUDITORIA) são hubs de acoplamento por dado.
- **5 bounded contexts identificados** (ver [`02-spec-moderna/bounded-contexts.md`](../../02-spec-moderna/bounded-contexts.md)): `beneficiary`, `program-catalog`, `payment`, `reconciliation`, `audit`.
- **Time pequeno (5 pessoas)** sem experiência operacional com service mesh, contratos versionados ou observabilidade distribuída em produção.
- **Cutover estilo Strangler Fig** sobre legado vivo — exige que os contextos possam ser extraídos um a um sem big-bang.
- **Invariantes financeiros (BR-017, BR-012)** exigem transações ACID locais; transações distribuídas (saga) introduzem complexidade que não se paga em ciclo mensal de baixa concorrência.

## Decisão

**Adotamos Modular Monolith** em Java 21 + Spring Boot 3.3, com Maven multi-module:

```
sifap-modern/
├── bootstrap/              # Spring Boot main + autowiring; único deployable
├── shared-kernel/          # Cpf, Money, Competencia, enums; sem Spring
├── beneficiary/
│   ├── domain/             # puro (sem Spring, sem JPA)
│   ├── application/        # services, ports
│   └── infrastructure/     # controllers, repositories, adapters
├── program-catalog/        # mesma estrutura interna
├── payment/                # mesma estrutura interna
├── reconciliation/         # mesma estrutura interna
└── audit/                  # mesma estrutura interna
```

**Regras de fronteira:**

1. Nenhum módulo importa classes de `infrastructure` de outro módulo. Validado por **ArchUnit** no CI.
2. Comunicação inter-módulo ocorre apenas via interfaces declaradas em `application/` (sync) ou eventos de domínio Spring (`ApplicationEventPublisher`, async via `@TransactionalEventListener`).
3. Banco único PostgreSQL 16, **schema-per-context** (`beneficiary.*`, `payment.*`, …). JOIN só dentro do próprio schema; cross-schema é proibido — replica-se via projeção ou lê-se via API do outro módulo.
4. Migração futura para microsserviços, se necessária, é **módulo por módulo** (extração de `audit` primeiro, depois `reconciliation`).

## Alternativas consideradas

| Alternativa                                          | Por que foi rejeitada                                                                                                                                                                                                          |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **A. Microsserviços desde o dia 1 (5 serviços)**     | Não cabe no orçamento de skill nem temporal do time. Service mesh, contratos versionados, observabilidade distribuída e CI/CD por serviço consomem semanas. Saga para transação `payment.generate + audit.record` é overkill. |
| **B. Monolito tradicional (pacote por camada)**      | Repete o anti-padrão do legado — `controllers/`, `services/`, `repositories/` misturam os 5 contextos. Bloqueia extração futura. `dependency-map.md` denuncia exatamente este acoplamento por dado.                            |
| **C. Modular Monolith com BD único sem schemas**     | Atalho perigoso — sem schemas, JOINs cross-context viram inevitáveis. Ao extrair `audit` no futuro, refatoração de SQL é massiva. Custo marginal de schemas no Postgres é zero.                                                |
| **D. Hexagonal por contexto, sem Spring Modulith**   | Considerado; rejeitado porque Spring Modulith dá verificação automática de fronteiras (`ApplicationModules.of(...).verify()`) com 1 linha — equivalente ao ArchUnit custom mas mantido pela Pivotal.                            |

## Consequências

- **Mais fácil:**
  - 1 pipeline CI/CD, 1 deployable, 1 conjunto de credenciais Azure.
  - Refactor cross-context com IDE em vez de PRs em múltiplos repos.
  - Transação ACID local para invariantes financeiros (BR-012 idempotência, BR-017 fórmula-mãe).
- **Mais difícil:**
  - Escalar contexto único (ex.: `payment` no fim do mês) — todo o monolito escala junto. **Mitigação:** Spring Batch isolado em `payment-batch-runner` (mesma codebase, deploy separado opcional).
  - Disciplina de fronteiras — fácil "burlar" e fazer import direto. **Mitigação:** ArchUnit + Spring Modulith + revisão de PR obrigatória cruzando módulos.
- **Riscos:**
  - **R-MM1:** time relaxa as regras de fronteira sob pressão de prazo → degrada para "big ball of mud". **Mitigação:** `ArchitectureTest` falha o build; PR template tem checklist de fronteira.
  - **R-MM2:** schema-per-context em Postgres único pode virar gargalo de conexões. **Mitigação:** HikariCP pool dimensionado; monitorar com Application Insights.
- **Critérios de envelhecimento (quando revisitar):**
  - Latência p99 do endpoint `/payments` > 500ms por 7 dias seguidos.
  - Tráfego de `payment` exigir scaling 5x superior aos demais contextos.
  - Time > 15 pessoas com squads dedicadas por contexto.

## Relacionado

- REQ-IDs: todos (estrutural)
- ADRs: ADR-0002 (política financeira), ADR-0003 (motor único de descontos)
- Arquivos-fonte: [`02-spec-moderna/bounded-contexts.md`](../../02-spec-moderna/bounded-contexts.md), [`.github/instructions/modular-monolith.instructions.md`](../../.github/instructions/modular-monolith.instructions.md)
- Exemplo de referência: [`08-exemplos/ADR-001-monolito-modular-exemplo.md`](../../08-exemplos/ADR-001-monolito-modular-exemplo.md)

## Referências

- Spring Modulith — <https://spring.io/projects/spring-modulith>
- ArchUnit — <https://www.archunit.org/>
- Sam Newman, *Monolith to Microservices* (cap. 3 — Strangler Fig + módulo-a-módulo).
