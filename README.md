# dashboard-c2s — C2S Hub

Painel operacional interno da **ImovelUm**: gestão de leads, BI e comissões.

🌐 **Produção:** https://hub.conexaoum.com.br

---

## Visão geral

Sistema de **arquivo único HTML** (`c2s-hub.html`, ~2300 linhas) servido via **GitHub Pages**. Consome dois back-ends:

- **`c2s-proxy`** (Node/Express no Render) — ponte com a API Contact2Sale
- **Firebase Realtime Database** — armazena status, observações, comissões, cache de leads

---

## Estrutura do repo

```
.
├── c2s-hub.html        ← aplicação principal (login + 3 abas)
├── importar-base.html  ← utilitário de import de XLSX (admin)
├── index.html          ← redirect 0s pra c2s-hub.html
├── CNAME               ← hub.conexaoum.com.br
└── README.md
```

---

## Abas do app

### 1. Operacional (todos os usuários)
- Tabela de leads com **20 status de atendimento** (Novo, Conversando, Visita Agendada, Negócio Fechado, etc)
- Filtros: data, equipe, corretor, status, cliente, bairro, imóvel, faixa de valor, natureza (compra/locação)
- Edição inline: status, observação (modal), data de visita, bairro/cidade manual, valor manual
- Botão 💰 abre modal de comissão (para Daiana/Milena/Thiago)
- Botão "↗ Reatribuir" (só gestores) abre C2S no navegador
- Paginação, contadores de status, save automático no Firebase

### 2. Inteligência · BI (só Thiago)
- KPIs (total, em negociação, novos, compra vs locação, ticket médio)
- Distribuição: canal de entrada, fonte, leads por corretor, leads por equipe
- Status × atendimento, faixa de preço × portal, visitas × dia da semana, etc
- Charts via **Chart.js 4.4.1** (CDN)
- Cruzamentos: fonte × tipo × filial, corretor × tipo
- Evolução temporal com média/pico/dias ativos

### 3. Comissões (Thiago + Daiana + Milena)
- **Caixa único** Daiana + Milena
- Registro de visitas/vendas/locações com valores tabelados
- Meta: 62 visitas de compra + 4 vendas no mês → majoração de valores
- Fluxo: secretária registra `pendente` → Thiago aprova → `aprovado` com valor calculado
- Thiago pode reprovar `aprovado` com motivo → notificação banner pro usuário

---

## Login e usuários

Login via campo nome + senha. **Autenticação manual**: nome é a chave no nó `/usuarios/<nome>` no Firebase, senha é comparada com `data.senha`.

> ⚠️ **Senhas em texto plano** no Firebase. Fase 2 vai migrar pra **Firebase Auth** real.

### Estrutura do nó `/usuarios/<nome>`
```json
{
  "senha": "...",
  "perfil": "gestor" | "corretor",
  "filtro_tipo": "empresa" | "equipe" | "individual",
  "bi": true | false
}
```

### Permissões por usuário
- `bi: true` → aba BI visível
- Daiana, Milena, ou `bi: true` → aba Comissões visível
- Thiago → ainda + botões Backup, Acessos, Reprovar comissão
- `filtro_tipo` define quais leads o usuário vê (toda a ImovelUm, equipe específica, ou só os próprios)

---

## Persistência (Firebase RTDB)

| Nó | Conteúdo | Acesso |
|---|---|---|
| `/usuarios/<nome>` | dados de login (senha em texto) | 🔒 autenticado (gravação) |
| `/registros/<lead_id>` | status, obs, hora, dataVisita, bairroM/cidadeM/valorM, histórico, aguardandoSync | 🌐 público |
| `/leads/<id>` | cache de leads sincronizados via proxy | 🌐 público |
| `/comissoes/YYYY/MM/<timestamp>` | registros de visitas/vendas/locações | 🔒 autenticado |
| `/notificacoes/<usuario_key>/<timestamp>` | notificações de reprovação | 🔒 autenticado |
| `/acessos` | log de logins/logouts | 🌐 público |
| `/sync_log` | logs do proxy (sync de leads e backup) | 🌐 público |
| `/backups/YYYY-MM-DD.json` | snapshots diários (via proxy) | 🔒 autenticado |

---

## Configurações no código

No topo do `<script>` do `c2s-hub.html`:

```javascript
var PROXY    = 'https://c2s-proxy.onrender.com/api';
var FIREBASE = 'https://c2s-operacional-default-rtdb.firebaseio.com';
```

Pra rodar contra um proxy local (porta 3000):
```javascript
var PROXY = 'http://localhost:3000/api';
```

---

## Como rodar local

### Mínimo (testar UI)
```powershell
Start-Process C:\caminho\para\c2s-hub.html
```
Funciona como `file://` — Firebase tem CORS aberto. Útil pra dev de UI sem rede.

### Recomendado (com servidor local)
```powershell
# Em qualquer diretório:
npx http-server C:\Users\user\Projetos\dashboard-c2s -p 4173
# Acessa: http://localhost:4173/c2s-hub.html
```

### Servidor de preview customizado
Já existe um em `C:\Users\user\.claude\c2s-preview-server.js` (Node nativo, sem deps).

---

## Deploy (GitHub Pages)

- **Branch:** `main`
- **Source:** raiz do repo
- **Custom domain:** `hub.conexaoum.com.br` (CNAME apontando pra `thiagodutra44.github.io`)
- **HTTPS:** ativado (cert do Let's Encrypt via GitHub)

Cada `git push origin main` dispara rebuild automático (~30s a 2min).

### Pra fazer deploy
```bash
git checkout main
git pull origin main
# (faz mudanças, commits)
git push origin main
# Aguarda ~30s a 2min pro Pages atualizar
```

Forçar refresh no navegador (`Ctrl+Shift+R`) pra pegar versão nova ignorando cache.

---

## Stack técnica

- **HTML / CSS / Vanilla JavaScript** — sem build, sem framework
- **Chart.js 4.4.1** via CDN
- **Google Fonts** (Rajdhani + Share Tech Mono) via CDN
- **XLSX.js** em `importar-base.html` (só pra import de Excel)

Visual: tema "tech/neon" — fundo `#04080f`, acentos cyan `#00d4ff` + orange `#ff7b2c`.

---

## Issues conhecidos & roadmap

### Resolvidos na Fase 1 (já em produção)
- ✅ Função `limparDados` removida (era o gatilho do incidente de 22/05/2026)
- ✅ XSS bloqueado em observações e nomes de corretores
- ✅ Toast visível em erros de salvamento (não engole mais silencioso)
- ✅ Senha não fica mais em `localStorage`
- ✅ Guardas em `comAprovar`/`comCancelar`/`reprovarConfirmar` (paliativas)
- ✅ Backup automático diário (via proxy)

### Pendente (Fase 1)
- ⏳ F1.6/F1.7 — Token C2S em env var + auth no proxy
- ⏳ F1.9 — Regras Firebase defensivas

### Roadmap maior
- **Fase 2** — Sync C2S↔Hub em tempo (quase) real. Hub vira espelho do C2S.
- **Fase 2** — Firebase Auth real (acabar com senha em texto plano)
- **Fase 2** — Cloud Functions pra lógica de comissões server-side
- **Fase 4** — Reengenharia: quebrar arquivo único em módulos, índices Firebase, testes

---

## Histórico de incidentes

- **22/05/2026** — Milena clicou em "Limpar Registros" e apagou 6.736 registros. Restaurado via backup manual. Botão removido da UI no mesmo dia, função removida do código na F1.A. Backup automatizado entrou na F1.D.
- **22/05/2026** — Antonio Paulo não aparecia no Hub. Milena não estava na lista `ATIVOS` do proxy. Adicionada + re-sincronização de 01/05 a 22/05 (1000 leads).

---

## Repos relacionados

- **`ThiagoDutra44/c2s-proxy`** (privado) — Node/Express no Render. Ver README lá pra detalhes do back-end.
