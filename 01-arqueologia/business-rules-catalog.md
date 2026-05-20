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
| BR-021 | Se o tipo de busca vier em branco, o sistema assume busca por CPF (padrão = C). | 01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L80-L81 | BENEFICIARIO.CPF, BENEFICIARIO.NIS | MÉDIO | Define comportamento default da consulta online. |
| BR-022 | Só são aceitos tipos de busca C (CPF) e N (NIS); qualquer outro valor é inválido e encerra a rotina. | 01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L85-L97 | BENEFICIARIO.CPF, BENEFICIARIO.NIS | ALTO | Regra de integridade de entrada da consulta. |
| BR-023 | Se o beneficiário não for encontrado, a consulta é interrompida sem exibir dados. | 01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L100-L102 | BENEFICIARIO.CPF, BENEFICIARIO.NIS | ALTO | Evita exibição de dados inconsistentes. |
| BR-024 | O status do beneficiário é traduzido para descrição de negócio: A=ATIVO, S=SUSPENSO, C=CANCELADO, I=INATIVO, D=DESLIGADO; demais valores viram DESCONHECIDO. | 01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L110-L124 | BENEFICIARIO.STATUS | MÉDIO | Mapeamento usado na tela de consulta. |
| BR-025 | O histórico exibido na consulta é limitado aos últimos 12 pagamentos. | 01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L156-L158 | PAGAMENTO.CPF-BENEF, PAGAMENTO.COMPETENCIA, PAGAMENTO.VLR-BRUTO, PAGAMENTO.VLR-LIQUIDO | BAIXO | Regra funcional de apresentação com limite fixo. |
| BR-026 | O CPF deve ser mascarado na saída da consulta; há regra específica para CPF com menos de 11 dígitos. | 01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L177-L188 | BENEFICIARIO.CPF | ALTO | Regra de proteção de dado sensível; comentário aponta inconsistência conhecida da máscara. |
| BR-027 | No relatório de pagamentos, o processamento para quando a competência lida ultrapassa a competência final informada. | 01-arqueologia/legado-sifap/natural-programs/RELPGT.NSN#L83-L85 | PAGAMENTO.COMPETENCIA | ALTO | Regra temporal de corte do período. |
| BR-028 | Filtro por programa: quando COD-PROG-FILTRO = 0, considera todos; quando diferente de 0, mantém apenas pagamentos do programa informado. | 01-arqueologia/legado-sifap/natural-programs/RELPGT.NSN#L79-L89 | PAGAMENTO.COD-PROGRAMA | MÉDIO | Regra explícita de escopo do relatório. |
| BR-029 | Subtotal deve ser fechado toda vez que houver mudança de COD-PROGRAMA durante a leitura. | 01-arqueologia/legado-sifap/natural-programs/RELPGT.NSN#L93-L98 | PAGAMENTO.COD-PROGRAMA, PAGAMENTO.VLR-BRUTO, PAGAMENTO.VLR-LIQUIDO | MÉDIO | Regra de agregação por programa. |
| BR-030 | O tipo de pagamento é categorizado como NORMAL (N), DECIMO (D), TERCEIRO (T), e OUTRO para demais códigos. | 01-arqueologia/legado-sifap/natural-programs/RELPGT.NSN#L116-L126 | PAGAMENTO.TIPO-PGTO | MÉDIO | Classificação usada na saída analítica. |
| BR-031 | O status do pagamento é categorizado como GERADO (G), PAGO (P), CANCELAD (C), DEVOLVID (D), ESTORNAD (E), e OUTRO para demais valores. | 01-arqueologia/legado-sifap/natural-programs/RELPGT.NSN#L128-L142 | PAGAMENTO.STATUS-PGTO | ALTO | Mapeamento operacional central para leitura da situação do pagamento. |
| BR-032 | No relatório de auditoria, se o tipo de saída estiver em branco, o padrão é T (tela). | 01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L80-L81 | AUDITORIA | BAIXO | Regra de default de interface. |
| BR-033 | Se as datas de filtro não forem informadas, usa DT-INI = 19970101 e DT-FIM = data atual. | 01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L84-L88 | AUDITORIA.DT-EVENTO | MÉDIO | Regra temporal padrão para abrangência histórica. |
| BR-034 | Registros de auditoria com ação EX são sempre ocultados do relatório, independentemente dos demais filtros. | 01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L105-L107 | AUDITORIA.ACAO | CRÍTICO | Regra com impacto de compliance e rastreabilidade de exclusões. |
| BR-035 | Filtros de ação, usuário e tabela só são aplicados quando informados; quando em branco, não restringem os resultados. | 01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L111-L129 | AUDITORIA.ACAO, AUDITORIA.USUARIO, AUDITORIA.TABELA-REF | ALTO | Implementa filtragem condicional por parâmetros de entrada. |

> Adicione mais linhas conforme necessário. Lembre-se: existem **10 regras escondidas** no código!

## Exemplo de linha bem preenchida

| ID     | Regra de Negócio                                                                        | Programa Fonte                                   | Campos DDM                                                               | Nível de Risco | Notas                                      |
| ------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ | -------------- | ------------------------------------------ |
| BR-013 | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L142-L148` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO        | Regra financeira. Tipo 'J' = exceção legal |

## Regras por Categoria

### Cálculos Financeiros

- Não foram encontradas regras de cálculo financeiro nestes três programas; o foco deles é consulta e relatório.

### Validações de Status

- BR-024: Mapeamento de status de beneficiário.
- BR-031: Mapeamento de status de pagamento.

### Regras de Autorização

- BR-034: Supressão obrigatória de ações EX no relatório.
- BR-035: Restrição condicional por ação, usuário e tabela.

### Regras de Negócio Temporais

- BR-027: Corte do período por competência final.
- BR-033: Datas padrão quando filtros não são informados.

## Resumo Estatístico

- Total de regras encontradas: 15
- Regras críticas: 1
- Regras com duplicação: 2
- Regras sem documentação (escondidas): 2

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

