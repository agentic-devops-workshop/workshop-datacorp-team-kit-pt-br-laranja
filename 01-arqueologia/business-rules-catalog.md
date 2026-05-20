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

> Faixas de linha são aproximadas (contagem a partir do início do arquivo, incluindo o cabeçalho de comentários). Par 2 deve validar via `grep -n` antes da Passagem H1.

| ID     | Regra de Negócio                                                                                                                       | Programa Fonte                                                                | Campos DDM                                                              | Nível de Risco | Notas                                                                                       |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------- |
| BR-036 | Apenas registros CNAB tipo `'3'` (detalhe) são conciliados; cabeçalhos/trailers são descartados                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L110-L113`         | —                                                                       | MÉDIO          | Layout CNAB 240 BB                                                                          |
| BR-037 | Valores no arquivo CNAB chegam em **centavos** e devem ser divididos por 100 para virarem reais                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L124-L126`         | `PAGAMENTO.VLR-LIQUIDO`                                                 | CRÍTICO        | Erro de unidade = pagamento 100× errado                                                     |
| BR-038 | Conciliação bancária só ocorre se casarem **3 chaves**: `NUM-PAGTO` + `CPF-BENEF` + `COMPETENCIA`                                      | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L131-L137`         | `PAGAMENTO.NUM-PAGTO`, `CPF-BENEF`, `COMPETENCIA`                       | ALTO           | Match parcial é tratado como "não encontrado"                                                |
| BR-039 | Tolerância de divergência de valor na conciliação é R$ 0,01 (diferença ≤ 1 centavo é conciliada)                                       | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L147-L152`         | `PAGAMENTO.VLR-LIQUIDO`                                                 | CRÍTICO        | Magic number sem documentação                                                                |
| BR-040 | Código de retorno bancário → status do pagamento: `'00'`→`'P'` (Pago), `'01'`→`'D'` (Devolvido), `'02'`→`'E'` (Estornado)              | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L160-L182`         | `PAGAMENTO.STATUS-PGTO`, `COD-RETORNO`                                  | CRÍTICO        | Outros códigos viram apenas WRITE em log — pagamento fica em status anterior (silencioso)   |
| BR-041 | Pagamento conciliado tem `COD-BANCO` fixado em `1` (hardcoded Banco do Brasil)                                                         | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L165`              | `PAGAMENTO.COD-BANCO`                                                   | ALTO           | Impede multi-banco apesar do histórico de "INC BANCO REAL"                                  |
| BR-042 | Toda conciliação (sucesso ou divergência) gera registro em `AUDITORIA` com ação `'CO'` ou `'DV'`                                       | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L200-L235`         | `AUDITORIA.ACAO`, `TABELA-REF`, `CHAVE-REF`                             | ALTO           | LGPD/compliance — preservar na modernização                                                  |
| BR-043 | Mapeamento `COD-REGIAO` → macro-região por faixa: 1-5=Norte, 6-10=Nordeste, 11-15=Sudeste, 16-20=Sul, 21+=Centro-Oeste                 | `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L98-L114`          | `BENEFICIARIO.COD-REGIAO`                                               | ALTO           | Beneficiário sem região (cod=0) cai em Centro-Oeste por fallback                            |
| BR-044 | Relatório consolidado **arredonda** valor bruto (`+ 0.005`) enquanto pagamento real é **truncado** → totais não batem                  | `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L118-L121`         | `PAGAMENTO.VLR-BRUTO`                                                   | CRÍTICO        | Inconsistência financeira plantada — ver MYS-001                                            |
| BR-045 | Domínio fechado de status do pagamento: `G`=Gerado, `P`=Pago, `C`=Cancelado, `D`=Devolvido, `E`=Estornado                              | `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L132-L145`         | `PAGAMENTO.STATUS-PGTO`                                                 | ALTO           | Status desconhecido vira "Gerado" silenciosamente (fallback `NONE`)                          |
| BR-046 | Geração de pagamento só ocorre para beneficiário com `STATUS = 'A'` (Ativo) e programa com `STATUS-PROG = 'A'`                         | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L150-L192`         | `BENEFICIARIO.STATUS`, `PROGRAMA-SOCIAL.STATUS-PROG`                    | CRÍTICO        | Validação dupla — beneficiário e programa                                                    |
| BR-047 | Idempotência mensal: não gera novo pagamento se já existe um para o mesmo CPF na mesma competência                                     | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L155-L165`         | `PAGAMENTO.CPF-BENEF`, `COMPETENCIA`                                    | CRÍTICO        | Garantia anti-duplicidade do batch                                                           |
| BR-048 | Fator regional aplicado ao benefício é indexado em tabela hardcoded de 27 posições (valores 1.00–1.40)                                 | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L116-L143`         | `BENEFICIARIO.COD-REGIAO`                                               | CRÍTICO        | Posições 26 e 27 inicializadas mas inalcançáveis (range checado é 1-25) — ver MYS-002       |
| BR-049 | Fator familiar escalonado: 0 dep=1.0; 1-2 dep=1.0+(n×0.05); 3-4 dep=1.10+((n-2)×0.03); ≥5 dep=1.16+((n-4)×0.02)                        | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L210-L222`         | `BENEFICIARIO.NUM-DEPENDENTES`                                          | CRÍTICO        | Regra escalonada — testar com 0, 2, 4, 5, 10 dependentes                                    |
| BR-050 | Faixas de renda familiar (5 faixas, fator decrescente): ≤300→1.00; ≤600→0.85; ≤1000→0.70; ≤1500→0.55; >1500→0.40                       | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L155-L164`         | `BENEFICIARIO.RENDA-FAMILIAR`                                           | CRÍTICO        | Valores políticos — confirmar com PO antes de mudar                                          |
| BR-051 | Fator idade: ≥65 anos=1.15 (idoso); ≥60=1.10; <18=1.05 (menor); demais=1.00                                                            | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L230-L240`         | `BENEFICIARIO.DT-NASCIMENTO`                                            | ALTO           | Idade calculada só por ano (ignora mês/dia) — ver MYS-003                                   |
| BR-052 | Fórmula do benefício bruto = `VLR-BASE × fator_reg × fator_fam × fator_renda × fator_idade × (1 + fator_reajuste)`                     | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L245-L250`         | `PROGRAMA-SOCIAL.VLR-BASE`, `FATOR-REAJUSTE`                            | CRÍTICO        | Regra-mãe do sistema — replicar bit-a-bit                                                    |
| BR-053 | Valores monetários são **truncados** (não arredondados) para 2 casas decimais via `INT(x×100)/100`                                     | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L251-L253`         | `PAGAMENTO.VLR-BRUTO`, `VLR-LIQUIDO`                                    | CRÍTICO        | Política financeira oficial — diverge do BATCHREL (ver BR-044)                              |
| BR-054 | 13º salário pago **apenas em dezembro** (mês=12), fórmula = `VLR-BASE × fator_reg × fator_idade` (sem fam, renda ou reajuste)          | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L260-L266`         | `PAGAMENTO.TIPO-PGTO`, `VLR-BRUTO`                                      | CRÍTICO        | Tipo de pagamento marcado `'D'` em dezembro                                                  |
| BR-055 | Abono de 15% sobre benefício mensal pago em dezembro **apenas para programas com `TIPO = 'A'`**                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L267-L271`         | `PROGRAMA-SOCIAL.TIPO`, `PAGAMENTO.VLR-ABONO`                           | CRÍTICO        | Magic number 0.15 sem comentário                                                             |
| BR-056 | Desconto único de 3% aplicado somente quando bruto > R$ 500,00 (sem faixas progressivas)                                               | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L282-L286`         | `PAGAMENTO.VLR-DESCONTO`                                                | CRÍTICO        | Magic numbers 500.00 e 0.03 — confirmar com PO                                              |
| BR-057 | Valor líquido nunca pode ser negativo — piso em zero                                                                                   | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L290-L292`         | `PAGAMENTO.VLR-LIQUIDO`                                                 | ALTO           | Proteção contra desconto > bruto                                                             |
| BR-058 | Pagamento criado tem status inicial `'G'` (Gerado), aguardando processamento bancário downstream                                       | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L305`              | `PAGAMENTO.STATUS-PGTO`                                                 | ALTO           | Máquina de estados: G → P/D/E (via BATCHCON)                                                |
| BR-059 | Competência derivada da data de execução: `(ano × 100) + mês` no formato AAAAMM                                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L85-L87`           | `PAGAMENTO.COMPETENCIA`                                                 | MÉDIO          | Implica execução **no mês de competência** — rodar em janeiro gera comp do mês anterior?    |

> Adicione mais linhas conforme necessário. Lembre-se: existem **10 regras escondidas** no código!

## Exemplo de linha bem preenchida

| ID     | Regra de Negócio                                                                        | Programa Fonte                                   | Campos DDM                                                               | Nível de Risco | Notas                                      |
| ------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ | -------------- | ------------------------------------------ |
| BR-013 | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L142-L148` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO        | Regra financeira. Tipo 'J' = exceção legal |

## Regras por Categoria

### Cálculos Financeiros

- BR-037 (centavos → reais), BR-039 (tolerância R$ 0,01), BR-044 (round vs truncate), BR-048 (fator regional), BR-049 (fator familiar), BR-050 (faixas de renda), BR-051 (fator idade), BR-052 (fórmula principal), BR-053 (truncamento), BR-054 (13º), BR-055 (abono 15%), BR-056 (desconto 3%), BR-057 (piso zero)

### Validações de Status

- BR-040 (cod retorno → status), BR-045 (domínio fechado de status), BR-046 (apenas ativos), BR-058 (status inicial 'G')

### Regras de Autorização

- (não identificadas neste lote de programas batch — verificar com Par 1/Par 4 nos `CADBENEF` e `VALELEG`)

### Regras de Negócio Temporais

- BR-047 (idempotência mensal), BR-054 (13º só em dezembro), BR-059 (cálculo de competência)

## Resumo Estatístico

- Total de regras encontradas: **24**
- Regras críticas: **17**
- Regras com duplicação: **1** (truncate vs round entre BATCHPGT e BATCHREL — BR-053/BR-044)
- Regras sem documentação (escondidas / magic numbers): **8** (BR-039, BR-041, BR-048 slots 26-27, BR-051 sem mês/dia, BR-055 0.15, BR-056 500/0.03, BR-059 mês corrente)

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
