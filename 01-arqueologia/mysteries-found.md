<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Mistérios Encontrados — SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **mysteries-found**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Registre aqui toda lógica, comportamento ou código que o time não conseguiu explicar.
> "Mistérios" são trechos de código sem documentação, com lógica não-óbvia ou que parecem workarounds.
>
> **Cota mínima para passar pelo portão do Estágio 2:** 5 mistérios documentados.

## O que conta como "mistério"?

- Código que faz algo inesperado sem comentário explicando por quê
- Valores hardcoded sem explicação (números mágicos)
- Lógica condicional que parece um workaround ou gambiarra
- Campos no DDM que não são usados por nenhum programa
- Programas que existem mas não são chamados por ninguém
- Comportamento diferente entre o que a documentação diz e o que o código faz
- Easter eggs deixados pelos desenvolvedores originais

## Níveis de Confiança

| Nível     | Significado                                         |
| --------- | --------------------------------------------------- |
| **ALTA**  | Temos certeza de que há algo estranho aqui          |
| **MÉDIA** | Parece suspeito, mas pode ter explicação            |
| **BAIXA** | Pode ser intencional, mas não conseguimos confirmar |

## Mistérios Catalogados

| ID      | Descrição | Onde Encontrado | Impacto Potencial | Confiança |
| ------- | --------- | --------------- | ----------------- | --------- |
| MYS-001 | CPF com todos dígitos iguais iniciando com 000 é aceito como válido (exceção "teste governo") | VALBENEF.NSN#L136-L142 | Backdoor que pode permitir CPFs fraudulentos no sistema moderno | ALTA |
| MYS-002 | 8 prefixos de CPF (000,001,002,010,011,099,100,999) fazem bypass TOTAL de toda validação de documentos | VALDOCS.NSN#L125-L135 | Risco de segurança: qualquer documento com esses prefixos é aceito sem verificação | ALTA |
| MYS-003 | Código de região 99 (internacional/diplomático) bypassa TODAS as regras de elegibilidade | VALELEG.NSN#L80-L84 | Beneficiários com região 99 entram em qualquer programa sem verificação | ALTA |
| MYS-004 | Valor R$ 600,00 hardcoded como teto de renda para programa assistencial sem constante nomeada | VALELEG.NSN#L139 | Número mágico — se o valor mudar, precisa alterar código; sem rastreabilidade | MÉDIA |
| MYS-005 | Validação de CPF está duplicada em VALBENEF e VALDOCS com lógica ligeiramente diferente (VALDOCS não verifica dígitos iguais) | VALDOCS.NSN#L83-L110 vs VALBENEF.NSN#L130-L157 | Inconsistência: um CPF pode ser válido em um módulo e inválido em outro | ALTA |
| MYS-006 | Programa tipo 'P' (Previdenciário) exige idade ≥ 60 hardcoded, ignorando o campo IDADE-MIN do cadastro do programa | VALELEG.NSN#L151-L155 | Regra duplicada/conflitante com a verificação genérica de faixa etária (L113-L123) | MÉDIA |
| MYS-007 | Código de elegibilidade usa codificação posicional (1ª posição='R' → NIS, 2ª posição='D' → dependentes) sem documentação | VALELEG.NSN#L175-L187 | Significado das posições 3-5 é desconhecido; pode haver regras não implementadas | MÉDIA |

## Detalhamento dos Mistérios

### MYS-001: CPF "000..." com dígitos iguais aceito como válido

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALBENEF.NSN#L136-L142`
- **Trecho de código**:

```natural
* EXCECAO: CPFs INICIADOS COM 000 SAO VALIDOS (TESTE GOVERNO)
    IF #DIG(1) = 0 AND #DIG(2) = 0 AND #DIG(3) = 0
      MOVE TRUE TO #CPF-VALIDO
      ESCAPE ROUTINE
    END-IF
```

- **O que esperávamos**: CPF com todos dígitos iguais (ex: 00000000000) deveria ser inválido
- **O que o código faz**: Aceita como válido se os 3 primeiros dígitos forem 0
- **Hipótese do time**: Backdoor para testes do governo que nunca foi removida
- **Risco se ignorarmos**: CPFs de teste podem entrar em produção no sistema moderno

---

### MYS-002: Prefixos especiais bypassam toda validação de documentos

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN#L125-L135`
- **Trecho de código**:

```natural
  FOR #I = 1 TO 8
    IF #PREF-CPF = #PREF-ESP(#I)
      MOVE TRUE TO #DOC-ESP-OK
      MOVE TRUE TO #CPF-OK
      MOVE 'V' TO #RESULTADO
      MOVE 0 TO #QTD-ERROS
      ESCAPE BOTTOM
    END-IF
  END-FOR
```

- **O que esperávamos**: Todo CPF deveria passar pela validação mod-11
- **O que o código faz**: Se os 3 primeiros dígitos são 000/001/002/010/011/099/100/999, zera todos os erros e aceita
- **Hipótese do time**: Documentos de órgãos governamentais ou registros de teste com prefixos reservados
- **Risco se ignorarmos**: Brecha de segurança no sistema moderno — atacante pode usar esses prefixos

---

### MYS-003: Região 99 garante elegibilidade automática sem verificações

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L80-L84`
- **Trecho de código**:

```natural
IF #COD-REG = 99
  MOVE TRUE TO #ELEGIVEL
  WRITE 'BENEFICIARIO ELEGIVEL - REGIAO ESPECIAL'
  ESCAPE ROUTINE
END-IF
```

- **O que esperávamos**: Todo beneficiário deveria passar por verificação de status, idade, renda
- **O que o código faz**: Região 99 pula TODAS as verificações — elegibilidade garantida
- **Hipótese do time**: Criado em 2013 (Anderson Lima) para beneficiários internacionais/diplomáticos
- **Risco se ignorarmos**: Qualquer beneficiário com região 99 pode entrar em qualquer programa

---

### MYS-004: R$ 600,00 hardcoded como número mágico

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN#L139`
- **Trecho de código**:

```natural
    IF #RENDA > 600.00
      IF #NUM-DEP < 1
```

- **O que esperávamos**: Limites de renda configuráveis no cadastro do programa (como RENDA-MAX)
- **O que o código faz**: Usa 600.00 hardcoded além do RENDA-MAX genérico
- **Hipótese do time**: Valor antigo de salário mínimo ou linha de pobreza que nunca foi parametrizado
- **Risco se ignorarmos**: Valor desatualizado; na migração deve virar parâmetro configurável

---

### MYS-005: Validação de CPF duplicada com comportamento diferente

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN#L83-L110` vs `VALBENEF.NSN#L130-L157`
- **Trecho de código**: (VALDOCS não verifica dígitos iguais; VALBENEF verifica mas com exceção 000)
- **O que esperávamos**: Uma única rotina de validação de CPF
- **O que o código faz**: Duas implementações com regras diferentes
- **Hipótese do time**: Código copiado entre programas sem refatoração; autores diferentes (Márcia vs Ana Lúcia)
- **Risco se ignorarmos**: Um CPF pode ser válido no cadastro mas inválido na validação de documentos (ou vice-versa)

## Easter Eggs

> Dica: existem **3 easter eggs** escondidos no código legado. Registre aqui os que encontrar:

1. [ ] Easter Egg 1: \_\_\_
2. [ ] Easter Egg 2: \_\_\_
3. [ ] Easter Egg 3: \_\_\_

## Resumo

- Total de mistérios encontrados: **7**
- Confiança alta: **5**
- Confiança média: **2**
- Confiança baixa: **0**
- Easter eggs encontrados: **0** / 3

---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="mysteries-checklist.md"><strong>mysteries-checklist.md</strong></a><br/>
<sub>Lista do que procurar.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="discovery-report.md"><strong>discovery-report.md</strong></a><br/>
<sub>Síntese final.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

