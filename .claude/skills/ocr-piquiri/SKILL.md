---
name: ocr-piquiri
description: Conhecimento da etiqueta Piquiri e do pipeline OCR do app de bobinas. Use sempre que for mexer na leitura de etiquetas, no parser, na releitura cirúrgica ou quando o usuário relatar campo lido errado ou em branco no OCR.
---

# OCR da etiqueta Piquiri — como trabalhar

## O padrão da etiqueta (layout fixo)

Todas as etiquetas Piquiri seguem o mesmo modelo:

- Faixa verde: CLIENTE/PRIME, `PESO LÍQ.: 720Kg` e GRAMATURA (texto branco sobre verde — o campo que mais some no OCR). Existe também PESO BRU na tabela de baixo; o padrão exige `PESO L` para não confundir.
- Tabela de baixo: **4 colunas fixas, cada uma com 2 pares rótulo/valor empilhados**:
  coluna 1 = NOTA FISCAL (valor) / MEDIDAS (valor); coluna 2 = PEDIDO / PALLET NÚMERO;
  coluna 3 = DATA FABRIC. / BOBINA/PALLET; coluna 4 = PESO BRU. / LOTE.
  Fisicamente NOTA FISCAL fica na MESMA COLUNA que MEDIDAS, uma linha acima do valor de MEDIDAS.
- O Tesseract lê essa tabela por **banda horizontal** (todas as colunas daquela altura viram uma linha de texto), não coluna por coluna: linha 1 = os 4 rótulos da 1ª leira (NOTA FISCAL/PEDIDO/DATA FABRIC/PESO BRU), linha 2 = os 4 valores correspondentes, linha 3 = os 4 rótulos da 2ª leira (MEDIDAS/PALLET NÚMERO/...), linha 4 = os 4 valores.
- `MEDIDAS: 750MM X 1100MM` → primeiro número = **formato**, segundo = **diâmetro**. O diâmetro tem sido sempre 1100.
- **"NOTA FISCAL" é o rótulo que mais sai mutilado** do OCR (às vezes vira só "NoT", perdendo "FISCAL" inteiro) — bem mais que "MEDIDAS", que costuma sobreviver mesmo truncado ("MEDID"). Por isso o código usa MEDIDAS como âncora de fallback para achar a NF (ver `regiaoAcimaDe`).

## Regras de negócio

- Piquiri: tipo é sempre "Capa"; não tem código de fornecedor — o nº da NF é usado como Cód. Fornecedor.
- "00000" (cinco zeros) é a convenção do usuário para "dado ausente na etiqueta".
- **Nunca quebrar a extração parcial**: preencher o que der, deixar em branco o resto para o usuário conferir no modal.

## Faixas plausíveis (validação de leitura)

| Campo | Faixa aceita | Observação |
|---|---|---|
| Nota Fiscal | 4–9 dígitos | só dígitos |
| Formato | 200–3000 mm | ex.: 750, 1790 |
| Diâmetro | 500–2000 mm | sempre 1100 até hoje |
| Gramatura | 40–450 g/m² | ex.: 120, 130 |
| Peso líq. | 50–5000 kg | ex.: 720, 815.5 |

Valor fora da faixa = leitura errada → descartar e reler.

## Confusões típicas do Tesseract nesta etiqueta

- `O↔0`, `I/l↔1`, `S↔5`, `G↔6`, `B↔8`, `Z↔2`
- `120gr` → `1209gr`/`1207` (o "gr" vira dígito grudado)
- `PESO LÍQ.:` → `PESO Lia:`; `Kg` → `K5`/`K9`
- Gramatura branca na faixa verde some ou gruda na linha do PESO LÍQ

## Arquitetura do pipeline (index.html)

1. **Passada 1**: foto redimensionada a 2600px, escala de cinza SEM reforço de contraste (contraste estoura o texto branco da faixa verde), PSM 4, `recognize(canvas, {}, {text, blocks})`.
2. **Parser** `parseEtiquetaPiquiri`: heurísticas sobre o texto corrido.
3. **Releitura cirúrgica** `relerCamposDuvidosos`: campo em branco ou fora da faixa → recorta a região ao lado/abaixo do rótulo (bboxes das palavras do Tesseract), amplia 3x, relê com PSM 7, extrai por candidatos (cru primeiro, depois `letrasParaDigitos`) e aceita o primeiro na faixa plausível.
   - **Nota Fiscal tem 3 tentativas em cascata**, do mais barato ao mais caro: (a) `notaFiscalPorPosicao` — acha um número solto de 4-9 dígitos no texto já lido, antes do bloco MEDIDAS, sem gastar nova chamada de imagem; (b) se o rótulo "FISCAL" foi reconhecido, recorta abaixo dele (`regiaoAbaixo`); (c) senão, recorta acima do rótulo "MEDID" (`regiaoAcimaDe`), que é a âncora mais confiável.
4. O modal mostra o texto bruto + as releituras em "Ver texto lido na foto (debug)" — o usuário sabe copiar isso.

## Como calibrar / testar

- Pedir ao usuário o texto do debug do modal com fotos reais — é a fonte da verdade.
- Testar a lógica em Node extraindo o script do index.html e mockando worker/canvas (ver sessões anteriores: `teste-parser*.js` no scratchpad — recriar quando precisar).
- Fluxo de foto que melhor funciona: modo "Documentos" da câmera do POCO + botão "Ler foto da galeria".
- Deploy: bump de `APP_VERSION` (index.html) e `CACHE` (sw.js) **juntos**, depois push (GitHub Pages).
