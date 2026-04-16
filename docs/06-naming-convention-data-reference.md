---
id: naming-convention
title: 6. Naming Convention & Data Reference
---

# 6. Naming Convention & Data Reference

## 6.1 Composition Name Format

All compositions must follow this exact 12-token underscore-delimited format. Any deviation causes a parsing error in the Index Compositions node.

`{category}_{campaign}_{instrument}_{version}_{regulation}_{rw}_{language}_{creativeType}_{creativeSize}_{creativeLength}_{source}_{acpm}`

Example:
`OTH_SM-instruments-charts-news-T_Tesla_v1_BAH_0_EN_Video_1080x1920_30sec_CT_0.007`

## 6.2 Regulatory Prefix Codes

| CRP | GOL | GAS | OIL | IND | FOX | PGE | COM | EQU | BND |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Corporate | Gold | Gas | Oil | Index | Fox | Page | Commodity | Equity | Bond |

Additionally: `OTH` (Other)

## 6.3 Language Codes (7th Segment)

| Code | Language | Code | Language | Code | Language | Code | Language |
| --- | --- | --- | --- | --- | --- | --- | --- |
| EN | English | DE | German | AR | Arabic | ES | Spanish |
| FR | French | IT | Italian | ZHT | Chinese Trad. | ZHS | Chinese Simp. |
| NL | Dutch | PL | Polish | PT | Portuguese | RO | Romanian |
| SV | Swedish | CZ | Czech | RU | Russian | GK | Greek |
| HU | Hungarian | | | | | | |

## 6.4 Regulation Codes (5th Segment)

| Code | Description |
| --- | --- |
| `BAH` | Bahrain regulation |
| `BAHnoCFD` | Bahrain — no CFD products |
| `CySec` | Cyprus Securities and Exchange Commission |
| `FCA` | Financial Conduct Authority (UK) |
| `ASIC` | Australian Securities and Investments Commission |
| `SCA` | Securities and Commodities Authority (UAE) |
| `SCAnoCFD` | SCA — no CFD products |
| `CySecnoCFD` | CySec — no CFD products |
| `ASICnoCFD` | ASIC — no CFD products |

:::info
When a comp name contains 'end', the suffix 'end' is appended to the regulation row name in the AE expression (e.g. `BAHend`).
:::
