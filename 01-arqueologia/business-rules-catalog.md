<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Catálogo de Regras de Negócio — SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **business-rules-catalog**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Registre aqui todas as regras de negócio extraídas do código Natural/Adabas.
> Cada regra precisa ter rastreabilidade até o código-fonte.
>
> **REGRA DURA:** linhas com `Programa Fonte` vazio são **inválidas** e não contam para o gate do Estágio 2. Use o formato `01-arqueologia/legado-sifap/natural-programs/ARQUIVO.NSN#L<inicio>-L<fim>` sempre que possível. Mínimo aceito: nome do arquivo .NSN.

## Como pensar em "regra de negócio"

O que conta:

- Um `IF` que decide algo no domínio (ex.: _"se a UF é do Nordeste e o programa é Seca, valor base × 1.2"_)
- Uma constante numérica sem explicação (ex.: `0.075` num cálculo de imposto)
- Uma transição de status com regra (ex.: _"só de A para S, nunca de I para A"_)
- Um tratamento especial para um caso (ex.: _"se o CPF começa com 999, é teste"_)

O que NÃO conta: paginação de relatório, formatação de saída, manipulação de cursor Adabas, abertura de arquivo. Ignore esses detalhes de implementação.

## Níveis de Risco

| Nível       | Descrição                                                     |
| ----------- | ------------------------------------------------------------- |
| **CRÍTICO** | Regra financeira ou de segurança — erro causa prejuízo direto |
| **ALTO**    | Regra de negócio central — afeta fluxo principal              |
| **MÉDIO**   | Regra de validação ou formatação — afeta qualidade dos dados  |
| **BAIXO**   | Regra de apresentação ou conveniência — impacto limitado      |

## Regras Encontradas

| ID     | Regra de Negócio | Programa Fonte | Campos DDM | Nível de Risco | Notas |
| ------ | ---------------- | -------------- | ---------- | -------------- | ----- |
| BR-001 | Tipos de desconto: Judicial, Pensão, Imposto, Sindical, Administrativo — cada um com fórmula e teto próprios | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L80-L120` | `PAGAMENTO.TIPO-DSCT`, `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT` | ALTO | Judicial não tem teto; Sindical sempre 1% |
| BR-002 | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L60-L65` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO | Regra financeira. Tipo 'J' = exceção legal |
| BR-003 | Desconto só é aplicado se estiver vigente (data início/fim) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L73-L80` | `BENEFICIARIO.DT-INICIO-DSCT`, `BENEFICIARIO.DT-FIM-DSCT` | MÉDIO | Controle de vigência de descontos |
| BR-004 | Contribuição social obrigatória por faixa de renda: 3%, 5%, 7%, 9% | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L45-L52` | `PAGAMENTO.VLR-BRUTO` | ALTO | Tabela de alíquotas, alterada em 2015 |
| BR-005 | Não corrigir pagamento já marcado como corrigido | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L96-L99` | `PAGAMENTO.IND-CORRIGIDO` | MÉDIO | Garante idempotência |
| BR-006 | Período de correção deve ser válido (competência inicial ≤ final) | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L85-L89` | `PAGAMENTO.COMPETENCIA` | BAIXO | Validação de entrada |
| BR-007 | Correção retroativa: índice acumulado IPCA por competência | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L102-L110` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-CORRECAO` | ALTO | Fórmula financeira, depende de tabela IPCA |
| BR-008 | Só aplicar correção se diferença for positiva | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L114-L125` | `PAGAMENTO.VLR-CORRECAO`, `PAGAMENTO.VLR-BRUTO` | MÉDIO | Evita gravação de correção nula |
| BR-009 | Tabelas IPCA mensais (2010-2012) usadas para cálculo de correção | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L52-L108` | — | MÉDIO | Base histórica, manutenção periódica |
| BR-010 | Só calcula benefício para beneficiário com status 'A' (ativo) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L270-L273` | `BENEFICIARIO.STATUS` | ALTO | Pré-condição obrigatória |
| BR-011 | Competência deve ser válida (mês 1-12) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L253-L258` | `PAGAMENTO.COMPETENCIA` | BAIXO | Validação de entrada |
| BR-012 | Fator regional aplicado conforme UF/região (27 regiões) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L120-L146` | `BENEFICIARIO.COD-REGIAO`, `PAGAMENTO.VLR-BRUTO` | ALTO | Diferença de valores por localização |
| BR-013 | Fator familiar progressivo conforme número de dependentes | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L307-L321` | `BENEFICIARIO.NUM-DEPENDENTES` | ALTO | Bônus por dependentes |
| BR-014 | Fator renda: 5 faixas progressivas (quanto maior a renda, menor o fator) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L151-L160` | `BENEFICIARIO.RENDA-FAMILIAR` | ALTO | Focalização social |
| BR-015 | Fator idade: bônus para idosos (≥60) e menores (<18) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L323-L337` | `BENEFICIARIO.DT-NASCIMENTO` | MÉDIO | Proteção a grupos vulneráveis |
| BR-016 | Fórmula principal do benefício: BASE × FAT_REG × FAT_FAM × FAT_RENDA × FAT_IDADE × (1+REAJUSTE) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L340-L347` | `PAGAMENTO.VLR-BRUTO`, `PROGRAMA.VLR-BASE`, `BENEFICIARIO.*` | CRÍTICO | Coração do cálculo do sistema |
| BR-017 | 13º salário só em dezembro, fórmula diferenciada | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L349-L373` | `PAGAMENTO.TIPO-PGTO`, `PAGAMENTO.VLR-13` | ALTO | Não aplica fator família/renda |
| BR-018 | Abono natalino: 15% extra só para programas tipo 'A' em dezembro | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L356-L365` | `PROGRAMA.TIPO`, `PAGAMENTO.VLR-ABONO` | MÉDIO | Regra específica para assistenciais |
| BR-019 | Truncamento para 2 casas decimais (não arredonda) | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L344-L347` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-LIQUIDO` | MÉDIO | Compatibilidade mainframe |
| BR-020 | Valor líquido nunca pode ser negativo | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L378-L383` | `PAGAMENTO.VLR-LIQUIDO` | BAIXO | Proteção contra erro de cálculo |

> Adicione mais linhas conforme necessário. Lembre-se: existem **10 regras escondidas** no código!

## Exemplo de linha bem preenchida

| ID     | Regra de Negócio                                                                        | Programa Fonte                                   | Campos DDM                                                               | Nível de Risco | Notas                                      |
| ------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ | -------------- | ------------------------------------------ |
| BR-013 | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L142-L148` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO        | Regra financeira. Tipo 'J' = exceção legal |

## Regras por Categoria

### Cálculos Financeiros

<!-- Liste aqui as regras relacionadas a cálculos de valores, benefícios, etc. -->

### Validações de Status

<!-- Liste aqui as regras de transição de status (A, S, C, I, D) -->

### Regras de Autorização

<!-- Liste aqui as regras de quem pode fazer o quê -->

### Regras de Negócio Temporais

<!-- Liste aqui regras com prazos, datas-limite, períodos -->

## Resumo Estatístico

- Total de regras encontradas: \_\_\_
- Regras críticas: \_\_\_
- Regras com duplicação: \_\_\_
- Regras sem documentação (escondidas): \_\_\_

---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="GUIDE.md"><strong>GUIDE do Estágio 1</strong></a><br/>
<sub>Passo a passo do estágio.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="dependency-map.md"><strong>dependency-map.md</strong></a><br/>
<sub>Mapa de quem chama quem.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

