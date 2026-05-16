---
title: "Breve storia: Dalla disperazione con Claude Code al mio workflow di sviluppo con OpenCode e Ollama Cloud"
date: 2026-05-16
description: "Come ho costruito un ambiente di coding assistito da AI completamente autonomo, senza limiti di utilizzo e senza dipendenza da un singolo provider."
tags: ["AI", "workflow", "OpenCode", "Ollama", "sviluppo"]
---

## La mia disperazione: i limiti di Claude Code

Per diverso tempo ho usato Claude Code come strumento principale per il coding assistito nei miei progetti personali. Funziona bene — legge il codebase, esegue comandi, fa diff, apre sessioni contestuali. Il problema è arrivato con le progressive restrizioni ai limiti di utilizzo introdotte da Anthropic: finestre di contesto compresse, rate limit più stringenti, sessioni che si esauriscono nel mezzo di un task complesso.

Lavorare in questo modo è diventato estenuante. I miei progetti personali — un agente multi-modello per l'analisi di integrazioni, una knowledge base con pipeline LLM, un sistema di compliance AI-driven — sono codebase distribuiti su decine di file con sessioni lunghe e molta navigazione del contesto. Un rate limit nel mezzo di un `plan → build` non è un inconveniente, è un blocco concreto.

Ho iniziato a cercare un'alternativa.

---

## La soluzione in due parti

### 1. OpenCode — l'agente di coding

[OpenCode](https://opencode.ai) è un agente di coding agentico da terminale (con estensione anche per VS Code), open source, compatibile con qualsiasi provider LLM che espone un'API OpenAI-compatible. Supporta tool calling, MCP server, navigazione del codebase, esecuzione di comandi bash e lettura di diff git — tutto quello che serve per un workflow serio.

La configurazione avviene tramite un file `opencode.jsonc` nella root del progetto o nella directory globale. Su macOS il path è `~/.config/opencode/opencode.json`, su Windows è `%APPDATA%\opencode\opencode.json`.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama Cloud",
      "options": {
        "baseURL": "https://ollama.com/v1"
      },
      "models": {
        "qwen3-coder:480b-cloud": {
          "name": "Qwen3 Coder 480B (cloud)",
          "tools": true
        },
        "deepseek-v3.1:671b-cloud": {
          "name": "DeepSeek V3.1 671B (cloud)",
          "tools": true
        }
      }
    }
  }
}
```

Il campo `"tools": true` è indispensabile — senza di esso il modello non può usare i tool di OpenCode e l'esperienza agentiva non funziona.

### 2. Ollama Cloud — l'LLM senza limiti di token

[Ollama Cloud](https://ollama.com/cloud) è la feature di inference remota di Ollama, lanciata a settembre 2025. Funziona esattamente come Ollama locale, ma invece di eseguire il modello sulla tua macchina lo esegue su GPU nei datacenter di Ollama (US, Europa, Asia-Pacific).

Il vantaggio principale per il mio caso d'uso: **billing basato sul tempo di GPU, non sui token**. Non ci sono rate limit per token come con Anthropic o OpenAI. Ci sono limiti di sessione che si resettano ogni 5 ore e limiti settimanali, ma per sessioni di sviluppo normali sono abbondantemente sufficienti.

I modelli cloud si distinguono da quelli locali con il suffisso `:cloud`:

```bash
# Modello locale (gira sulla tua macchina)
ollama run qwen3:8b

# Stesso ecosistema, inference remota su GPU cloud
ollama run qwen3-coder:480b-cloud
```

Per autenticarsi basta fare `ollama signin` oppure esportare l'API key:

```bash
export OLLAMA_API_KEY=your_key_here
```

Il catalogo include modelli di dimensioni impraticabili in locale: `qwen3-coder:480b-cloud`, `deepseek-v3.1:671b-cloud`, `gpt-oss:120b-cloud` — modelli che su hardware consumer non girerebbero mai. La stessa interfaccia Ollama che già conosci, zero setup aggiuntivo.

---

## L'architettura del progetto: i file di contesto

Il vero differenziatore del mio workflow non è il tool, ma la struttura del contesto che fornisco all'agente. Ogni progetto ha una serie di file markdown che l'LLM legge all'inizio di ogni sessione per ricostruire la continuità del lavoro.

### `AGENTS.md` — il contratto con l'agente

È il file più importante. Descrive esattamente cosa l'agente deve fare, come deve comportarsi, le convenzioni del progetto e le regole di lavoro. Viene letto all'inizio di ogni sessione e funge da system prompt persistente scritto da me, non generato automaticamente.

#### Opzione rapida: `/init` per i progetti già esistenti

Se stai portando OpenCode su un progetto già avviato e non vuoi scrivere `AGENTS.md` da zero, OpenCode offre il comando `/init` che automatizza questo passaggio. Basta aprire OpenCode nella cartella del progetto e digitare:

```
/init
```

OpenCode analizza i file più rilevanti del repository, fa eventuali domande mirate se il codebase non risponde da solo, e genera un `AGENTS.md` calibrato sul progetto reale. Il file include automaticamente i comandi di build, lint e test rilevati, la struttura del repo e le convenzioni non ovvie dai soli nomi dei file.

Vale la pena committare il file generato su Git — così tutte le sessioni future (e chiunque altro lavori al progetto) parte dallo stesso contesto. Se la struttura del progetto cambia in modo significativo, `/init` può essere rieseguito per aggiornare il file.

#### Esempio scritto manualmente

Per i nuovi progetti parto sempre da un `AGENTS.md` scritto a mano, perché mi permette di essere esplicito su cose che `/init` non può inferire — come le temperature per ruolo o le regole di output. Esempio reale dal progetto `integration-analyst-agent`:

```markdown
# AGENTS.md — integration-analyst-agent

## Ruolo
Sei un agente di sviluppo per il progetto integration-analyst-agent.
Il tuo compito è implementare e mantenere una pipeline multi-agente
che analizza documentazione tecnica e produce report strutturati.

## Stack
- Python 3.11, Ollama locale
- python-telegram-bot per la ricezione richieste
- Pipeline tri-agente: scraper → validator → extractor
- reportlab per PDF, python-docx per Word

## Architettura modelli
Ogni ruolo ha un modello assegnato e una temperatura fissa:
- classifier: qwen2.5:3b — temperature 0.0
- scraper: mistral-nemo — temperature 0.2
- analyst: deepseek-r1:8b — temperature 0.0
- writer: mistral-nemo — temperature 0.0

## Regole
- Le temperature non si modificano senza istruzione esplicita
- Ogni chiamata ollama.chat() deve includere options={"temperature": T}
- Non usare print() in produzione, usa il modulo logging
- Output del report sempre in italiano

## Prima di iniziare
Leggi TODO.md per i task aperti e checkpoint.md per
lo stato dell'ultima sessione.
```

### `TODO.md` — i task aperti

Lista dei task con priorità, organizzati per area. L'agente lo aggiorna durante la sessione man mano che completa i task; io lo revisionno a fine sessione.

```markdown
# TODO

## Alta priorità
- [ ] Fix: il validator_agent accetta contenuto parziale da siti
      con Cloudflare WAF — aggiungere strategia javascript_render
- [ ] Fix: il report finale a volte mescola italiano e inglese
      nelle sezioni generate dall'analyst

## In corso
- [x] Aggiunto ThreadedHTTPServer per gestire richieste concorrenti

## Backlog
- [ ] Cache locale con TTL 24h per evitare scraping ripetuto
      dello stesso URL nella stessa giornata
- [ ] Modalità "analisi batch" via Telegram (lista di URL in un messaggio)
```

### `logs.md` — il diario tecnico della sessione

Ogni sessione di lavoro viene documentata qui: cosa è stato fatto, quali decisioni sono state prese e perché. È il formato più utile per ricostruire il contesto nelle sessioni successive senza dover rileggere il codice.

```markdown
# Logs

## 2026-05-10

### Sessione: fix language mixing nel report
- **Problema**: il writer_agent produceva sezioni in inglese anche
  con system prompt in italiano
- **Causa**: il modello ignorava la lingua se il contenuto estratto
  dall'extractor era in inglese
- **Soluzione**: aggiunto vincolo esplicito nel prompt del writer:
  "Rispondi sempre e solo in italiano, indipendentemente dalla
  lingua del contenuto in input"
- **File modificati**: `agent/writer.py`
```

### `checkpoint.md` — il punto di ripresa

Questo è il file che salva la sessione prima che il contesto si esaurisca. Quando il contesto dell'agente si avvicina al limite, chiedo esplicitamente di aggiornare il checkpoint prima di chiudere. La sessione successiva riparte da qui.

```markdown
# Checkpoint — 2026-05-10

## Stato attuale
Il fix language mixing è applicato e testato. Il report ora
esce tutto in italiano anche con sorgenti in inglese.

## Prossimi passi
1. Testare con documentazione Stripe (sito con Cloudflare attivo)
2. Implementare la strategia javascript_render come fallback
   nel scraper_agent quando httpx restituisce contenuto vuoto

## Problema aperto
- I siti con Cloudflare WAF restituiscono 403 con httpx puro
  → javascript_render con playwright è la strada giusta
  → playwright deve girare in modalità headless su Mac Mini

## File rilevanti modificati
- `agent/writer.py`
- `scraping/validator_agent.py` (aggiunto controllo lunghezza minima)
```

---

## MCP server — gli strumenti dell'agente

I file markdown forniscono il contesto statico. I MCP (Model Context Protocol) server danno all'agente la capacità di agire sul progetto in autonomia durante la sessione.

La configurazione MCP va nel file `opencode.jsonc` del progetto — non globalmente, perché aggiungere troppi MCP aumenta il consumo di contesto. Meglio abilitare solo i server utili per il codebase su cui si lavora:

```jsonc
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": [
        "npx", "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/andrea/projects/integration-analyst-agent"
      ],
      "enabled": true
    },
    "git": {
      "type": "local",
      "command": [
        "uvx", "mcp-server-git",
        "--repository", "/Users/andrea/projects/integration-analyst-agent"
      ],
      "enabled": true
    },
    "review": {
      "type": "local",
      "command": [
        "npx", "-y",
        "@modelcontextprotocol/server-fetch"
      ],
      "enabled": true
    }
  }
}
```

> **Nota**: `@modelcontextprotocol/server-git` non è più disponibile su npm. L'alternativa funzionante è `mcp-server-git` via `uvx` — basta installare `uv` con `pip install uv --break-system-packages` e `uvx` è disponibile senza setup aggiuntivo.

I tre server svolgono ruoli distinti:

**`filesystem`** — permette all'agente di navigare, leggere e scrivere file nel progetto senza che tu debba incollare il contenuto manualmente. Su un progetto con 15+ file Python distribuiti in sottocartelle è la differenza tra un agente che lavora in autonomia e uno che dipende da te per ogni lettura.

**`git`** — l'agente può vedere diff, log e lo stato del repository. Durante la fase di plan questo è prezioso: capisce cosa è cambiato rispetto all'ultimo commit senza che tu debba spiegarlo.

**`review`** (fetch) — permette all'agente di recuperare documentazione esterna durante la sessione: docs di librerie, specifiche API, reference della stdlib. Elimina il copia-incolla di documentazione nel prompt.

---

## Il workflow in pratica

Una sessione tipo su un progetto attivo segue questo schema:

```
1. Apro il terminale nella cartella del progetto
2. Avvio opencode
3. L'agente legge AGENTS.md, TODO.md, checkpoint.md
4. Descrivo il task della sessione in una riga
5. L'agente fa il plan:
   - usa git per vedere cosa è cambiato dall'ultimo commit
   - usa filesystem per leggere i file rilevanti
   - usa fetch per consultare docs se necessario
6. L'agente implementa e aggiorna i file
7. Fine sessione: chiedo di aggiornare checkpoint.md e logs.md
```

Il vantaggio di questo schema è la **continuità tra sessioni**. Non importa se il contesto si azzera, se cambio macchina, o se riprendo un progetto dopo settimane: la prossima sessione trova tutto documentato e riparte da dove l'ultima si era fermata — senza che io debba respiegare l'architettura o il contesto ogni volta.

---

## Conclusione

La combinazione OpenCode + Ollama Cloud risolve i problemi che avevo con Claude Code: niente rate limit bloccanti nel mezzo di un task, modelli grandi disponibili on-demand, e un'architettura di contesto che rende le sessioni riproducibili.

Il vero valore però non è nello strumento — è nella disciplina dei file markdown. `AGENTS.md`, `TODO.md`, `logs.md` e `checkpoint.md` trasformano una sessione di coding assistito da un'esperienza usa-e-getta a un processo con memoria e continuità. Qualsiasi agente, su qualsiasi provider, può riprendere il lavoro da dove il precedente l'aveva lasciato.

---

*Stack: OpenCode · Ollama Cloud · MCP server (filesystem, git, fetch) · Mac Mini M4*
