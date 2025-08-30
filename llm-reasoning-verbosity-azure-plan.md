# Implementierungsplan: Reasoning-Effort, Verbosity, OpenAI/Azure-Parität und TUI-Unterstützung (vereinfachtes, konsistentes Design)

Ziele (KISS, konsistente Semantik)
- Reasoning Effort und Verbosity werden identisch behandelt:
  - Bindung an can_reason: Beide wirken nur, wenn das Modell reasoning-fähig ist.
  - Vorrang/Reihenfolge: SelectedModel-Override > Model-Default > nichts.
  - Request-Injektion: per-Call; UI (Sidebar) zeigt den effektiven Wert (Override, sonst Default).
- Parität OpenAI/Azure: Gleiches Verhalten für beide Provider.
- Anthropic: Unverändert; „Thinking“ (bool) bleibt separat.
- Kein providers.extra_body für „verbosity“ (explizit gefiltert).
- OOB-Default für Verbosity direkt am Modell in providers.<id>.models[].default_verbosity (enum: low|medium|high).
- Modellwechsel: „Reset to defaults“ (keine Prompts).

Dieser Plan ist für zwei PRs ausgelegt (PR 1: Funktional/Anzeige; PR 2: TUI/Persistenz).

---

## 0) Kontext (Kurzfassung)

- Catwalk liefert bekannte Provider/Modelle. Crush lädt diese und mapt sie auf interne Provider/Modelle.
- Beim Start: configureProviders() mergen Katalog und lokale Konfiguration; configureSelectedModels() setzt die Slots „large“/„small“.
- Requests: Provider-Clients bauen die Parameter inkl. Reasoning/Verbosity (falls unterstützt) und senden.
- UI (Sidebar): Zeigt aktuelle Slot-Einstellungen (Model, Reasoning/Thinking, Verbosity).

---

## 1) Zielbild und Prinzipien (final)

- Reasoning Effort und Verbosity identisch, mit can_reason-Gating.
- Vorrang: SelectedModel.{reasoning_effort | verbosity} > Model-Default (Reasoning: default_reasoning_effort, Verbosity: default_verbosity) > nichts.
- Request und UI nutzen denselben effektiven Wert.
- Kein providers.extra_body["verbosity"].
- Reset-on-switch: Beim Wechsel werden Slot-Werte deterministisch auf Defaults gesetzt.

---

## 2) PR 1 – Funktional, Parität, Defaults und Anzeige

### A) OpenAI/Azure: can_reason + Injektion + Defaults (identisch für Effort/Verbosity)

- ReasoningEffort:
  - Fallback auf model.DefaultReasoningEffort, wenn SelectedModel.ReasoningEffort leer und model.CanReason.
- Verbosity:
  - SelectedModel.Verbosity wird genutzt, wenn gesetzt und model.CanReason.
  - Ist SelectedModel.Verbosity leer und model.CanReason: Fallback auf Modell-Default aus providers.<id>.models[].default_verbosity.
  - Request-Injektion: option.WithJSONSet("verbosity", v) nur, wenn v != "" und model.CanReason.
  - providers.extra_body["verbosity"] wird explizit herausgefiltert.

Implementiert:
- internal/llm/provider/openai.go
  - can_reason-Gating und per-Call Injektion mit Fallback auf Model.default_verbosity.
  - Filter in createOpenAIClient: extra_body["verbosity"] wird ignoriert.
- internal/llm/provider/azure.go
  - Filter für extra_body["verbosity"]; Calls laufen über openaiClient → Parität.

### B) OOB-Default für Verbosity direkt am Modell

- Schema: Model.default_verbosity (enum) ergänzt.
- Loader: providers.<id>.models[] wird zusätzlich als Raw gelesen und default_verbosity intern pro model.id erfasst (keine public Overrides nötig).
- Ergebnis: Lokale provider models[].default_verbosity funktioniert sofort OOB; keine Catwalk-Änderung erforderlich.

Implementiert:
- internal/config/config.go
  - Internal map DefaultVerbosityByModel (json:"-") + Custom Unmarshal für ProviderConfig, das models[].default_verbosity ausliest.
- internal/config/load.go
  - Durchreichen der DefaultVerbosityByModel.
- internal/tui/components/chat/sidebar/sidebar.go
  - Anzeige-Fallback für Verbosity (SelectedModel.Verbosity oder DefaultVerbosityByModel), nur wenn can_reason.

### C) Reset-on-Switch (Konsistenz)

- Beim Wechsel (Slot):
  - MaxTokens = model.DefaultMaxTokens
  - ReasoningEffort = model.DefaultReasoningEffort (falls can_reason)
  - Verbosity = "" (→ Fallback auf default_verbosity, falls can_reason)
  - Think = false

Implementiert:
- internal/config/load.go: ReasoningEffort wird auf Default gesetzt, wenn nicht explizit; Verbosity bleibt leer (Fallback zur Laufzeit via Request/UI). TUI erzwingt in PR 2 den Reset beim Modellwechsel.

---

## 3) PR 2 – TUI/Persistenz

- Commands:
  - „Set Reasoning Effort“ (minimal/low/medium/high) – nur sichtbar, wenn can_reason.
  - „Set Verbosity“ (low/medium/high) – nur sichtbar, wenn can_reason.
  - „Switch Model“ – erzwingt immer „reset to defaults“ (keine Prompts).
- Persistenz:
  - Änderungen an SelectedModel via config.UpdatePreferredModel(...), Ziel: ~/.local/share/crush/crush.json.

---

## 4) Precedence & Zusammenspiel

- Vorrang:
  - SelectedModel.{reasoning_effort | verbosity} > Model-Defaults (default_reasoning_effort / default_verbosity) > nichts.
- can_reason:
  - Beide Felder gelten nur, wenn model.CanReason == true.
- Kein providers.extra_body für „verbosity“.

---

## 5) Tests, Verifikation, BC

- Unit-/Manuelle Tests:
  - Wenn SelectedModel.Verbosity leer und can_reason=true und default_verbosity gesetzt → Request enthält „verbosity“; UI zeigt denselben Wert.
  - Wenn can_reason=false → kein „verbosity“ im Request/Anzeige.
- Backwards-Compatibility:
  - Keine Breaking Changes. Neue Option default_verbosity in models[] ist additiv.
  - Sichtbare Änderung: providers.extra_body["verbosity"] greift nicht mehr (KISS; dokumentiert).

---

## 6) „Richtiger“ Azure-Provider für Non-OAI-Kompatible Modelle (optional später)

- Nicht Teil dieses Wurfs. Später separater Provider-Typ falls benötigt.

---

## 7) Branch-/Commit- und Review-Strategie (aktualisiert)

- Branches:
  - pr1-reasoning-verbosity-parity-can-reason-and-model-defaults
  - pr2-ui-toggles-effort-verbosity-reset
- PR 1 Commits:
  1) config.go/load.go: internal DefaultVerbosityByModel + Unmarshal + Durchreichen
  2) openai.go/azure.go: can_reason-Gating + Fallback (Model.default_verbosity) + Filter extra_body["verbosity"]
  3) sidebar.go: Anzeige Reasoning/Verbosity inkl. Fallback
  4) schema.json: Model.default_verbosity (enum)
  5) README: Kurze Doku (siehe unten)
- PR 2 Commits:
  1) commands.go: TUI Commands (Effort/Verbosity) + Switch Model => reset to defaults
  2) README: Abschnitt zu TUI-Befehlen

---

## 8) Akzeptanzkriterien

- PR 1:
  - Verbosity identisch zu Reasoning Effort (can_reason-Bindung, per-Call Injektion, effektive Anzeige)
  - Verbosity-Default via providers.<id>.models[].default_verbosity
  - providers.extra_body["verbosity"] wird ignoriert
  - OpenAI/Azure verhalten sich identisch
- PR 2:
  - TUI-Commands setzen Effort/Verbosity; Modellwechsel setzt immer auf Defaults; UI/Requests spiegeln Werte

---

## 9) Risiken & Mitigation

- Verwechslung mit extra_body["verbosity"]:
  - Mitigation: explizit gefiltert; README-Hinweis
- Catwalk-Default:
  - Sobald Catwalk default_verbosity liefert, greift es OOB (wir lesen es aus models[]); kein weiterer PR nötig

---

## 10) Doku/README (Kurztext)

- Configure per-model defaults:
  ```json
  {
    "providers": {
      "openai": {
        "id": "openai",
        "type": "openai",
        "api_key": "$OPENAI_API_KEY",
        "base_url": "$OPENAI_API_ENDPOINT",
        "models": [
          {
            "id": "gpt-5",
            "default_verbosity": "high"
          }
        ]
      }
    }
  }
  ```
- Precedence:
  - SelectedModel.{reasoning_effort|verbosity} > model defaults > none (only when can_reason)
- Note:
  - providers.extra_body["verbosity"] is ignored to avoid split-brain configuration
  - Model switch resets to defaults (no prompts)

---

## Anhang – PR-Texte (englisch)

PR 1 Title:
Reasoning Effort & Verbosity parity (OpenAI/Azure), can_reason binding, per-model default_verbosity, and UI alignment

PR 1 Summary:
This PR fixes inconsistencies and aligns OpenAI and Azure:
- Reasoning Effort default is now applied in requests when SelectedModel is empty (UI and request are aligned).
- Verbosity behaves identically to Reasoning Effort:
  - Applies only when the model can_reason = true
  - Precedence: SelectedModel override > per-model default_verbosity > none
  - Injected per-call; sidebar shows the same effective value
- Per-model defaults: default_verbosity is read from providers.<id>.models[].default_verbosity; no provider-wide extra_body
- Azure matches OpenAI behavior (headers/body/per-call options)
- Model switch always resets the slot to defaults (no prompts)

Examples of fixed issues:
- Missing Reasoning Effort in requests while UI showed a default:
  - Before: UI badge “Reasoning High” (from model default) but HTTP request contained no reasoning_effort when models.large.reasoning_effort was empty.
  - After: If SelectedModel.ReasoningEffort is empty and can_reason = true, we send the model default (ReasoningEffort) in the request; UI and request are aligned.
- Azure parity for extra headers/body and per-call options:
  - Before: providers.azure.extra_headers/extra_body not applied; per-call options behaved differently vs. OpenAI.
  - After: Azure now applies extra headers/body like OpenAI. Note: extra_body["verbosity"] is intentionally ignored to avoid split-brain configuration.
- Verbosity not applied/not visible:
  - Before: Verbosity neither injected into OpenAI/Azure requests nor visible in Sidebar.
  - After: Verbosity is injected per-call (only when can_reason = true), with precedence SelectedModel.Verbosity > model default_verbosity. Sidebar shows the same effective value.
- Selected vs. Catalog model inconsistencies:
  - Before: Switching models could carry stale overrides, causing mismatches with model defaults.
  - After: Model switch resets to defaults deterministically (MaxTokens, ReasoningEffort default, Verbosity via default_verbosity fallback, Think=false).

Docs:
- README: Add “Per-model defaults” (default_verbosity), precedence, and note that extra_body["verbosity"] is ignored

BC:
- No breaking changes; default_verbosity is additive and optional

PR 2 Title:
TUI Commands: Set Reasoning Effort and Verbosity; model switch resets to defaults

PR 2 Summary:
- Adds TUI commands to set SelectedModel Reasoning Effort (minimal/low/medium/high) and Verbosity (low/medium/high) for can_reason models
- Model switch always resets the slot to defaults (no prompts)
- Persist in ~/.local/share/crush/crush.json; UI reflects changes immediately
- Persistence uses the same mechanism and location as Anthropic’s “Think”: SelectedModel fields are written via config.UpdatePreferredModel(...) to the data config at ~/.local/share/crush/crush.json (no separate persistence layer).

Shared Issue (optional follow-up):
Unify default_verbosity in Catwalk model metadata
- Crush already reads default_verbosity from local provider models[] OOB
- When Catwalk adds default_verbosity, it will be picked up directly
- Keep precedence: SelectedModel > per-model defaults
