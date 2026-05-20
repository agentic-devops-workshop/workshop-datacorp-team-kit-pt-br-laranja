<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# C4 Nível 1 — System Context — SIFAP 2.0

![ESTÁGIO 02 Spec](https://img.shields.io/badge/ESTÁGIO-02%20Spec-00A4EF?style=for-the-badge) ![C4 L1 System Context](https://img.shields.io/badge/C4-L1%20System%20Context-1A1A1A?style=for-the-badge)

> **Período retratado:** janela de Strangler Fig — SIFAP 2.0 em operação paralela ao SIFAP legado.

## Diagrama

```mermaid
C4Context
    title SIFAP 2.0 — System Context (Strangler Fig window)

    Person(operador, "Operador SIFAP", "OPR — cadastro e consulta diários, opera via web")
    Person(supervisor, "Supervisor", "SUP — autoriza operações sensíveis (alteração CPF, cancelamento)")
    Person(auditor, "Auditor", "AUD — consulta trilha, fiscaliza pagamentos (CGU/TCU)")
    Person(adm, "Administrador", "ADM — gestão de programas sociais, perfis, parâmetros")
    Person(beneficiario, "Beneficiário", "Consulta seu próprio extrato (autoatendimento)")

    System_Boundary(sifap2, "SIFAP 2.0") {
        System(sifap, "SIFAP 2.0", "Java 21 + Spring Boot + Next.js sobre Azure. Modular Monolith com 5 contextos.")
    }

    System_Ext(legacy3270, "SIFAP Legado (3270)", "Natural/Adabas no mainframe. Operação reduzida durante Strangler; aposentadoria gradual.")
    System_Ext(bb, "Banco do Brasil + multi-banco", "Recebe remessa de pagamento, devolve retorno CNAB 240")
    System_Ext(siafi, "SIAFI", "Empenho e execução orçamentária federal (integração planejada — Estágio 4)")
    System_Ext(aad, "Azure AD / Entra ID", "Autenticação federada SSO; perfis ADM/OPR/CON/AUD/SUP via scope claim")
    System_Ext(receita, "Receita Federal", "Validação eventual de CPF/RG (consulta opcional)")

    Rel(operador, sifap, "Cadastra/consulta beneficiários, lança descontos", "HTTPS")
    Rel(supervisor, sifap, "Autoriza, cancela", "HTTPS")
    Rel(auditor, sifap, "Consulta trilha, gera relatórios CGU", "HTTPS")
    Rel(adm, sifap, "Configura programas, perfis, parâmetros", "HTTPS")
    Rel(beneficiario, sifap, "Consulta extrato pessoal", "HTTPS pública")

    Rel(sifap, aad, "Valida token JWT", "OIDC")
    Rel(sifap, bb, "Envia remessa, importa retorno", "SFTP + CNAB 240")
    Rel(sifap, receita, "Consulta CPF (opcional)", "HTTPS / API")
    Rel_Back(sifap, legacy3270, "Lê dados não-migrados via gateway", "Adaptador read-only")
    Rel(sifap, siafi, "Empenho (futuro)", "SOAP/REST")
```

## Atores

| Ator           | Perfil legado | Capacidades principais                                                          |
| -------------- | ------------- | ------------------------------------------------------------------------------- |
| Operador       | OPR           | Cadastro/atualização de beneficiário, consulta, lançamento de descontos          |
| Supervisor     | SUP           | Aprovação de operações sensíveis (alteração CPF, cancelamento de pagamento)     |
| Auditor        | AUD           | Consulta trilha de auditoria com mascaramento controlado, relatórios CGU/TCU    |
| Administrador  | ADM           | Cadastro/manutenção de programas sociais, parâmetros do sistema                 |
| Beneficiário   | (novo — N2)   | Autoatendimento: consulta extrato pessoal (greenfield, não existe no legado)    |

## Sistemas externos

| Sistema           | Direção  | Protocolo            | Notas                                                                                       |
| ----------------- | -------- | -------------------- | ------------------------------------------------------------------------------------------- |
| Azure AD / Entra  | in       | OIDC / OAuth2        | Substitui autorização interna do Com-plete; perfis no scope claim                           |
| Banco BB + outros | bidir    | SFTP + CNAB 240      | Saída: remessa; entrada: retorno. Multi-banco habilitado (fim do `COD-BANCO=1` MYS-005)     |
| Receita Federal   | out      | HTTPS                | Consulta opcional de CPF — não bloqueia cadastro                                            |
| SIAFI             | out      | SOAP/REST            | Empenho e execução orçamentária — entra no Estágio 4 (escopo de evolução)                   |
| SIFAP Legado      | bidir RO | Adapter via gateway  | Leitura de dados não migrados durante Strangler; nenhuma escrita                            |

## Estratégia Strangler Fig

Durante o cutover, `SIFAP 2.0` coexiste com legado:

1. **Fase 1 — read-through:** todo POST/PUT vai para SIFAP 2.0; GET fallback para legado se contexto ainda não migrado.
2. **Fase 2 — write-through dual:** escritas em `beneficiary` vão para os dois sistemas (legado em modo somente-leitura para leitura cruzada).
3. **Fase 3 — cutover por contexto:** `audit` primeiro (baixo risco), depois `beneficiary`, `program-catalog`, `reconciliation` e finalmente `payment` (mais arriscado por causa de BR-017).
4. **Fase 4 — aposentadoria:** legado em modo emergência por 1 ciclo mensal completo; depois desligado.

Métrica de bloqueio (NFR-004): `payment.cycle.diff_vs_legacy_centavos` precisa ficar em zero por 3 ciclos antes do cutover de `payment`.
