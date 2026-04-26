---
title: "Welkomme and dear agent friend"
date: 2026-04-26
description: "Il primo post del blog: chi sono, di cosa scriverò, e come ho progettato un assistente AI personale self-hosted con Python, Telegram e OpenRouter."
tags: ["ai", "python", "telegram", "self-hosted", "llm"]
draft: false
---

Benvenuto. Questo è il primo post del mio blog, e voglio usarlo per presentarmi e spiegare di cosa parlerò qui.

Mi chiamo Andrea Raieta, sono un Cyber Security Developer. Lavoro ogni giorno con Python, Splunk, Docker e automazione SOC. Nel tempo libero esploro come applicare l'intelligenza artificiale a problemi concreti — non l'AI come hype, ma come strumento reale da integrare nel proprio workflow.

Questo blog nasce da una semplice convinzione: le cose interessanti che si imparano lavorando meritano di essere scritte e condivise. Troverai post su Python, cyber security, Splunk, automazione, architettura software e AI applicata. Tutto quello che incontro nel lavoro quotidiano e che vale la pena approfondire.

Per inaugurare, parto da un progetto su cui ho ragionato di recente: un assistente AI personale self-hosted.

---

## Il problema con gli assistenti AI esistenti

La maggior parte degli assistenti AI funziona in modalità pull: sei tu che vai da loro, apri una scheda, scrivi un messaggio, ricevi una risposta. Funziona, ma manca qualcosa di fondamentale.

Un assistente davvero utile dovrebbe conoscerti, ricordare il contesto delle conversazioni precedenti, agire in modo proattivo e girare su infrastruttura che controlli tu. Questo è il ragionamento dietro progetti come OpenClaw — un agente self-hosted che si connette alle tue app di messaggistica, gira un loop LLM continuo e può eseguire azioni reali per conto tuo.

## L'architettura in cinque layer

Prima di scrivere una riga di codice conviene pensare per layer. Un assistente AI personale è fondamentalmente cinque problemi distinti tenuti insieme, e mantenerli separati è ciò che rende il sistema estendibile.

**Layer 1 — Messaging frontend.** Qualunque app di chat già usi. Telegram è il punto di partenza più semplice: la Bot API è gratuita, ben documentata e stabile. L'obiettivo è astrarre questo layer in modo che il resto del sistema non sappia mai se sta parlando con Telegram, WhatsApp o una CLI.

**Layer 2 — Message router.** Normalizza i messaggi in arrivo in un formato interno consistente — qualcosa tipo `{user_id, text, timestamp}` — gestisce l'autenticazione e mette in coda i task. In Phase 1 è solo un webhook handler; nelle fasi successive diventa una coda vera.

**Layer 3 — Agent core.** Il processo long-running sulla tua macchina. Preleva un messaggio dalla coda, carica una persona e la memoria rilevante nel system prompt, invia il contesto completo all'LLM, fa il parsing della risposta e la rimanda indietro. Uno scheduler separato gestisce il comportamento proattivo — briefing mattutini, reminder, riepiloghi cron-triggered — iniettando messaggi sintetici nello stesso loop.

**Layer 4 — Tool e skill system.** Questo è ciò che separa un agente da un chatbot. I tool sono funzioni chiamabili: invia un'email, esegui un comando shell, apri una scheda del browser, interroga un calendario. Le skill sono composizioni di tool di livello più alto.

**Layer 5 — Persistenza.** Cronologia delle conversazioni, memory store dei fatti estratti dalle chat passate, skill file su disco. In Phase 1 è solo un dict Python in memoria. In Phase 3 diventa SQLite con ricerca semantica opzionale.

## Perché questa separazione conta

Il confine architetturale più importante è tra `bot/` e `agent/`. L'adapter Telegram non sa nulla di LLM. L'agent non sa nulla di Telegram. Questo rende sia l'espansione futura che la sostituzione futura lineari: aggiungere un canale WhatsApp significa scrivere un nuovo adapter senza toccare l'agent; sostituire OpenRouter con un modello locale Ollama significa cambiare un file senza toccare il bot.

## La struttura del progetto in Phase 1

Phase 1 ha uno scope deliberatamente stretto: uno skeleton funzionante che risponde ai messaggi su Telegram via un modello OpenRouter gratuito. Nessun database, nessun tool, nessuno scheduler. L'obiettivo è provare che tutto il plumbing funziona nel giro di un pomeriggio.

```
my-assistant/
├── .env                      # Token Telegram, chiave OpenRouter — mai committare
├── .env.example              # Template con le chiavi richieste — questo si committa
├── requirements.txt          # python-telegram-bot, httpx, python-dotenv
├── main.py                   # Entry point — avvia tutto
├── bot/
│   ├── __init__.py
│   ├── telegram_handler.py   # Riceve messaggi, chiama l'agent, invia la risposta
│   └── router.py             # Normalizza Telegram Update in plain dict
└── agent/
    ├── __init__.py
    ├── core.py               # Costruisce il prompt, chiama OpenRouter, ritorna la risposta
    ├── history.py            # Cronologia in memoria per user_id
    └── persona.py            # System prompt e nome dell'assistente
```

`agent/core.py` è il cuore. Recupera la cronologia per l'utente corrente, assembla un array di messaggi `[system_prompt] + [history] + [new_message]` e fa una POST all'endpoint `/chat/completions` di OpenRouter. Essendo OpenAI-compatible, è una singola chiamata `httpx.post` — nessun SDK necessario.

## Il roadmap a fasi

Provare a costruire tutto in una volta è il modo più comune per abbandonare un progetto come questo. Un approccio a fasi de-rischia ogni step e mantiene il sistema funzionante ad ogni checkpoint.

- **Phase 1** — skeleton funzionante: un canale, un LLM, nessun tool, nessuna persistenza
- **Phase 2** — tool layer: accesso shell, email e calendario via Google API
- **Phase 3** — memoria: fatti estratti dalle conversazioni e salvati in SQLite
- **Phase 4** — comportamento proattivo via scheduler
- **Phase 5** — l'agente scrive i propri skill file

## Cosa scriverò nei prossimi post

Nei post successivi entrerò nel dettaglio di ciascuna fase, ma non mi limiterò solo a questo progetto. Scriverò anche di Splunk, automazione SOC, Docker, Python applicato alla sicurezza e tutto ciò che incontro nel lavoro quotidiano e che merita di essere condiviso.

Se sei nel mondo della cyber security, dello sviluppo backend o semplicemente curioso di come si costruisce qualcosa di utile con l'AI, sei nel posto giusto.
