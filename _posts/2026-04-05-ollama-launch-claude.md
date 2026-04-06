---
layout: post
tag: de
title: Ollama startet Claude
subtitle: Mit Gemma4 endlich alles lokal betreiben?
1ate: 2026-04-05
background: '/images/ki-sandbox.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

[Ollama](https://ollama.com) ist eigentlich schon totgeglaubt. Es war mal ein Werkzeug, um Large Language Models (LLM) lokal zu betreiben, runterzuladen und ansprechbar über eine API, quasi so wie docker. Es hatte seine Grenzen, da LLM nur mit GPU-Speicher liefen und das auch nur sehr sehr langsam. Und nicht jeder LLM-Provider bietet seine Modelle einfach so zum Download an. Mit Version 0.20 wird alles ganz anders. Und es gibt Gemma4, das erste LLM, was ohne GPU auskommt.

# Ollama Launch

Das ist neu. Ollama kann jetzt andere Programme starten. Mit `ollama launch` gelangt man zu claude code, cline, opencode und dem allseits beliebten openclaw. Dabei übernimmt Ollama die Authentifizierung und baut quasi eine Klammer um die bisherigen Tools. Im Ergebnis kann man plötzlich `claude code` mit seinem eigenen LLM betreiben.

# Gemma4

Google stellt sein neuestes Model als OpenSource unter dem Namen [Gemma4](https://ollama.com/library/gemma4) zur Verfügung. Es gibt sogenannte Edge-Device Varianten e2b und e4b, was auch auf normalen Computern laufen soll, sofern man 10GB Memory hat.
Leider ist das Kontextfenster mit 4k viel zu klein, um das Tooling im LLM zu aktivieren, damit die agentierte KI auch was macht.
Aber auch dem kann neuerdings geholfen werden.

# Pimp your own LLM

Mit Ollama kann man auch sein eigenes LLM erstellen. Schauen wir uns dazu das Gemma4-LLM einmal an:

```bash
$ ollama show gemma4:e2b
  Model
    architecture        gemma4
    parameters          5.1B
    context length      131072
    embedding length    1536
    quantization        Q4_K_M
    requires            0.20.0

  Capabilities
    completion
    vision
    audio
    tools
    thinking

  Parameters
    temperature    1
    top_k          64
    top_p          0.95

  License
    Apache License
    Version 2.0, January 2004
    ...
```

Es hat über 5 Milliarden Parameter und KANN bis zu 131k Kontextlänge. Wenn man es aber startet, wenn man sowas sehen:

```bash
$ ollama ps
NAME          ID              SIZE      PROCESSOR    CONTEXT    UNTIL
gemma4:e2b    7fbdbf8f5e45    7.9 GB    100% GPU     4096       4 minutes from now
```

Es werden also nur 4k genutzt. Wir schreiben uns ein `Modelfile` und erhöhen diesen Wert

```
FROM gemma4:e2b
PARAMETER num_ctx 128000
SYSTEM "You are a senior DevOps Engineer. Your primary job is to diagnose, troubleshoot, and resolve problems on Kubernetes clusters using `kubectl` and related tooling via Bash."
```

Neben der Parameteranpassung sagen wir dem LLM auch gleich noch, welche Rolle er einehmen soll und sparen uns so bei jedem Start diesen Kontext. Jetzt noch ein neues LLM erzeugen:

```bash
ollama create gemma4-k8s -f Modelfile
```

und schon kann es losgehen.

# Ollama launch

```bash
ollama launch claude --model gemma4-k8s:latest
```

<iframe width="560" height="315" src="https://www.youtube.com/embed/5PuXx68qAIY?si=LnfM_VVXESHyaxiq" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Fangen wir mal mit den guten Sachen an:

- Es werden tatsächlich als Agent lokale Kommandos ausgeführt
- Die Ergebnisse werden verarbeitet
- Das LLM behält den Kontext

ABER:

- Es dauert alles ewig
- Vom Wissenstand ist es eher Junior-Level und nicht Senior-Devops

Klar, nach 2-3 Rückfragen kommt er dann vielleicht auch zur richtigen Lösung mit `kubectl rollback`, aber so richtig alltagstauglich ist das nicht.

# Tips

Mit `export OLLAMA_HOST=http://192.168.0.27:11434` kann man Ollama auf einem anderen Computer verwenden, etwa zu Hause einen Mac Mini. Voraussetzung dort, ein Ollama läuft oder ist gestartet mit `ollama serve`. Standardmässig ist das 127.0.0.1 und muss entsprechend angepasst werden.

Mit einer [CLAUDE.md Datei](https://github.com/eumel8/k8s-agent/blob/main/CLAUDE.md) kann man das Verhalten von Claude LLM beeinflussen, etwa was erlaubt ist und was verboten, oder welche Rolle das LLM einnehmen soll. Ansonsten sollte das Verzeichnis leer sein, in dem `ollama launch` gestartet wird, da es sonst erstmal lokal alle Dateien untersucht. Hintergrund: Es ist ein Coding-Werkzeug und soll eigentlich helfen, Programmcode zu schreiben.

# Fazit

Der autonome KI-Agent funktioniert eigentlich ganz gut. Es ist so bisschen wie OpenClaw, nur mit etwas mehr Kontrolle. Man müsste ihm nur die Langsamkeit austreiben und etwas intelligenter machen. Aber dazu bedarf es grössere Modelle, die Gemma4 auch bereitstellt. Aber im die lokal zu laufen, brauch man auch mehr Rechenleistung und GPU-Speicher. Agentierte KI ist also noch nicht ganu dort, wo sie sein sollte.
