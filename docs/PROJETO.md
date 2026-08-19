# Projeto Contábil — Documento de Projeto

> Documento vivo. Origem: rascunho manuscrito de 18/08/2026, transcrito em
> [`rascunho-original.md`](./rascunho-original.md). Tudo aqui é proposta —
> os pontos marcados com **[DECIDIR]** ainda dependem do dono do produto.

---

## 1. Visão

Uma plataforma — **web e mobile** — que serve de ponte entre o **escritório de
contabilidade** e a **empresa cliente**. O escritório publica guias e documentos; o app extrai os
dados do PDF, cria o vencimento, joga na agenda do cliente, avisa antes de
vencer, permite pagar ali mesmo, mostra quanto a empresa fatura e quanto paga
de imposto — e cobra o honorário do próprio escritório pelo app.

**Frase de posicionamento:** _"O cliente do escritório nunca mais perde um
vencimento nem pergunta 'quanto eu pago de imposto?'."_

### Problema

| Dor | Quem sente |
| --- | --- |
| Guia chega por e-mail/WhatsApp e se perde; cliente paga com multa e juros | Cliente e escritório (que leva a culpa) |
| Escritório gasta horas cobrando extrato, nota e documento todo mês | Escritório |
| Cliente não tem noção da própria carga tributária | Cliente |
| Honorário atrasa; cobrança é manual e constrangedora | Escritório |
| Documentos espalhados em e-mail, Drive e papel | Ambos |

### Não-objetivos (v1)

- Não é um ERP nem sistema de emissão de nota fiscal.
- Não substitui o sistema contábil do escritório (Domínio, Alterdata, Questor…) —
  convive com ele.
- Não faz escrituração nem apuração automática de imposto.

---

## 2. Personas e superfícies

### Papéis

| Papel | O que faz |
| --- | --- |
| **Escritório (admin)** | Cadastra empresas, gerencia usuários, define honorários, vê a carteira inteira |
| **Contador / analista** | Sobe guias e documentos em lote, acompanha pendências, cobra documento |
| **Cliente (empresário)** | Recebe guia, paga, envia extrato/documento, vê dashboard, paga honorário |
| **Convidado do cliente** (sócio, financeiro) | Acesso somente leitura ou pagamento, conforme permissão |

### Superfícies — **[DECIDIDO]**

O rascunho dizia _"app apenas p/ celular"_. **Decisão: web também**, para os
dois lados. O produto tem duas superfícies com paridade de funcionalidade:

| | Mobile (iOS/Android) | Web |
| --- | --- | --- |
| **Cliente** | Superfície principal: push de vencimento, foto de documento, pagamento, dashboard | Mesmas funções em tela grande; melhor para revisar dashboard e histórico |
| **Escritório** | Acompanhamento da carteira, aprovar extração, responder cliente | Superfície principal: upload em lote, revisão de extração, conciliação, cobrança |

Cada lado tem uma superfície *principal* — é onde o trabalho de verdade
acontece e onde a UX é otimizada — mas nenhuma função fica trancada em uma só.

**O que isso custa.** A maior parte do trabalho é mesmo compartilhada: API,
modelo de dados, regras de negócio, permissões, pipeline de extração,
validações — nada disso se duplica. O que **não** vem de graça:

- **Recursos nativos** sem equivalente no navegador: push confiável, biometria,
  scanner de documento com câmera, deep link para o app do banco. No web viram
  fallback (e-mail, upload de arquivo, copiar linha digitável).
- **Publicação nas lojas**: revisão da Apple/Google, versionamento e o fato de
  que usuário desatualizado continua existindo — a API precisa versionar.
- **Layout responsivo de verdade** nas telas densas (tabela de carteira,
  revisão de extração lado a lado com o PDF).

A estratégia de arquitetura (§5) é montada justamente para que a afirmação
"o trabalho é basicamente o mesmo" seja verdadeira: monorepo, domínio e
cliente de API compartilhados, e UI compartilhada onde compensa.

---

## 3. Módulos

Os quatro "grupos" do rascunho, expandidos e reorganizados:

### 3.1 Gestor de Arquivos

Repositório único de documentos por empresa.

- Organização por **empresa → competência (mês/ano) → tipo** (guia, balancete,
  DRE, nota fiscal, extrato, contrato social, procuração…).
- Upload nos dois sentidos: escritório → cliente e cliente → escritório.
- Versionamento (guia retificada não apaga a original) e trilha de auditoria
  (quem subiu, quem baixou, quando).
- Busca por texto extraído do PDF (OCR), não só por nome do arquivo.
- Retenção: documento fiscal precisa sobreviver ~5 anos — política de retenção
  e exportação em massa desde a v1.
- **Solicitação de documento**: o escritório pede "extrato de julho"; vira
  pendência com prazo e lembrete no app do cliente. Resolve o
  _"envio de documentação"_ do rascunho.

### 3.2 Gestor de Vencimentos

Coração do produto.

- Uma **obrigação** = tipo (DAS, DARF, INSS, FGTS, ISS, honorário…), competência,
  vencimento, valor, status, documento anexo, linha digitável / Pix.
- Origem: extraída de um PDF, criada manualmente, ou gerada por **regra
  recorrente** (ex.: DAS todo dia 20).
- Estados: `pendente` → `agendado` → `pago` → `conciliado`; mais `vencido`,
  `cancelado`, `contestado`.
- **Sync com agenda** (rascunho: _"vencimentos linkados com agendas"_):
  evento no Google Calendar do cliente, atualizado quando muda valor ou data
  e removido quando pago.
- Notificações escalonadas: D-7, D-3, D-1, no dia, e no atraso. Push como canal
  principal; e-mail e WhatsApp como fallback. **[DECIDIR]** canais.

### 3.3 Extração de PDF (o "Facilitador com I.A.")

Pipeline que transforma um PDF em dado estruturado:

```
upload → OCR (se necessário) → classificação do documento → extração de campos
       → validação (checksum da linha digitável, CNPJ, faixas de valor)
       → confiança alta? grava : manda para revisão humana
```

Campos-alvo por guia: CNPJ, competência, código de receita, vencimento,
principal, multa, juros, total, linha digitável / QR Pix.

**Regra inegociável:** nada com valor financeiro é pago com base apenas em
extração automática. Abaixo do limiar de confiança, um humano confirma. O
campo extraído sempre mostra a origem (página e trecho do PDF).

O mesmo pipeline lê **balancete / DRE** para alimentar os dashboards, e o
**extrato bancário** enviado pelo cliente.

> Sobre o "facilitador com I.A." do rascunho: ele tem **duas leituras
> possíveis** — (a) o extrator de documentos descrito acima, ou (b) um
> assistente conversacional ("quanto paguei de ISS esse ano?", "o que é essa
> guia?"). Este documento assume que (a) é a v1 (é o que gera valor
> mensurável) e (b) entra na Fase 4, já em cima de dados estruturados —
> assistente sobre PDF cru alucina valor de imposto. **[DECIDIR]**

### 3.4 Dashboards

Da página 2 do rascunho:

- **Evolução de receita** — série mensal, comparativo com o mesmo mês do ano anterior.
- **Evolução de impostos** — série mensal por tributo (empilhado).
- **Percentagem de impostos** — carga tributária = impostos / receita, mês a mês.

Extras de baixo custo e alto valor percebido: total pago no ano, próximo
vencimento em destaque, e quanto foi economizado em multa/juros desde a adoção
(essa é a métrica que vende renovação).

### 3.5 Pagamentos

Dois fluxos distintos, que não devem ser confundidos:

1. **Pagamento da guia** (rascunho: _"interface p/ o pagamento da guia"_) —
   o dinheiro é do cliente e vai para a União/estado/município.
   Caminho v1 realista: **copiar linha digitável / QR Pix + deep link para o app
   do banco**, com o cliente marcando como pago (ou conciliação via Open
   Finance). Pagamento dentro do app exige PSP licenciado e traz
   responsabilidade sobre valor errado — decisão consciente, não detalhe técnico.
2. **Cobrança de honorário** (rascunho: _"cobrança de honorário direto no app"_) —
   o dinheiro é do escritório. Assinatura recorrente por CNPJ, via Pix
   recorrente/automático, boleto ou cartão. Split entre plataforma e escritório
   se o modelo de receita for marketplace. **[DECIDIR]**

### 3.6 Integrações com o fisco (o "API com a Receita")

Não existe API pública geral da Receita Federal para consulta de débitos e
guias de um contribuinte. Os caminhos reais:

| Caminho | O que dá | Custo/risco |
| --- | --- | --- |
| **Procuração eletrônica e-CAC** (certificado A1 do escritório) | Caixa postal (DTE), situação fiscal, DCTFWeb, parcelamentos | Automação é frágil e sujeita a mudança de layout; precisa de consentimento formal do cliente |
| **Integrações prontas de mercado** (parceiros de consulta fiscal) | Situação cadastral, CND, DTE | Custo por consulta; menos manutenção |
| **PGDAS-D / Simples Nacional** | Apuração e DAS | Só com procuração; layout instável |
| **APIs públicas de cadastro** (CNPJ, CNDs) | Dados cadastrais e certidões | Simples, sem procuração; valor limitado |

**Recomendação:** a v1 **não depende** disso. O escritório já tem as guias no
sistema contábil dele — o gargalo é a *entrega ao cliente*, não a obtenção
junto ao fisco. Integração fiscal entra na Fase 5, começando pelo **DTE
(caixa postal)**, que é o de maior valor e menor esforço.

### 3.7 Open Finance

Objetivo: puxar o **extrato** automaticamente em vez de pedir ao cliente
(rascunho: _"envio de extrato"_), e conciliar pagamento de guia com débito na
conta.

Ser instituição participante do Open Finance não é viável para um produto
novo. O caminho é **agregador** (Pluggy, Belvo, Klavi e afins): consentimento
do cliente, leitura de contas e transações, renovação periódica do consent.
Sempre **somente leitura** — iniciação de pagamento (Pix por iniciação) é
outro nível regulatório.

Enquanto isso, na v1: upload de OFX/PDF do extrato, lido pelo mesmo pipeline
de extração.

---

## 4. Modelo de dados (esboço)

Multi-tenant em dois níveis: **escritório** (tenant) → **empresas** (clientes).

```
organizations        escritório contábil
users                pessoas (podem pertencer a vários contextos)
memberships          user × (organization | company) × papel
companies            empresa cliente (CNPJ, regime tributário, IE, IM)

documents            metadados (empresa, tipo, competência, origem, autor)
document_versions    arquivo em si (chave no storage, hash, tamanho)
document_requests    "mande o extrato de julho" (prazo, status)
extractions          resultado do pipeline por versão (JSON + confiança + revisor)

obligations          vencimento (tipo, competência, due_date, valor, status)
obligation_rules     recorrência que gera obrigações
obligation_documents obrigação × documento (a guia)
payments             comprovante/registro de quitação de uma obrigação

bank_connections     consentimento Open Finance por empresa
bank_accounts        conta
bank_transactions    lançamento (usado para conciliar)

fee_contracts        honorário: valor, reajuste, dia de cobrança
invoices             fatura de honorário
subscriptions        assinatura no PSP

calendar_links       obrigação × evento externo (Google Calendar)
notifications        o que foi enviado, para quem, por qual canal
audit_log            quem fez o quê (obrigatório: dado fiscal)
```

Regras estruturais que evitam retrabalho depois:

- Todo registro de negócio carrega `company_id`, isolado por **RLS** no Postgres.
- Valores monetários em **inteiro de centavos**, nunca `float`.
- Datas de vencimento em `date` (sem timezone); timestamps de evento em `timestamptz`.
- Nada é apagado de verdade — `deleted_at`, por causa da retenção legal.

---

## 5. Arquitetura proposta

```
┌──────────────────┐   ┌──────────────────┐
│   App mobile     │   │       Web        │
│  React Native    │   │     Next.js      │
│     (Expo)       │   │ cliente + escrit.│
└────────┬─────────┘   └────────┬─────────┘
         │                      │
         │   packages/core  ────┤  domínio, tipos, validações,
         │   packages/api-client│  cliente de API — compartilhados
         │                      │
         └──────────┬───────────┘
                    │  HTTPS / REST (versionada)
                    ▼
           ┌─────────────────┐      ┌──────────────┐
           │   API (Node/TS) │─────▶│  PostgreSQL  │
           │  Next API/Nest  │      │  (RLS + pgvector opcional)
           └────┬───────┬────┘      └──────────────┘
                │       │
                ▼       ▼
       ┌────────────┐  ┌──────────────┐
       │  Storage   │  │ Fila (Redis) │
       │ S3 / MinIO │  │   BullMQ     │
       └────────────┘  └──────┬───────┘
                              ▼
                     ┌──────────────────┐
                     │ Workers          │
                     │ OCR · extração   │
                     │ IA · notificação │
                     │ sync agenda      │
                     └──────────────────┘
```

### Monorepo

```
apps/
  api           API + workers (Node/TS, Prisma)
  web           Next.js — cliente e escritório
  mobile        Expo / React Native
packages/
  core          domínio: tipos, estados de obrigação, regras de vencimento,
                validação de linha digitável, cálculo de carga tributária
  api-client    cliente HTTP tipado, gerado do contrato da API
  ui            componentes compartilhados (React Native Web) — opcional
```

**Escolhas e porquês:**

- **Next.js + Prisma + Postgres**: alinhado com a stack já usada na
  infraestrutura existente (Docker Swarm + Traefik + Portainer), o que encurta
  o caminho até produção.
- **React Native (Expo)** para o app: um código para iOS e Android, push nativo,
  biometria, câmera e deep link para app de banco.
- **`packages/core` é onde mora a regra de negócio.** Se "vencimento em fim de
  semana antecipa" ou "carga tributária = impostos/receita" for reimplementado
  em cada superfície, web e mobile vão divergir e mostrar números diferentes
  para a mesma empresa — o pior tipo de bug num produto financeiro. Regra de
  negócio duplicada é o único jeito de o custo de duas superfícies sair caro.
- **Contrato de API único e versionado**, consumido pelas duas superfícies via
  `api-client` gerado. Nenhuma superfície tem endpoint privativo.
- **UI compartilhada onde compensa** (React Native Web ou Expo Web): vale para
  formulário, card de vencimento, lista de documento. Não vale para as telas
  densas do escritório (tabela de carteira, revisão de extração ao lado do PDF)
  nem para telas nativas de câmera — essas são nativas de cada superfície,
  de propósito.
- **Fila obrigatória**: OCR e chamada de modelo são lentos e falham; nunca no
  request HTTP.
- **Storage privado** com URL assinada de curta duração. Nenhum documento fiscal
  em bucket público, em nenhuma hipótese.
- **RLS no banco**, não só filtro no ORM: com dados de várias empresas na mesma
  tabela, um `where` esquecido vaza dado fiscal de terceiro.

---

## 6. Roadmap

| Fase | Entrega | Critério de pronto |
| --- | --- | --- |
| **0 — Descoberta** | 1 escritório piloto, 5 empresas reais, mapa do fluxo atual e amostra de 50 guias | Amostra de PDFs em mãos e fluxo desenhado |
| **1 — MVP** | Web (escritório e cliente) + app mobile do cliente: arquivos, vencimentos manuais, agenda, push, solicitação de documento | Piloto roda um mês inteiro sem planilha paralela |
| **2 — Automação** | Extração de PDF, criação automática de obrigação, dashboards (receita, impostos, %) | ≥90% das guias extraídas sem revisão humana |
| **3 — Dinheiro** | Linha digitável/Pix + conciliação, honorário recorrente | Primeiro honorário cobrado pelo app |
| **4 — Dados** | Open Finance (leitura) e assistente de IA sobre dados estruturados | Extrato chega sozinho; assistente responde sem alucinar valor |
| **5 — Fisco** | DTE/caixa postal, situação fiscal, certidões | Alerta de intimação chega antes do cliente descobrir |

A ordem é deliberada: **valor antes de integração**. As fases 1–2 já resolvem
a dor principal e não dependem de terceiro nenhum.

**Sobre as duas superfícies no roadmap:** a partir da Fase 2, cada entrega sai
nas duas — a regra vive em `packages/core` e as duas superfícies a consomem.
Na Fase 1 há uma exceção prática: o app mobile depende de revisão nas lojas,
então o web sai primeiro e o piloto começa por ele enquanto a primeira build
mobile é submetida. Não é escopo cortado, é ordem de publicação.

---

## 7. Riscos e obrigações

| Risco | Mitigação |
| --- | --- |
| **LGPD** — dado fiscal e bancário de PJ e de sócios PF | Base legal por contrato, minimização, criptografia em repouso, retenção declarada, DPA com o escritório |
| **Valor extraído errado gerar pagamento errado** | Confiança mínima + revisão humana + comprovante sempre exibindo o PDF original; nunca pagar sem confirmação |
| **Guarda de documento fiscal (~5 anos)** | Retenção e exportação em massa desde a v1; backup testado |
| **Pagamento dentro do app** | Não custodiar dinheiro: PSP licenciado ou deep link para o banco |
| **Open Finance** | Só via agregador; consentimento explícito, renovável, revogável pelo cliente |
| **Automação do e-CAC** | Procuração eletrônica formal; assumir que quebra a cada mudança de layout |
| **Vazamento entre empresas (multi-tenant)** | RLS no banco + teste automatizado de isolamento por tenant |
| **Notificação que não chega** | Push tem entrega não garantida; e-mail como espelho de todo aviso crítico |

---

## 8. Métricas de sucesso

- **% de obrigações pagas em dia** (a métrica-mãe; comparar com o pré-adoção)
- **R$ economizados em multa e juros** por empresa — argumento de renovação
- **Tempo médio entre solicitação e entrega de documento** pelo cliente
- **% de guias extraídas sem revisão humana**
- **Inadimplência de honorário** antes × depois
- Retenção do escritório em 6 meses e nº de CNPJs ativos por escritório

---

## 9. Modelo de receita — **[DECIDIR]**

Hipótese principal: **SaaS B2B2C** — quem paga é o escritório, por CNPJ ativo
por mês; o cliente final usa de graça. O escritório vende o app como
diferencial e o custo se paga com a hora que ele deixa de gastar cobrando
documento.

Alternativas: taxa sobre honorário cobrado via app (alinha incentivo, mas
depende de volume), ou plano freemium por empresa (fricção alta em PME).

---

## 10. Decisões

### Fechadas

- **Superfícies** — web **e** mobile, para cliente e escritório, com paridade de
  funcionalidade e superfície principal distinta para cada lado (§2). Arquitetura
  em monorepo com domínio compartilhado para sustentar isso (§5).

### Em aberto

1. **"Facilitador com I.A."** — extrator, assistente conversacional, ou os dois? (§3.3)
2. **Pagamento da guia** — deep link para o banco (v1) ou pagamento in-app com PSP?
3. **Regimes tributários atendidos** — só Simples Nacional na v1, ou também Lucro Presumido/Real? Isso define quais guias o extrator precisa entender.
4. **Canais de notificação** — push + e-mail bastam, ou WhatsApp é obrigatório? (No Brasil, provavelmente é.)
5. **Escritório piloto** — quem é? Sem um parceiro real, as fases 0–1 viram suposição.
6. **Marca e nome do produto.**

---

## 11. Próximos passos sugeridos

1. Fechar as decisões em aberto 1 e 3 (§10) — são as que ainda travam a modelagem.
2. Conseguir o escritório piloto e uma amostra de 50 guias reais (DAS, DARF,
   FGTS, ISS) para calibrar o extrator.
3. Prototipar as telas que definem o produto, nas duas superfícies:
   **próximo vencimento**, **detalhe da guia** e **dashboard de carga
   tributária** (cliente); **upload em lote** e **revisão de extração**
   (escritório, web).
4. Provar tecnicamente o pedaço mais arriscado: extrair corretamente os campos
   de um DAS e de um DARF reais, com validação de linha digitável.
