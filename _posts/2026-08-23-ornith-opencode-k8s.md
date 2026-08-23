---
layout: post
tag: de
title: Ornith im K8S-Nest
subtitle: Das lokale DevOps-LLM für OpenCode - und nicht nur auf dem Mac
date: 2026-08-23
background: '/images/ornith.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Vor ein paar Monaten habe ich [Ollama beim Starten von Claude Code](https://blog.k8s.de/2026/04/ollama-launch-claude/) ausprobiert. Ergebnis damals: Es funktioniert, aber Gemma4 als Gehirn war eher Junior-Level und langsam. Der Ausweg war [OpenCode](https://blog.k8s.de/2026/04/opencode-k8s/) mit einem grossen Cloud-Modell — funktioniert gut, aber die Cluster-Daten verlassen das Haus.

Was fehlte, war ein lokales Modell, das klein genug für einen Heimserver, aber schlau genug für agentisches Arbeiten ist. Mit [Ornith-1.0](https://huggingface.co/ornith-ai/Ornith-1.0-9B) von DeepReinforce gibt es das jetzt. Und das Beste: Es läuft nicht nur auf einem Mac.

# Was ist Ornith?

[Ornith-1.0](https://deep-reinforce.com/understand_ornith_1_0.html) ist eine Familie von Open-Source-Modellen für agentisches Coding: 9B-Dense, 31B-Dense, 35B-MoE und 397B-MoE. Keine Modelle von Grund auf trainiert, sondern Post-Trainings auf Gemma4 und Qwen 3.5 — die 9B-Variante beruht auf Qwen 3.5-9B. Die Besonderheit ist das Self-Improving Training: Per Reinforcement Learning lernt das Modell nicht nur Lösungen zu erzeugen, sondern gleich auch die Scaffoldings, also die Werkzeugabläufe, mit denen es zu dieser Lösung kommt.

Das Ergebnis kann sich sehen lassen. SWE-bench Verified:

| Modell | Score |
|---|---|
| **Ornith-1.0-9B** | **69,4 %** |
| Qwen3.5-35B | 70,0 % |
| Gemma4-31B | 52,0 % |
| Qwen3.5-9B (Basis) | 53,2 % |
| Gemma4-12B | 44,2 % |

Ein 9B-Modell auf Augenhöhe mit einem 35B-Modell — und damit deutlich über der eigenen Basis. Dazu MIT-Lizenz, natives Tool Calling, 262K Kontextlänge. Genau die Zutaten, die ein DevOps-Agent braucht.

# GGUF und die Bit-Varianten

Für den lokalen Betrieb braucht es das Modell im [GGUF-Format](https://huggingface.co/ornith-ai/Ornith-1.0-9B-GGUF) — eine Einzeldatei mit Gewichten, Tokenizer und Metadaten für llama.cpp, Ollama oder LM Studio. Dabei werden die Gewichte quantisiert, also auf wenige Bits pro Parameter komprimiert. Das offizielle Repo liefert fünf Stufen:

| Variante | Größe | ~Bits/Gewicht | Typische Hardware |
|---|---|---|---|
| bf16 | 17,9 GB | 16 | Server-GPU ab 24 GB |
| Q8_0 | 9,5 GB | 8,5 | 16 GB+ RAM/GPU |
| Q6_K | 7,4 GB | 6,6 | Mac/HomeServer 24 GB+ |
| Q5_K_M | 6,5 GB | 5,5 | 16–24 GB |
| Q4_K_M | 5,6 GB | 4,9 | ab 32 GB inkl. Kontext |

Die Namenslogik: `Q4` = Zielpräzision in Bit, `_K` = k-quants (gruppierte Quantisierung mit Skalen pro Block), `_M` = *medium*-Mischung, bei der kritische Tensoren höherwertig bleiben. Ich fahre `Q4_K_M`, weil neben dem Modell noch genügend RAM für den KV-Cache des grossen Kontextfensters übrig bleibt. Wer mehr Speicher hat, nimmt `Q6_K` oder `Q8_0` und kauft sich etwas Präzision zurück.

# Nicht nur Mac

Ollama läuft überall: Linux, Windows, macOS, Docker. Damit ergeben sich drei sinnvolle Betriebsvarianten:

**Mac Mini (wie gehabt):** Apple Silicon hat vereinheitlichten Memory, das Modell liegt komplett im RAM und wird auf der GPU gerechnet. Läuft out-of-the-box.

**Linux-HomeServer mit GPU:** Ob Nvidia mit CUDA oder AMD mit ROCm — Ollama erkennt beides automatisch. Mein alter Desktop mit einer Radeon hat das Q4_K_M-Modell problemlos geladen, die Token-Rate reicht für agentisches Arbeiten locker.

**Reiner CPU-Server:** Auch ohne GPU geht es, solange genug RAM da ist (für 5,6 GB Modell plus Kontext schonend 16–32 GB). Es wird spürbar langsamer, weil jedes Token durch den Speicher muss, aber für eine Diagnose-Session am Abend ist das völlig okay.

Und wie schon beim letzten Mal beschrieben: Ollama muss gar nicht auf dem Rechner laufen, an dem man sitzt. Auf dem Server starten, von überall benutzen:

```bash
$ export OLLAMA_HOST=http://192.168.0.27:11434
```

# Eigenbau mit dem Modelfile

Das Modell zieht man direkt von Hugging Face — inklusive Quantisierungsstufe:

```bash
$ ollama run hf.co/deepreinforce-ai/Ornith-1.0-9B-GGUF:Q4_K_M
```

Interessanter wird es mit einem eigenen `Modelfile`. Denn Ornith ist ein Reasoning-Modell und denkt standardmässig laut vor sich hin in `<think>`-Blöcken. Für einen DevOps-Agenten will ich das unterdrücken und ihm stattdessen eine Rolle mitgeben:

```
FROM hf.co/deepreinforce-ai/Ornith-1.0-9B-GGUF:Q4_K_M

PARAMETER temperature 0.2
PARAMETER top_p 0.9
PARAMETER top_k 20
PARAMETER repeat_penalty 1.05
PARAMETER num_ctx 262144

SYSTEM """
You are an autonomous DevOps and Kubernetes engineering agent.
ACT WITH TOOLS NOW. Do not describe a command that you could execute. Execute it.
Never invent command output. Ground everything in real tool output.
For Kubernetes problems: inspect state, gather evidence, identify cause,
make the smallest appropriate change, verify the result.
Be extremely terse. Match the user's language.
/nothink
"""
```

Bauen und checken:

```bash
$ ollama create ornith-k8s -f Modelfile
$ ollama ps
NAME             ID               SIZE      PROCESSOR    CONTEXT     UNTIL
ornith-k8s:latest    a1b2c3d4e5f6    8.9 GB    100% GPU     262144      4 minutes from now
```

Der komplette Systemprompt samt Troubleshooting-Playbook liegt im Repo [eumel8/k8s-agent](https://github.com/eumel8/k8s-agent) — zusammen mit OpenCode-Konfiguration, MCP-Anbindung und Incident-Response-Skills.

# Anbindung an OpenCode

OpenCode spricht jede OpenAI-kompatible API, also auch die von Ollama. In der `opencode.json` des Projekts:

```jsonc
{
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama",
      "options": { "baseURL": "http://192.168.0.27:11434/v1" },
      "models": {
        "ornith-k8s:latest": {
          "name": "Ornith K8S",
          "capabilities": { "tools": true },
          "limit": { "context": 262144, "output": 32768 },
          "timeout": 600000
        }
      }
    }
  }
}
```

Wichtig sind zwei Dinge: `"tools": true`, sonst bekommt das Modell gar keine Funktionen angeboten, und ein grosszügiger Timeout, weil lokale Inferenz nun mal kein Rechenzentrum im Nacken hat. Im Projektverzeichnis liegt eine `AGENTS.md`, die dem Agenten seine Rolle als Senior DevOps Engineer beschreibt — inspect before acting, follow the evidence, smallest appropriate change.

Wer lieber Claude Code nutzt, geht den Weg über `ollama launch claude --model ornith-k8s`. Ich bleibe bei OpenCode, weil ich dort die Permissions (`bash`, `edit`, `write`) pro Projekt steuern kann.

# Beispiele

Erstmal die Grundprobe — einfache Inspektion ohne Umschweife:

```text
## User
execute kubectl to get nodes on the current context

## Assistant (63s)
Tool: bash → { "command": "kubectl get nodes" }

NAME   STATUS   ROLES                  AGE    VERSION
k8s    Ready    control-plane,worker   473d   v1.34.6
```

Keine Präambel, keine Rückfrage, direkt das Tool. Spannender wird es bei einer echten Diagnose:

```text
## User
im namespace staging startet mein pod frontend-7d9f8 immer wieder neu

## Assistant
Tool: bash → kubectl get pods -n staging
→ frontend-7d9f8b6c4-x2k9p   0/1   CrashLoopBackOff   8 (2m ago)

Tool: bash → kubectl describe pod frontend-7d9f8b6c4-x2k9p -n staging
→ Last State: Terminated, Reason: OOMKilled, Exit Code: 137

Tool: bash → kubectl logs frontend-7d9f8b6c4-x2k9p -n staging --previous | tail -20

Der Pod wurde mit OOMKilled beendet: Der Container überschreitet sein
Memory-Limit (256Mi), bevor die Anwendung startklar ist.

Vorschlag (Deployment patchen):
  resources.limits.memory: 512Mi

Soll ich das Change durchführen?
```

Genau das Verhalten, das ich im Systemprompt haben wollte: erst Evidenz sammeln (Pods, Describe, Logs), dann Ursache benennen (OOMKilled, Exit 137), dann den kleinstmöglichen Fix vorschlagen — und vor der Änderung nachfragen. Das Gleiche funktioniert mit Flux (`flux get kustomizations`, HelmRelease-Status), Storage (PVC → PV → StorageClass-Kette) und Netzwerk-Diagnose per `dig` und `curl` im Pod.

# Tips

Ein paar Dinge, die ich unterwegs gelernt habe:

- **Kontext frisst RAM.** `num_ctx 262144` klingt verlockend, aber der KV-Cache wächst mit jedem Token. Auf einer 16-GB-Maschine lieber bei 64K bleiben — das reicht für AGENTS.md plus Tooldefinitionen meistens aus.
- **MCP-Server kosten Tokens, auch wenn man sie nicht nutzt.** Jeder aktive Server lädt seine Tooldefinitionen in jeden Request. GitLab-MCP allein sind das schnell 10.000 Tokens pro Aufruf. Also nur aktivieren, was die aktuelle Aufgabe braucht.
- **`/nothink` spart Zeit.** Ohne Thinking-Block antwortet das Modell deutlich schneller. Für operatives Arbeiten brauche ich keinen Monolog, sondern Kommandos.
- **Permissions bewusst setzen.** In OpenCode kann ich `bash: allow` setzen und der Agent arbeitet vollautonom. Für Produktion-Cluster empfehle ich trotzdem, kritische Namespaces read-only zu halten bzw. den Agenten fragen zu lassen, bevor etwas geändert wird.
- **Temperature niedrig halten.** 0.2 statt der empfohlenen 0.6 macht die Diagnosen reproduzierbar. Für Code-Kreativarbeit mag höhere Temperatur gelten, für `kubectl` nicht.

# Fazit

Die Lücke von damals ist geschlossen: Mit Ornith gibt es ein lokales Modell, das agentisches Tool Calling beherrscht, auf einem Heimserver läuft — ob Mac, Linux mit GPU oder purer CPU — und im Kubernetes-Alltag wirklich Senior-Level arbeitet. OpenCode sorgt für die Hände und Füsse, das Modelfile für die Rolle, und die Daten bleiben im LAN. Cloud-Token-Kosten: null. Nächster Schritt wäre wieder die Selbstheilung: Der Alertmanager schickt bei einem P1-Alert den Agenten los, bevor der Oncall überhaupt das Handy in der Hand hat. Aber das ist Stoff für einen weiteren Beitrag.
