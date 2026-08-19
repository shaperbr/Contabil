# CLAUDE.md

Contexto para sessões do Claude Code neste repositório.

## Estado atual

Projeto em **fase de definição**. Não há código de aplicação ainda — só
documentação. Nada de build, teste ou lint para rodar.

## Onde está o quê

- **`docs/PROJETO.md`** — documento de projeto. É a fonte da verdade sobre
  escopo, arquitetura e decisões. Leia antes de propor qualquer implementação.
- **`docs/rascunho-original.md`** — transcrição do rascunho manuscrito que
  originou o projeto. Referência histórica; não editar.

## Como trabalhar aqui

- **Decisões de produto ficam em `docs/PROJETO.md` §10**, no formato D1–D8, com
  três estados: *fechada*, *padrão* (vale enquanto ninguém decidir o contrário)
  e *aberta*. Ao implementar, siga a posição registrada. Ao mudar uma decisão,
  atualize o registro em §10 — não deixe a mudança só no código.
- **Não invente decisão fechada.** D7 (escritório piloto) e D8 (marca) estão
  abertas de propósito.
- Idioma da documentação e dos commits: **português**.

## Restrições que valem para qualquer código futuro

Vêm de §5 e §7 do documento de projeto:

- **Regra de negócio mora em `packages/core`.** Web e mobile consomem de lá.
  Regra duplicada faz as duas superfícies exibirem números diferentes para a
  mesma empresa — inaceitável num produto financeiro.
- **Valor monetário em inteiro de centavos**, nunca `float`.
- **Vencimento em `date`** (sem timezone); timestamp de evento em `timestamptz`.
- **Isolamento multi-tenant por RLS no Postgres**, não só filtro no ORM.
- **Documento fiscal nunca em bucket público** — storage privado com URL
  assinada de curta duração.
- **Nada é apagado de verdade** (`deleted_at`), por causa da retenção legal.
- **Nenhum valor financeiro é gravado só por extração automática.** Abaixo do
  limiar de confiança, revisão humana.
