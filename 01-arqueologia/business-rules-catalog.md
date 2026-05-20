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
| BR-060 | CPF deve ser validado pelo algoritmo módulo-11 (dois dígitos verificadores) | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L130-L157` | `BENEFICIARIO.CPF` | CRÍTICO | Implementação padrão Receita Federal |
| BR-061 | CPF com todos os dígitos iguais é inválido, EXCETO se iniciar com 000 (teste governo) | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L136-L142` | `BENEFICIARIO.CPF` | ALTO | Exceção não documentada — possível backdoor |
| BR-062 | Data de nascimento deve ter ano entre 1900 e ano atual | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L163-L166` | `BENEFICIARIO.DT-NASCIMENTO` | MÉDIO | Impede datas absurdas |
| BR-063 | Nome do beneficiário deve conter pelo menos um espaço (nome + sobrenome) | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L180-L190` | `BENEFICIARIO.NOME` | MÉDIO | Adicionada em 2010 por José Ferreira |
| BR-064 | UF deve ser uma das 27 unidades federativas válidas do Brasil | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L107-L117` | `BENEFICIARIO.UF` | MÉDIO | Tabela hardcoded de 27 UFs |
| BR-065 | Status do beneficiário só aceita valores: A (Ativo), S (Suspenso), C (Cancelado), I (Inativo), D (Desligado) | `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L122-L127` | `BENEFICIARIO.STATUS` | ALTO | Define máquina de estados do beneficiário |
| BR-066 | RG deve ter pelo menos 5 caracteres | `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN#L112-L118` | `BENEFICIARIO.RG` | MÉDIO | Validação mínima de formato |
| BR-067 | Documentos com prefixo CPF especial (000, 001, 002, 010, 011, 099, 100, 999) ignoram TODA validação e são aceitos automaticamente | `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN#L125-L135` | `BENEFICIARIO.CPF` | CRÍTICO | Bypass total — risco de segurança se migrado sem controle |
| BR-068 | Beneficiário com código de região 99 (internacional/diplomático) é automaticamente elegível, sem verificar nenhuma outra regra | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L80-L84` | `BENEFICIARIO.COD-REGIAO` | ALTO | Bypass completo de elegibilidade |
| BR-069 | Somente beneficiários com status 'A' (Ativo) são elegíveis; S=suspenso, C/D=cancelado/desligado, I=inativo todos impedem elegibilidade | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L89-L108` | `BENEFICIARIO.STATUS` | ALTO | Pré-requisito para qualquer programa |
| BR-070 | Idade do beneficiário deve estar dentro da faixa etária definida pelo programa (IDADE-MIN e IDADE-MAX) | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L113-L123` | `PROGRAMA-SOCIAL.IDADE-MIN`, `PROGRAMA-SOCIAL.IDADE-MAX` | ALTO | Faixa configurável por programa |
| BR-071 | Renda familiar não pode exceder o teto máximo definido pelo programa (RENDA-MAX) | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L128-L133` | `BENEFICIARIO.RENDA-FAMILIAR`, `PROGRAMA-SOCIAL.RENDA-MAX` | CRÍTICO | Regra financeira de corte |
| BR-072 | Programa tipo 'A' (Assistencial): renda > R$ 600,00 sem dependentes torna inelegível | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L139-L145` | `BENEFICIARIO.RENDA-FAMILIAR`, `BENEFICIARIO.NUM-DEPENDENTES` | CRÍTICO | Valor 600.00 hardcoded — número mágico |
| BR-073 | Programa tipo 'P' (Previdenciário): requer idade mínima de 60 anos | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L151-L155` | `BENEFICIARIO.DT-NASCIMENTO` | ALTO | Regra fixa independente do parâmetro IDADE-MIN |
| BR-074 | Programa tipo 'T' (Trabalho): idade deve estar entre 16 e 65 anos | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L156-L160` | `BENEFICIARIO.DT-NASCIMENTO` | ALTO | Limites legais de idade para trabalho |
| BR-075 | Código de elegibilidade iniciando com 'R' exige NIS cadastrado (não-zero) | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L175-L180` | `BENEFICIARIO.NIS`, `PROGRAMA-SOCIAL.COD-ELEGIBILIDADE` | ALTO | NIS = 0 bloqueia inscrição |
| BR-076 | Código de elegibilidade com 2ª posição = 'D' exige dependentes > 0 | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L182-L187` | `BENEFICIARIO.NUM-DEPENDENTES`, `PROGRAMA-SOCIAL.COD-ELEGIBILIDADE` | MÉDIO | Codificação posicional do campo elegibilidade |
| BR-077 | Programa tipo 'A' (Assistencial) exige documentação completa (DOCUMENTOS-OK = 'S') | `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L146-L150` | `BENEFICIARIO.DOCUMENTOS-OK` | ALTO | Bloqueio por docs incompletos |

> Adicione mais linhas conforme necessário. Lembre-se: existem **10 regras escondidas** no código!

## Exemplo de linha bem preenchida

| ID     | Regra de Negócio                                                                        | Programa Fonte                                   | Campos DDM                                                               | Nível de Risco | Notas                                      |
| ------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ | -------------- | ------------------------------------------ |
| BR-013 | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L142-L148` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO        | Regra financeira. Tipo 'J' = exceção legal |

## Regras por Categoria

### Cálculos Financeiros

- **BR-071** — Renda familiar vs. teto do programa
- **BR-072** — Renda > R$ 600 sem dependentes (programa assistencial)

### Validações de Status

- **BR-065** — Status aceitos: A, S, C, I, D
- **BR-069** — Somente status 'A' permite elegibilidade

### Regras de Autorização

- **BR-067** — Prefixos especiais de CPF bypassing validação (governo/teste)
- **BR-068** — Região 99 bypass total de elegibilidade
- **BR-061** — CPF 000... com dígitos iguais é aceito (teste governo)

### Regras de Negócio Temporais

- **BR-062** — Ano de nascimento entre 1900 e ano atual
- **BR-070** — Faixa etária por programa (IDADE-MIN / IDADE-MAX)
- **BR-073** — Previdenciário exige idade ≥ 60
- **BR-074** — Trabalho exige idade entre 16 e 65

## Resumo Estatístico

- Total de regras encontradas: **18**
- Regras críticas: **4** (BR-060, BR-067, BR-071, BR-072)
- Regras com duplicação: **1** (validação CPF duplicada entre VALBENEF e VALDOCS)
- Regras sem documentação (escondidas): **3** (BR-061, BR-067, BR-068)

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

