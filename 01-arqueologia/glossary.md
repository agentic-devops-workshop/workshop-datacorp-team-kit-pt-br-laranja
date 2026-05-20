<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Glossário do SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **glossary**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Preencha esta tabela com todos os termos, abreviações e siglas encontrados no código Natural/Adabas.
> **Meta: no mínimo 30 termos.**

## Por que isso importa

Sistemas legados têm vocabulário próprio que ninguém documenta em lugar nenhum — só está no nome das variáveis. Se o time do Estágio 2 não souber o que `DSCT`, `BENF`, `PE` ou `CTC` significam, vai escrever uma spec sobre o que ele _acha_ que isso significa. Glossário é o que evita esse desencontro.

## Como preencher

- **Termo**: a abreviação ou sigla exatamente como aparece no código
- **Expansão**: o significado completo do termo
- **Programa**: em qual arquivo `.NSN` ou `.ddm` o termo foi encontrado
- **Contexto**: breve explicação de como/onde o termo é usado

## Dica de extração

Prompt útil no Copilot Chat (cole o conteúdo de 2–3 arquivos `.NSN` no chat antes):

> _"Liste todas as abreviações e siglas usadas neste código Natural. Para cada uma, sugira a expansão e marque com 'CONFIRMADO' ou 'HIPÓTESE'."_


## Termos encontrados

| #   | Termo | Expansão | Programa | Contexto |
| --- | ----- | -------- | -------- | -------- |
| 1   | `SIFAP` | Sistema de Fiscalização e Administração de Pagamentos | Todos os `.NSN` e `.ddm` | Nome do sistema legado — aparece no cabeçalho de todos os programas |
| 2   | `CNAB` | Código Nacional de Atividades Bancárias | `BATCHCON.NSN` | Layout de arquivo bancário de 240 bytes por registro. Usado na conciliação com o Banco do Brasil (`CNAB 240`) |
| 3   | `CPF` | Cadastro de Pessoas Físicas | Todos os `.NSN`, `BENEFICIARIO.ddm`, `PAGAMENTO.ddm` | Identificador único do beneficiário. Campo `N11`. Algoritmo de validação (dígitos verificadores) implementado em `CADBENEF.NSN`, `VALBENEF.NSN`, `VALDOCS.NSN` |
| 4   | `NIS` | Número de Identificação Social | `BATCHPGT.NSN`, `CADBENEF.NSN`, `CONSBENF.NSN`, `BENEFICIARIO.ddm` | Identificador alternativo ao CPF. Campo `N11`. A tela `CONSBENF.NSN` aceita busca por `C`=CPF ou `N`=NIS |
| 5   | `RG` | Registro Geral (identidade civil) | `CADBENEF.NSN`, `VALDOCS.NSN`, `BENEFICIARIO.ddm` | Documento de identidade. DDM tem 4 campos: `RG-NUMERO`, `RG-ORGAO`, `RG-UF`, `RG-DT-EXPEDICAO` |
| 6   | `CEP` | Código de Endereçamento Postal | `CADBENEF.NSN`, `BENEFICIARIO.ddm` | CEP do endereço. Armazenado como `N8` sem hífen |
| 7   | `UF` | Unidade Federativa (sigla do estado) | Todos os `.NSN`, `BENEFICIARIO.ddm` | Sigla do estado (`A2`). Tabela com 27 UFs válidas em `VALBENEF.NSN`. Campo indexado (DE) no DDM |
| 8   | `VLR` | Valor (monetário) | Todos os `.NSN` e `.ddm` | Prefixo de campos financeiros: `VLR-BRUTO`, `VLR-LIQUIDO`, `VLR-DESCONTO`, `VLR-BASE`. Formato `N9.2` |
| 9   | `NUM` | Número (sequencial ou identificador) | Todos os `.NSN` e `.ddm` | Prefixo de campos identificadores: `NUM-PAGTO`, `NUM-INSCRICAO`, `NUM-DEPENDENTES`, `NUM-OB-SIAFI` |
| 10  | `COD` | Código | Todos os `.NSN` e `.ddm` | Prefixo de campos de código: `COD-PROGRAMA`, `COD-BANCO`, `COD-REGIAO`, `COD-RETORNO`, `COD-ACAO` |
| 11  | `DT` | Data | Todos os `.NSN` e `.ddm` | Prefixo de campos de data no formato `N8` (`AAAAMMDD`): `DT-NASCIMENTO`, `DT-GERACAO`, `DT-PAGAMENTO` |
| 12  | `HR` | Hora | `BATCHCON.NSN`, `RELAUDIT.NSN`, `AUDITORIA.ddm` | Prefixo de campos de hora no formato `N6` (`HHMMSS`): `HR-EVENTO`, `HR-INCLUSAO`, `HR-ULT-ALTERACAO` |
| 13  | `QTD` | Quantidade | `BATCHCON.NSN`, `BATCHPGT.NSN`, `BATCHREL.NSN`, `RELAUDIT.NSN` | Prefixo de contadores: `QTD-LIDOS`, `QTD-CONCILIADOS`, `QTD-DIVERGENTES`, `QTD-GERADOS` |
| 14  | `PCT` | Percentual | `CALCDSCT.NSN`, `PAGAMENTO.ddm`, `PROGRAMA-SOCIAL.ddm` | Percentual de desconto ou reajuste. Formato `N3.2`: `PCT-DSCT`, `PCT-REAJUSTE-ANUAL`, `PCT-DESCONTO` |
| 15  | `IND` | Indicador (flag booleano ou categórico) | `CALCCORR.NSN`, `BENEFICIARIO.ddm`, `PROGRAMA-SOCIAL.ddm` | Campos flag: `IND-CORRIGIDO`, `IND-BIOMETRIA` (S/N/P), `IND-DEFICIENCIA` (S/N), `IND-EXIGE-ESCOLA` (S/N) |
| 16  | `SIT` | Situação (status do registro) | `BENEFICIARIO.ddm`, `PAGAMENTO.ddm`, `PROGRAMA-SOCIAL.ddm` | Estado atual de uma entidade. `SIT-BENEFICIARIO`: A=Ativo, S=Suspenso, C=Cancelado, I=Inativo, D=Desligado. `SIT-PAGAMENTO`: P/G/E/C/D/X/R |
| 17  | `GRP` | Grupo (estrutura agrupadora de campos) | `BENEFICIARIO.ddm`, `PAGAMENTO.ddm`, `AUDITORIA.ddm` | Campos agrupadores sem tipo próprio (`-`): `GRP-ENDERECO`, `GRP-DEPENDENTE`, `GRP-DESCONTO`, `GRP-ANTES`, `GRP-DEPOIS` |
| 18  | `TAB` | Tabela (array de valores em memória) | `CALCBENF.NSN`, `CALCCORR.NSN`, `VALBENEF.NSN` | Arrays locais usados em cálculo: `#TAB-REG` (fatores regionais/27), `#IPCA-ANO` (índices mensais), `#UF-TAB` (UFs válidas/27) |
| 19  | `ALIQ` | Alíquota | `CALCDSCT.NSN` | Percentual de contribuição compulsória por faixa. Array `#ALIQ-CONTRIB(N3.2/4)`: 3%, 5%, 7%, 9% |
| 20  | `PGTO` | Pagamento | `BATCHPGT.NSN`, `BATCHCON.NSN`, `RELPGT.NSN`, `PAGAMENTO.ddm` | Abreviação em nomes de variáveis e programas: `NUM-PAGTO`, `STATUS-PGTO`, `TIPO-PGTO` |
| 21  | `BENEF` | Beneficiário | `CALCBENF.NSN`, `CONSBENF.NSN`, `VALBENEF.NSN`, `BENEFICIARIO.ddm` | Pessoa cadastrada que recebe benefício social. Arquivo Adabas FNR 150, ~4,2 milhões de registros |
| 22  | `DSCT` | Desconto | `CALCDSCT.NSN`, `PAGAMENTO.ddm`, `BENEFICIARIO.ddm` | Dedução sobre o valor bruto. `TIPO-DSCT`: C=Contribuição, I=Imposto, J=Judicial, S=Sindical, P=Pensão, A=Administrativo |
| 23  | `COMP` / `COMPETENCIA` | Competência (mês/ano de referência do benefício) | Todos os `.NSN`, `PAGAMENTO.ddm` | Período de referência do pagamento. Formato `N6` (`AAAAMM`). Comentado como `/* AAAAMM */` em `CALCBENF.NSN` |
| 24  | `OPER` | Operação | `CADBENEF.NSN`, `CADPROG.NSN` | Código da operação na tela de cadastro: `I`=Inclusão, `A`=Alteração, `C`=Consulta |
| 25  | `ARQ` | Arquivo Adabas (referenciado por número FNR) | Múltiplos `.NSN` (comentários `ARQ 150`, `155`, `160`, `170`) | Referência ao arquivo pelo número: 150=BENEFICIARIO, 151=PROGRAMA-SOCIAL, 152=PAGAMENTO, 153=AUDITORIA |
| 26  | `CALC` | Cálculo (prefixo de subprogramas de cálculo) | `CALCBENF.NSN`, `CALCDSCT.NSN`, `CALCCORR.NSN` | Prefixo de programas que executam cálculo: benefício (`CALCBENF`), descontos (`CALCDSCT`), correção IPCA (`CALCCORR`) |
| 27  | `CAD` | Cadastro (prefixo de programas de manutenção) | `CADBENEF.NSN`, `CADDEPEND.NSN`, `CADPROG.NSN` | Prefixo de programas de manutenção: beneficiário (`CADBENEF`), dependentes (`CADDEPEND`), programas sociais (`CADPROG`) |
| 28  | `VAL` | Validação (prefixo de subprogramas de validação) | `VALBENEF.NSN`, `VALDOCS.NSN`, `VALELEG.NSN` | Prefixo de programas de validação: dados cadastrais (`VALBENEF`), documentos (`VALDOCS`), elegibilidade (`VALELEG`) |
| 29  | `REL` | Relatório (prefixo de programas de relatório) | `BATCHREL.NSN`, `RELAUDIT.NSN`, `RELPGT.NSN` | Prefixo de geradores de relatório: consolidado (`BATCHREL`), auditoria (`RELAUDIT`), pagamentos (`RELPGT`) |
| 30  | `ELEG` | Elegibilidade | `VALELEG.NSN`, `CADPROG.NSN`, `PROGRAMA-SOCIAL.ddm` | Conjunto de critérios que um beneficiário deve atender: renda máxima per capita, faixa etária, documentação, região |
| 31  | `FATOR-K` | Fator K — fator de correção especial não documentado | `CADPROG.NSN`, `PROGRAMA-SOCIAL.ddm` | **HIPÓTESE** — multiplicador inserido em ago/2008 "atendendo solicitação SENARC". Afeta `VLR-BASE` no cadastro de programas. Campo marcado `>>> NAO DOCUMENTADO <<<` no DDM |
| 32  | `IPCA` | Índice Nacional de Preços ao Consumidor Amplo | `CALCCORR.NSN` | Índice de inflação para correção retroativa de pagamentos. Tabela `#IPCA-ANO(N3.6/10,12)` com valores mensais por ano (2010–2014) |
| 33  | `SIAFI` | Sistema Integrado de Administração Financeira do Governo Federal | `PAGAMENTO.ddm` | Sistema de pagamentos federais integrado ao SIFAP desde 2002. Campos: `NUM-OB-SIAFI`, `NUM-NE-SIAFI`, `COD-UG-EMITENTE`, `COD-GESTAO`, `SIT-INTEG-SIAFI` |
| 34  | `PBF` | Programa Bolsa Família | `PROGRAMA-SOCIAL.ddm` | Exemplo de valor do campo `SIGLA-PROGRAMA`. Programa assistencial de transferência de renda |
| 35  | `BPC` | Benefício de Prestação Continuada | `PROGRAMA-SOCIAL.ddm` | Exemplo de valor do campo `SIGLA-PROGRAMA`. Benefício previdenciário para idosos e pessoas com deficiência |
| 36  | `PETI` | Programa de Erradicação do Trabalho Infantil | `PROGRAMA-SOCIAL.ddm` | Exemplo de valor do campo `SIGLA-PROGRAMA` |
| 37  | `MDS` | Ministério do Desenvolvimento Social | `PROGRAMA-SOCIAL.ddm` | Campo `ORGAO-RESPONSAVEL` referencia `MDS/MDAS` como responsável pelos programas sociais |
| 38  | `SENARC` | Secretaria Nacional de Renda de Cidadania | `PROGRAMA-SOCIAL.ddm` | Secretaria que autorizou a criação do `FATOR-K` em 2008. Responsável pela coordenação de benefícios |
| 39  | `CGTI` | Coordenação Geral de Tecnologia da Informação | `BENEFICIARIO.ddm` | Órgão que emite portarias de controle do DDM. Citado em `PORT. 847/2003` que proíbe alteração de ordem de campos |
| 40  | `TCU` | Tribunal de Contas da União | `AUDITORIA.ddm` | Determinou a obrigatoriedade do log de auditoria via `IN-TCU 63/2010`. Arquivo 153 tem retenção mínima de 10 anos |
| 41  | `FEBRABAN` | Federação Brasileira de Bancos | `PAGAMENTO.ddm` | Entidade que define a tabela de códigos de banco. Campo `COD-BANCO (A3)` usa a codificação FEBRABAN |
| 42  | `OB` | Ordem Bancária | `PAGAMENTO.ddm` | Instrumento de pagamento no SIAFI. Campo `NUM-OB-SIAFI (A12)` |
| 43  | `NE` | Nota de Empenho | `PAGAMENTO.ddm` | Documento de compromisso orçamentário no SIAFI. Campo `NUM-NE-SIAFI (A12)` |
| 44  | `UG` | Unidade Gestora | `PAGAMENTO.ddm` | Unidade orçamentária emissora do pagamento. Campo `COD-UG-EMITENTE (A6)` |
| 45  | `FNR` | File Number (número do arquivo Adabas) | Todos os `.ddm` | Identificador numérico do arquivo no Adabas: `150`=BENEFICIARIO, `151`=PROGRAMA-SOCIAL, `152`=PAGAMENTO, `153`=AUDITORIA |
| 46  | `DBID` | Database ID (identificador do banco de dados Adabas) | Todos os `.ddm` | Todos os arquivos SIFAP usam `DBID: 57` |
| 47  | `DE` | Descriptor (campo indexado para busca no Adabas) | Todos os `.ddm` | Marcador `(DE)` indica campo com índice. Ex: `NUM-CPF`, `DT-EVENTO`, `COD-PROGRAMA`, `DT-CADASTRO` |
| 48  | `PE` | Periodic Group (grupo periódico — estrutura repetitiva) | `BENEFICIARIO.ddm`, `PAGAMENTO.ddm`, `PROGRAMA-SOCIAL.ddm`, `CADDEPEND.NSN` | Grupo de campos que se repete N vezes por registro (análogo a array de structs). Max: 10 dependentes, 8 descontos, 5 faixas de cálculo, 6 regiões |
| 49  | `MU` | Multiple Value Field (campo com múltiplos valores) | `PAGAMENTO.ddm`, `AUDITORIA.ddm`, `PROGRAMA-SOCIAL.ddm` | Campo simples com até N valores (análogo a array primitivo). `TIPO-DSCT-APLIC` aceita até 8 tipos; `CAMPO-ALTERADO-ANT` até 20 |
| 50  | `ISN` | Internal Sequence Number (chave primária interna do Adabas) | `BENEFICIARIO.ddm` | Chave primária gerada pelo Adabas. `NUM-INSCRICAO` é descrito como "ISN ALTERNATIVO / MATRICULA" |
| 51  | `IR` (desconto) | Imposto de Renda Retido na Fonte | `PAGAMENTO.ddm`, `PROGRAMA-SOCIAL.ddm` | Código de tipo de desconto no campo `TIPO-DESCONTO`. **ATENÇÃO**: colide com `IR`=Irmão em parentesco (ver #56) |
| 52  | `JD` | Judicial (desconto por ordem judicial) | `PAGAMENTO.ddm`, `CALCDSCT.NSN`, `PROGRAMA-SOCIAL.ddm` | Tipo de desconto por decisão judicial. Quando `TIPO-DESCONTO = 'JD'`, o campo `NUM-PROCESSO (A20)` torna-se obrigatório |
| 53  | `CS` | Consignado (desconto em folha) | `PAGAMENTO.ddm`, `PROGRAMA-SOCIAL.ddm` | Tipo de desconto `CS` — empréstimo consignado descontado do benefício |
| 54  | `PA` | Pensão Alimentícia | `PAGAMENTO.ddm`, `PROGRAMA-SOCIAL.ddm` | Tipo de desconto `PA` por decisão judicial de pensão alimentícia |
| 55  | `FI` | Filho (grau de parentesco) | `CADDEPEND.NSN`, `BENEFICIARIO.ddm` | Código `FI` no campo `PARENTESCO (A2)`. Conjunto completo: `FI`=Filho, `CO`=Cônjuge (no DDM: `CJ`), `IR`=Irmão, `OU`=Outros |
| 56  | `IR` (parentesco) | Irmão (grau de parentesco) | `CADDEPEND.NSN` | **AMBIGUIDADE**: `IR` em `PARENTESCO` = Irmão; `IR` em `TIPO-DESCONTO` = Imposto de Renda. Mesmo código, contextos incompatíveis |
| 57  | `EST-CIVIL` | Estado Civil | `BENEFICIARIO.ddm` | Campo `AH (A1)`: `S`=Solteiro, `C`=Casado, `D`=Divorciado, `V`=Viúvo, `U`=União Estável |
| 58  | `MOT` | Motivo | `BENEFICIARIO.ddm`, `PAGAMENTO.ddm` | Prefixo de campos de motivo: `MOT-SITUACAO (A3)`, `MOT-CANCELAMENTO (A3)`. Valores definidos em "TAB INTERNA" não encontrada nos DDMs |
| 59  | `CTPS` | Carteira de Trabalho e Previdência Social | `VALDOCS.NSN` | Documento opcional validado em `VALDOCS.NSN`. Campo local `#CTPS (A15)` |
| 60  | `TITULO` | Título de Eleitor | `VALDOCS.NSN` | Documento opcional validado em `VALDOCS.NSN`. Campo local `#TITULO (A12)` |
| 61  | `IBGE` | Instituto Brasileiro de Geografia e Estatística | `BENEFICIARIO.ddm` | Campo `COD-IBGE (N7)` armazena o código oficial do município segundo classificação IBGE |
| 62  | `COD-PERFIL` | Código de Perfil do usuário | `AUDITORIA.ddm` | Campo `EC (A3)`: `ADM`=Administrador, `OPR`=Operador, `CON`=Consultor, `AUD`=Auditor, `SUP`=Supervisor |
| 63  | `TS` | Timestamp | `AUDITORIA.ddm` | Campo `TS-EVENTO (N14)` — precisão total `AAAAMMDDHHMMSS`, complementar aos campos `DT-EVENTO` e `HR-EVENTO` |
| 64  | `COD-ACAO` | Código de Ação de auditoria | `AUDITORIA.ddm`, `RELAUDIT.NSN` | Campo `BA (A2)`: `IN`=Inclusão, `AL`=Alteração, `EX`=Exclusão, `CO`=Consulta, `LG`=Login, `LO`=Logout, `BT`=Batch, `ER`=Erro, `AU`=Autorização, `RE`=Rejeição |
| 65  | `BT` | Batch (tipo de ação de auditoria) | `AUDITORIA.ddm` | Código `BT` em `COD-ACAO` identifica eventos gerados por processamento não interativo. Habilita campos `NUM-CICLO-BATCH`, `NOM-JOB-BATCH`, `SIT-BATCH` |
| 66  | `JES2` | Job Entry Subsystem 2 (agendador de jobs do mainframe IBM) | `AUDITORIA.ddm` | Referenciado em `NOM-JOB-BATCH (A16)` como "NOME JOB JES2/JCL". Identifica o job que gerou o evento de auditoria |
| 67  | `UUID` | Universally Unique Identifier | `AUDITORIA.ddm` | Campo `ID-CORRELACAO (A36)` armazena UUID para rastrear operações compostas que afetam múltiplos registros |
| 68  | `FATOR-REG` | Fator Regional | `CALCBENF.NSN`, `BATCHPGT.NSN`, `PROGRAMA-SOCIAL.ddm` | Multiplicador aplicado ao valor base por estado. Array `#TAB-REG(N3.4/27)` — uma entrada por UF, variando de 1.20 a 1.40 |
| 69  | `FATOR-FAM` | Fator Familiar | `CALCBENF.NSN`, `BATCHPGT.NSN` | Multiplicador baseado no número de dependentes (`NUM-DEPENDENTES`). Calculado internamente em `CALCBENF.NSN` |
| 70  | `FATOR-IDADE` | Fator Idade | `CALCBENF.NSN`, `BATCHPGT.NSN` | Multiplicador baseado na idade calculada a partir de `DT-NASCIMENTO`. Parte da fórmula do valor bruto do benefício |

> Adicione mais linhas conforme necessário. Não se limite a 30!

## Exemplo de linha bem preenchida

| #   | Termo  | Expansão | Programa                        | Contexto                                                                                                         |
| --- | ------ | -------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1   | `DSCT` | Desconto | `CALCDSCT.NSN`, `PAGAMENTO.ddm` | Tipo de dedução aplicada sobre valor bruto do pagamento. Tipos: 'J' (judicial), 'I' (imposto), 'T' (trabalhista) |

## Observações

- Anote aqui qualquer padrão de nomenclatura que o time identificou:
- Convenções de prefixo/sufixo encontradas:
- Termos ambíguos que precisam de validação com especialista:

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
<a href="business-rules-catalog.md"><strong>business-rules-catalog.md</strong></a><br/>
<sub>Catálogo de regras.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

