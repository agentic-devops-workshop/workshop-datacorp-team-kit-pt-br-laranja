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

> Faixas de linha são aproximadas (contagem a partir do início do arquivo, incluindo o cabeçalho de comentários). Par 2 deve validar via `grep -n` antes da Passagem H1.

| ID     | Regra de Negócio                                                                                                                       | Programa Fonte                                                                | Campos DDM                                                              | Nível de Risco | Notas                                                                                       |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------- |
| BR-001 | Apenas registros CNAB tipo `'3'` (detalhe) são conciliados; cabeçalhos/trailers são descartados                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L110-L113`         | —                                                                       | MÉDIO          | Layout CNAB 240 BB                                                                          |
| BR-002 | Valores no arquivo CNAB chegam em **centavos** e devem ser divididos por 100 para virarem reais                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L124-L126`         | `PAGAMENTO.VLR-LIQUIDO`                                                 | CRÍTICO        | Erro de unidade = pagamento 100× errado                                                     |
| BR-003 | Conciliação bancária só ocorre se casarem **3 chaves**: `NUM-PAGTO` + `CPF-BENEF` + `COMPETENCIA`                                      | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L131-L137`         | `PAGAMENTO.NUM-PAGTO`, `CPF-BENEF`, `COMPETENCIA`                       | ALTO           | Match parcial é tratado como "não encontrado"                                                |
| BR-004 | Tolerância de divergência de valor na conciliação é R$ 0,01 (diferença ≤ 1 centavo é conciliada)                                       | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L147-L152`         | `PAGAMENTO.VLR-LIQUIDO`                                                 | CRÍTICO        | Magic number sem documentação                                                                |
| BR-005 | Código de retorno bancário → status do pagamento: `'00'`→`'P'` (Pago), `'01'`→`'D'` (Devolvido), `'02'`→`'E'` (Estornado)              | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L160-L182`         | `PAGAMENTO.STATUS-PGTO`, `COD-RETORNO`                                  | CRÍTICO        | Outros códigos viram apenas WRITE em log — pagamento fica em status anterior (silencioso)   |
| BR-006 | Pagamento conciliado tem `COD-BANCO` fixado em `1` (hardcoded Banco do Brasil)                                                         | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L165`              | `PAGAMENTO.COD-BANCO`                                                   | ALTO           | Impede multi-banco apesar do histórico de "INC BANCO REAL"                                  |
| BR-007 | Toda conciliação (sucesso ou divergência) gera registro em `AUDITORIA` com ação `'CO'` ou `'DV'`                                       | `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN#L200-L235`         | `AUDITORIA.ACAO`, `TABELA-REF`, `CHAVE-REF`                             | ALTO           | LGPD/compliance — preservar na modernização                                                  |
| BR-008 | Mapeamento `COD-REGIAO` → macro-região por faixa: 1-5=Norte, 6-10=Nordeste, 11-15=Sudeste, 16-20=Sul, 21+=Centro-Oeste                 | `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L98-L114`          | `BENEFICIARIO.COD-REGIAO`                                               | ALTO           | Beneficiário sem região (cod=0) cai em Centro-Oeste por fallback                            |
| BR-009 | Relatório consolidado **arredonda** valor bruto (`+ 0.005`) enquanto pagamento real é **truncado** → totais não batem                  | `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L118-L121`         | `PAGAMENTO.VLR-BRUTO`                                                   | CRÍTICO        | Inconsistência financeira plantada — ver MYS-001                                            |
| BR-010 | Domínio fechado de status do pagamento: `G`=Gerado, `P`=Pago, `C`=Cancelado, `D`=Devolvido, `E`=Estornado                              | `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN#L132-L145`         | `PAGAMENTO.STATUS-PGTO`                                                 | ALTO           | Status desconhecido vira "Gerado" silenciosamente (fallback `NONE`)                          |
| BR-011 | Geração de pagamento só ocorre para beneficiário com `STATUS = 'A'` (Ativo) e programa com `STATUS-PROG = 'A'`                         | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L150-L192`         | `BENEFICIARIO.STATUS`, `PROGRAMA-SOCIAL.STATUS-PROG`                    | CRÍTICO        | Validação dupla — beneficiário e programa                                                    |
| BR-012 | Idempotência mensal: não gera novo pagamento se já existe um para o mesmo CPF na mesma competência                                     | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L155-L165`         | `PAGAMENTO.CPF-BENEF`, `COMPETENCIA`                                    | CRÍTICO        | Garantia anti-duplicidade do batch                                                           |
| BR-013 | Fator regional aplicado ao benefício é indexado em tabela hardcoded de 27 posições (valores 1.00–1.40)                                 | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L116-L143`         | `BENEFICIARIO.COD-REGIAO`                                               | CRÍTICO        | Posições 26 e 27 inicializadas mas inalcançáveis (range checado é 1-25) — ver MYS-002       |
| BR-014 | Fator familiar escalonado: 0 dep=1.0; 1-2 dep=1.0+(n×0.05); 3-4 dep=1.10+((n-2)×0.03); ≥5 dep=1.16+((n-4)×0.02)                        | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L210-L222`         | `BENEFICIARIO.NUM-DEPENDENTES`                                          | CRÍTICO        | Regra escalonada — testar com 0, 2, 4, 5, 10 dependentes                                    |
| BR-015 | Faixas de renda familiar (5 faixas, fator decrescente): ≤300→1.00; ≤600→0.85; ≤1000→0.70; ≤1500→0.55; >1500→0.40                       | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L155-L164`         | `BENEFICIARIO.RENDA-FAMILIAR`                                           | CRÍTICO        | Valores políticos — confirmar com PO antes de mudar                                          |
| BR-016 | Fator idade: ≥65 anos=1.15 (idoso); ≥60=1.10; <18=1.05 (menor); demais=1.00                                                            | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L230-L240`         | `BENEFICIARIO.DT-NASCIMENTO`                                            | ALTO           | Idade calculada só por ano (ignora mês/dia) — ver MYS-003                                   |
| BR-017 | Fórmula do benefício bruto = `VLR-BASE × fator_reg × fator_fam × fator_renda × fator_idade × (1 + fator_reajuste)`                     | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L245-L250`         | `PROGRAMA-SOCIAL.VLR-BASE`, `FATOR-REAJUSTE`                            | CRÍTICO        | Regra-mãe do sistema — replicar bit-a-bit                                                    |
| BR-018 | Valores monetários são **truncados** (não arredondados) para 2 casas decimais via `INT(x×100)/100`                                     | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L251-L253`         | `PAGAMENTO.VLR-BRUTO`, `VLR-LIQUIDO`                                    | CRÍTICO        | Política financeira oficial — diverge do BATCHREL (ver BR-009)                              |
| BR-019 | 13º salário pago **apenas em dezembro** (mês=12), fórmula = `VLR-BASE × fator_reg × fator_idade` (sem fam, renda ou reajuste)          | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L260-L266`         | `PAGAMENTO.TIPO-PGTO`, `VLR-BRUTO`                                      | CRÍTICO        | Tipo de pagamento marcado `'D'` em dezembro                                                  |
| BR-020 | Abono de 15% sobre benefício mensal pago em dezembro **apenas para programas com `TIPO = 'A'`**                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L267-L271`         | `PROGRAMA-SOCIAL.TIPO`, `PAGAMENTO.VLR-ABONO`                           | CRÍTICO        | Magic number 0.15 sem comentário                                                             |
| BR-021 | Desconto único de 3% aplicado somente quando bruto > R$ 500,00 (sem faixas progressivas)                                               | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L282-L286`         | `PAGAMENTO.VLR-DESCONTO`                                                | CRÍTICO        | Magic numbers 500.00 e 0.03 — confirmar com PO                                              |
| BR-022 | Valor líquido nunca pode ser negativo — piso em zero                                                                                   | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L290-L292`         | `PAGAMENTO.VLR-LIQUIDO`                                                 | ALTO           | Proteção contra desconto > bruto                                                             |
| BR-023 | Pagamento criado tem status inicial `'G'` (Gerado), aguardando processamento bancário downstream                                       | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L305`              | `PAGAMENTO.STATUS-PGTO`                                                 | ALTO           | Máquina de estados: G → P/D/E (via BATCHCON)                                                |
| BR-024 | Competência derivada da data de execução: `(ano × 100) + mês` no formato AAAAMM                                                        | `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN#L85-L87`           | `PAGAMENTO.COMPETENCIA`                                                 | MÉDIO          | Implica execução **no mês de competência** — rodar em janeiro gera comp do mês anterior?    |
| BR-025 | Desconto judicial (`TIPO-DSCT='J'`) NÃO é submetido ao teto de 30% — todos os demais são | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L121-L158` | `PAGAMENTO.VLR-BRUTO`, `BENEFICIARIO.TIPO-DSCT` | CRÍTICO | Aplicar teto = descumprir decisão judicial |
| BR-026 | Desconto sindical (`TIPO-DSCT='S'`) é sempre 1% do bruto, ignorando `VLR-DSCT`/`PCT-DSCT` | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L145-L147` | `BENEFICIARIO.TIPO-DSCT`, `PAGAMENTO.VLR-BRUTO` | ALTO | Magic number 0.01 hardcoded |
| BR-027 | Desconto só é aplicado se `hoje ∈ [DT-INICIO-DSCT, DT-FIM-DSCT]` (DT-FIM=0 ⇒ indeterminado) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L107-L114` | `BENEFICIARIO.DT-INICIO-DSCT`, `DT-FIM-DSCT` | ALTO | Vigência por item do PE de descontos |
| BR-028 | Contribuição social compulsória em 4 faixas: ≤500→3%, ≤1000→5%, ≤2000→7%, >2000→9% | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L51-L58, L165-L172` | `PAGAMENTO.VLR-BRUTO` | CRÍTICO | Tabela em memória; alterar alíquotas exige novo build |
| BR-029 | Pagamento já marcado `IND-CORRIGIDO='S'` NÃO é recalculado em correção retroativa | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L96-L99` | `PAGAMENTO.IND-CORRIGIDO` | ALTO | Idempotência da correção |
| BR-030 | Período de correção exige `COMP-INI ≤ COMP-FIM`; senão aborta com `ESCAPE ROUTINE` | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L85-L89` | — | MÉDIO | Validação de input |
| BR-031 | Índice IPCA acumulado mês-a-mês: `IND_ACUM = ∏ (1 + ipca[ano,mês])` aplicado a `VLR-BRUTO` | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L102-L160` | `PAGAMENTO.VLR-BRUTO`, `VLR-CORRECAO` | CRÍTICO | Tabelas IPCA hardcoded só até 2014 — dívida técnica |
| BR-032 | Correção retroativa só grava se `VLR-CORR − VLR-ORIG > 0`; diferença ≤ 0 = SKIP | `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN#L114-L125` | `PAGAMENTO.VLR-CORRECAO`, `DT-CORRECAO` | ALTO | Protege contra "correção negativa" |
| BR-033 | Cálculo de benefício mensal só ocorre se `BENEFICIARIO.STATUS='A'` | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L270-L273` | `BENEFICIARIO.STATUS` | CRÍTICO | Reforça BR-011 — validação dupla |
| BR-034 | Competência inválida (`MES<1` ou `MES>12`) aborta com `ESCAPE ROUTINE` | `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN#L253-L258` | `PAGAMENTO.COMPETENCIA` | MÉDIO | Validação AAAAMM |
| BR-035 | CPF obrigatório e validado por dígito verificador módulo 11 na inclusão de beneficiário | `01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN#L243-L295` | `BENEFICIARIO.CPF` | CRÍTICO | Algoritmo replicado em VALBENEF e VALDOCS — risco de divergência |
| BR-036 | Inclusão de beneficiário com CPF já existente é bloqueada; alteração exige beneficiário existir | `01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN#L141-L151` | `BENEFICIARIO.CPF` | ALTO | Unicidade por CPF |
| BR-037 | Novo beneficiário recebe `STATUS='A'`; se `IDADE>75` no cadastro, status passa silenciosamente para `'S'` | `01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN#L165-L174` | `BENEFICIARIO.STATUS`, `DT-NASCIMENTO` | CRÍTICO | Regra demográfica oculta — ver MYS-014 |
| BR-038 | Cadastro de dependente bloqueado se titular tem `STATUS='C'` ou `'D'` | `01-arqueologia/legado-sifap/natural-programs/CADDEPEND.NSN#L52-L55` | `BENEFICIARIO.STATUS` | ALTO | Protege integridade do PE group |
| BR-039 | Limite de dependentes hardcoded em **5** no programa, embora o DDM permita até **10** posições no PE | `01-arqueologia/legado-sifap/natural-programs/CADDEPEND.NSN#L59-L62` | `BENEFICIARIO.DEPENDENTES (PE)` | ALTO | Inconsistência código vs DDM — ver MYS-015 |
| BR-040 | `PARENTESCO` restrito ao domínio fechado `{FI, CO, IR, OU}` (Filho/Cônjuge/Irmão/Outro) | `01-arqueologia/legado-sifap/natural-programs/CADDEPEND.NSN#L82-L86` | `BENEFICIARIO.PARENTESCO` | MÉDIO | Enum no domínio |
| BR-041 | Dependente com mesmo CPF de outro já cadastrado para o titular é bloqueado | `01-arqueologia/legado-sifap/natural-programs/CADDEPEND.NSN#L92-L102` | `BENEFICIARIO.CPF-DEP` | ALTO | Anti-fraude |
| BR-042 | Ao cadastrar programa, `VLR-BASE` armazenado = informado × `(1 + FATOR-REAJUSTE × 0.347215)` | `01-arqueologia/legado-sifap/natural-programs/CADPROG.NSN#L75-L78` | `PROGRAMA-SOCIAL.VLR-BASE`, `FATOR-REAJUSTE` | CRÍTICO | Constante mágica `0.347215` sem documentação — ver MYS-016 |
| BR-043 | Consulta de beneficiário suporta busca por CPF (`TIPO-BUSCA='C'`) ou NIS (`'N'`); default = CPF | `01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L75-L92` | `BENEFICIARIO.CPF`, `NIS` | MÉDIO | NIS é descritor alternativo no DDM |
| BR-044 | Tela mascara CPF para `***.***.XXX-XX` (LGPD); algoritmo varia conforme padding e admite bug conhecido | `01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L168-L189` | `BENEFICIARIO.CPF` | ALTO | Comentário "INCONSISTENCIA CONHECIDA" no fonte — ver MYS-017 |
| BR-045 | Histórico exibido na consulta limita aos **12** pagamentos mais recentes | `01-arqueologia/legado-sifap/natural-programs/CONSBENF.NSN#L138-L156` | `PAGAMENTO.CPF-BENEF`, `COMPETENCIA` | BAIXO | Magic number 12 sem documentação |
| BR-046 | Relatório de auditoria oculta silenciosamente eventos com `ACAO='EX'` (Exclusão) | `01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L98-L103` | `AUDITORIA.ACAO` | CRÍTICO | Compliance: trilha deveria mostrar tudo — ver MYS-018 |
| BR-047 | Período default do relatório de auditoria começa em **19970101** (início do SIFAP) | `01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN#L77-L82` | — | BAIXO | Magic date hardcoded |
| BR-048 | Relatório analítico de pagamentos faz control-break por `COD-PROGRAMA` com subtotal de qtd/bruto/líquido | `01-arqueologia/legado-sifap/natural-programs/RELPGT.NSN#L93-L99, L189-L196` | `PAGAMENTO.COD-PROGRAMA` | MÉDIO | Padrão mainframe — 66 linhas/página |
| BR-049 | CPF com 11 dígitos iguais é inválido, EXCETO se os 3 primeiros forem `000` (libera como válido) | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L218-L234` | `BENEFICIARIO.CPF` | CRÍTICO | Backdoor "TESTE GOVERNO" — ver MYS-019 / EGG-002 |
| BR-050 | Data de nascimento válida: `1900 ≤ ANO ≤ ano atual`, `1 ≤ MES ≤ 12`, `1 ≤ DIA ≤ DIAS_MES` | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L257-L274` | `BENEFICIARIO.DT-NASCIMENTO` | ALTO | Fevereiro fixo em 29 dias (ignora regra bissexta real) — ver MYS-020 |
| BR-051 | Nome do beneficiário deve conter ao menos um espaço (heurística "nome+sobrenome") | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L278-L292` | `BENEFICIARIO.NOME` | MÉDIO | Validação fraca |
| BR-052 | UF válida pertence ao conjunto fechado de 27 unidades federativas brasileiras | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L66-L92, L173-L186` | `BENEFICIARIO.UF` | MÉDIO | Tabela hardcoded |
| BR-053 | `VALDOCS` aceita 8 prefixos CPF SEM verificação de dígito: `000, 001, 002, 010, 011, 099, 100, 999` | `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN#L36-L43, L171-L188` | `BENEFICIARIO.CPF` | CRÍTICO | Backdoor de teste — ver MYS-021 / EGG-002 |
| BR-054 | RG válido deve ter ≥ 5 caracteres (medido pela posição do primeiro espaço) | `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN#L148-L165` | `BENEFICIARIO.RG` | BAIXO | Sem dígito verificador |
| BR-055 | Beneficiário com `COD-REGIAO=99` é automaticamente elegível, pulando TODAS as demais validações | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L91-L96` | `BENEFICIARIO.COD-REGIAO` | CRÍTICO | Região "internacional/diplomática" — ver MYS-022 |
| BR-056 | Status diferente de `'A'` torna o beneficiário inelegível com motivo específico (S/C/D/I) | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L98-L117` | `BENEFICIARIO.STATUS` | CRÍTICO | Reforça BR-011 |
| BR-057 | Elegibilidade por tipo: A renda≤R$600 OU ≥1 dep+docs; P exige idade≥60; T exige `16 ≤ idade ≤ 65` | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L139-L177` | `PROGRAMA-SOCIAL.TIPO`, `BENEFICIARIO.RENDA-FAMILIAR`, `NUM-DEPENDENTES`, `DT-NASCIMENTO` | CRÍTICO | Magic numbers 600.00, 60, 16, 65 |
| BR-058 | `COD-ELEGIBILIDADE` posicional de 5 letras: 1ª letra `R` exige NIS; 2ª letra `D` exige ≥1 dep | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L194-L213` | `PROGRAMA-SOCIAL.COD-ELEGIBILIDADE`, `BENEFICIARIO.NIS`, `NUM-DEPENDENTES` | ALTO | Encoding posicional sem documentação |
| BR-059 | Programa com `STATUS-PROG ≠ 'A'` (inativo) bloqueia toda análise de elegibilidade | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L86-L89` | `PROGRAMA-SOCIAL.STATUS-PROG` | ALTO | Reforço de BR-011 |

> Adicione mais linhas conforme necessário. Lembre-se: existem **10 regras escondidas** no código!

## Exemplo de linha bem preenchida

| ID     | Regra de Negócio                                                                        | Programa Fonte                                   | Campos DDM                                                               | Nível de Risco | Notas                                      |
| ------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ | -------------- | ------------------------------------------ |
| BR-EX  | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L142-L148` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO        | Linha ilustrativa apenas (a regra real está em BR-025) |

## Regras por Categoria

### Cálculos Financeiros

- BR-002 (centavos → reais), BR-004 (tolerância R$ 0,01), BR-009 (round vs truncate), BR-013 (fator regional), BR-014 (fator familiar), BR-015 (faixas de renda), BR-016 (fator idade), BR-017 (fórmula principal), BR-018 (truncamento), BR-019 (13º), BR-020 (abono 15%), BR-021 (desconto 3%), BR-022 (piso zero), BR-025 (teto judicial), BR-026 (sindical 1%), BR-028 (contribuição 4 faixas), BR-031 (IPCA acumulado), BR-042 (fator K 0.347215)

### Validações de Status

- BR-005 (cod retorno → status), BR-010 (domínio fechado de status), BR-011 (apenas ativos), BR-023 (status inicial 'G'), BR-033 (status 'A' p/ benefício), BR-056 (status ≠ 'A' inelegível), BR-059 (programa inativo bloqueia)

### Regras de Autorização / Elegibilidade

- BR-055 (região 99 elegível por default), BR-057 (elegibilidade por tipo A/P/T), BR-058 (COD-ELEG posicional R/D), BR-049 (CPF backdoor 000), BR-053 (8 prefixos CPF backdoor)

### Regras de Negócio Temporais

- BR-012 (idempotência mensal), BR-019 (13º só em dezembro), BR-024 (cálculo de competência), BR-027 (vigência de desconto), BR-029 (idempotência de correção), BR-031 (IPCA acumulado mês-a-mês)

### Cadastro

- BR-035 (CPF mod-11 obrigatório), BR-036 (unicidade CPF), BR-037 (idade>75 → suspenso silencioso), BR-038/039/040/041 (regras de dependente), BR-042 (fator K no programa)

### Consulta e Relatórios

- BR-043 (busca CPF/NIS), BR-044 (máscara CPF LGPD), BR-045 (histórico 12 pgtos), BR-046 (filtra exclusões da auditoria), BR-047 (default 19970101), BR-048 (control-break por programa)

### Validações de Documentos

- BR-050 (data de nascimento), BR-051 (nome com espaço), BR-052 (27 UFs), BR-054 (RG ≥ 5 caracteres)

## Resumo Estatístico

- Total de regras encontradas: **59** (BR-001 a BR-059)
- Regras críticas: **34**
- Regras com duplicação: **2** (BR-018/BR-009 truncate × round; BR-011/BR-033/BR-056 validação tripla de status)
- Regras sem documentação / magic numbers / backdoors: **15** (BR-004, BR-006, BR-013, BR-016, BR-020, BR-021, BR-026, BR-037, BR-039, BR-042, BR-045, BR-047, BR-049, BR-053, BR-055)

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

