---
layout: post
tag: de
title: Opencode K8S
subtitle: Die KI repariert jetzt endlich unsere Cluster
1ate: 2026-04-06
background: '/images/gitlab-ci-k8s.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

De Facto Standard für agentierte KI ist `claude code`. Man installiert es mit einem Einzeiler auf Mac/Linux/Windows, startet einfach `claude`, loggt sich bei Anthropic ein, lädt dort sein Konto noch mit ein paar Euro auf und schon geht's los.

Dabei nutzen wir im Betrieb das Tool etwas anders als der Entwickler, der damit Code produzieren lässt.

# Anwendungsfall Kubernetes Cluster reparieren

<iframe width="560" height="315" src="https://www.youtube.com/embed/rXHeNxkzR1I?si=tNQjeRIuVrU6CaFr" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Agenten als solche

Das Problem bei Claude Code ist, dass es nur mit Claude funktioniert. Man will aber vielleicht eine gekappselte Lösung auf eigener Infrastruktur oder zumindest nicht die hausinternen Informationen zu Trainingszwecken mit der Welt teilen. 
Dazu gibt es Produkte wie ChatGPT Enterprise oder Github Copilot Enterprise. Im Hintergrund werden zwar dieselben Large Language Models (LLM) benutzt, aber im Gegensatz zu personellen Accounts verwenden die dortigen Agents die Eingabedaten des Users nicht weiter. 

Aber was sind jetzt diese Agents? Und wie kommen die Daten überhaupt zum LLM? ChatGPT beantwortet diese Frage so:


+-------------------+
|       User        |
|  (Prompt/Input)   |
+---------+---------+
          |
          v
+-----------------------------+
|     Input Processing        |
|  - Parsing                  |
|  - Context Injection        |
|  - Memory (optional)        |
+-------------+---------------+
              |
              v
+-----------------------------+
|        LLM Core             |
|  (Foundation Model)         |
+-------------+---------------+
              |
              v
+-----------------------------+
|   Reasoning / Thinking      |
|  - Task Decomposition       |
|  - Planning (Agent Loop)    |
|  - Chain-of-Thought (impl.) |
+-------------+---------------+
              |
      +-------+--------+
      |                |
      v                v
+-------------+   +------------------+
| Tool Calling|   | Internal Knowledge|
| (APIs, CLI, |   |  (Weights only)   |
|  DB, etc.)  |   +------------------+
+------+------+ 
       |
       v
+-----------------------------+
|   Environment Interaction   |
|  (e.g. kubectl, APIs, FS)   |
+-------------+---------------+
              |
              v
+-----------------------------+
|   Observation / Feedback    |
|  - Tool Results             |
|  - Errors / State           |
+-------------+---------------+
              |
              v
      (Agent Loop: iterate)
              |
              v
+-----------------------------+
|   Response Construction     |
|  - Formatting               |
|  - Filtering / Safety       |
+-------------+---------------+
              |
              v
+-------------------+
|       User        |
|   (Antwort)       |
+-------------------+

Das LLM ist zwar das Gehirn (und brauch auch die meisten Resourcen), aber davor brauch es ein Input Prozessing, damit das LLM die Daten mundgerecht dargereicht wird. Und das Gehirn brauch vielleicht zusätzliche Informationen, die es durch Frage/ANtwort vom Benutzer erfragen kann, oder optimalerweise sich selbst beschafft, etwa mit `kubectl get nodes` vom Rechner des Benutzers. Agenten sind also sowas wie Hände und Füsse für das Gehirn.

# Opencode

[Opencode](https://opencode.ai/) ist also so eine agentierte KI, die Hände und Füsse eines Gehirns
Es kann mit einem Einzeiler installiert werden:

```bash
curl -fsSL https://opencode.ai/install | bash
```

Wer dem Shellscript aus dem Internet nicht traut, es gibt noch npm Installations-Versionen, denn es handelt sich im eine NodeJS App. 
Es installiert sich selbständig in `~/.opencode/bin/opencode` und von dort muss es auch im Terminal aufgerufen werden. Das nennt sich interaktive Terminaloberfläche (TUI).

Nach dem Start ist es mit dem hauseigenen Model verbunden und man kann sofort loslegen. Aber Halt, wir wollen ja unsere Daten behalten und Enterprise Produkte als LLM verwenden.

Kein Problem. Man ruft einfach `/connect` auf, um einen neuen LLM-Provider auszuwählen. 

Es bieten sich als Optionen an:

- OpenAI (Plus/Pro/API), muss dann einen API-Key eingeben und das Model seiner Wahl. Aber Vorsicht: ChatGPT Enterprise bietet nicht automatisch API-Zugang zu OpenAI. Das muss der Enterprise-Administrator extra freigeben, ansonsten bezahlt man an der Stelle extra mit einem Individualzugang.
- Github Copilot. Hier gibt es zwei Wege. Entweder man hat Github selbst gehostet unter eigener Domain oder einer Subdomain von github.com, dann muss man diese eingeben. Oder bei Github Public erkennt der Login, dass es sich um einen Enterprise-Account handelt und wird zum Single-Sign-On seiner Firma über https://github.com/login/device/ weitergeleitet. Dort muss man über Cortex einen Code eingeben, der von opencode generiert wurde und wenn der stimmt, ist man quasi eingeloggt. 
Jetzt noch das richtige Model wählen wie Claude Sonnet 4.6 und schon kanns losgehen.

# Claude Markdown

Es gibt immer wieder Horrormeldungen, wo die KI ganze Festplatten löscht und scheinbar ausser Rand und Band gerät. Nun, bei claude code gibt es bei jeder Aktion eine Sicherheitsabfrage, bevor wirklich was gemacht wird.
Ausserdem kann man das Verhalten des Agenten mit einer [CLAUDE.md Datei](https://github.com/eumel8/k8s-agent/blob/main/CLAUDE.md) steuern. Man beschreibt etwa, was die Rolle des Agenten sein soll, wie er an die Kubernetes Cluster rankommt, was der übliche Ablauf ist und welche Stop-Rules es gibt. Zum Beispiel: Rumgucken ist immer erlaubt, aber wenn was geändert oder gelöscht werden soll, soll der Agent explizit nachfragen.

# Fazit

Opencode ist die Handreichung eines Open-Source-Projekts, dass man jederzeit auf [Github](https://github.com/anomalyco/opencode) verfolgen und mitmachen kann, um herstellerunabhängig KI nutzen zu können. Dabei muss man nicht auf die grossen, leistungsstarken LLM verzichten. Bei der täglichen Arbeit kann es enorm helfen, den Aufwand stark reduzieren und Ausfallzeiten verkürzen. Der nächste Schritt wäre vielleicht die Selbstheilung, also der Prometheus Alertmanager schickt die KI los, statt über Webhook ein Ticket zu erstellen.

