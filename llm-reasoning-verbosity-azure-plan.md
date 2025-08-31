# Combined PR Plan: Reasoning Effort & Verbosity Parity (OpenAI/Azure), can_reason Binding, Model Defaults, TUI Controls

Kurzfassung
- Ein PR, kleiner und review-freundlich, der alle Inkonsistenzen behebt und OpenAI/Azure paritätisch macht.
- Reasoning Effort und Verbosity werden absolut identisch behandelt (inkl. can_reason).
- Defaults kommen aus dem Katalog/der lokalen Provider-Modelldefinition (OOB), SelectedModel-Overrides haben Vorrang.
- TUI-Commands (Effort/Verbosity) sind enthalten, Persistenz identisch zu Anthropic (SelectedModel in ~/.local/share/crush/crush.json).
- Kein providers.extra_body["verbosity"] (explizit gefiltert).

Warum ein einziger PR?
- Repo ist High Frequency, ein PR reduziert Review-/Merge-Overhead.
- Bei Beanstandung kann die TUI-Komponente (nur commands.go) gezielt zurückgebaut werden.

Scope (in einem PR)
- Config/Loader/Schema:
  - Model.default_verbosity (enum: low|medium|high) in schema.json.
  - ProviderConfig custom Unmarshal: liest providers.<id>.models[].default_verbosity und hält die Werte intern pro model.id (OOB, ohne Catwalk-PR).
- Provider OpenAI & Azure:
  - can_reason Binding für ReasoningEffort und Verbosity.
  - Precedence: SelectedModel override > Model default > none.
  - Per-Call Injektion; providers.extra_body["verbosity"] wird ignoriert.
  - Azure-Client Parität bzgl. extra headers/body und per-call options.
  - Prompt-Mapping: Azure nutzt jetzt denselben Coder-Prompt wie OpenAI (v2.md) statt anthropic.md (vorheriger Gap).
- UI (Sidebar):
  - Anzeige Reasoning Effort (Selected vs. Default) und Verbosity (Selected vs. Default) nur wenn can_reason.
- TUI:
  - Commands: „Cycle Reasoning Effort (Current: X)“ und „Cycle Verbosity (Current: Y)“ statt 7 Einträgen. Reihenfolge: Effort minimal → low → medium → high → off; Verbosity low → medium → high → off. Sichtbar nur, wenn can_reason und Provider ∈ {openai, azure}.
  - Das Command-Window bleibt beim Toggeln geöffnet (kein Close); Labels aktualisieren sich live.
  - Persistenz über config.UpdatePreferredModel(...) (SelectedModel.*) nach ~/.local/share/crush/crush.json – identisch zu Anthropic „Think“.
  - Modellwechsel: „Reset to defaults“ (Effort=Default, Verbosity=Fallback auf default_verbosity, MaxTokens=Default, Think=false) – explizit im TUI Switch-Flow implementiert.

Semantik (final)
- Gilt für OpenAI/Azure:
  - Beide Felder nur, wenn can_reason==true.
  - SelectedModel.{reasoning_effort|verbosity} > Model.{default_reasoning_effort|default_verbosity} > none.
  - Request und UI zeigen denselben effektiven Wert.
  - Kein providers.extra_body["verbosity"] (gefiltert).
- Anthropic: unverändert (Think on/off).

Konfigurationsbeispiel (OOB, ohne Catwalk-PR)
```json
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "openai": {
      "id": "openai",
      "type": "openai",
      "api_key": "$OPENAI_API_KEY",
      "base_url": "$OPENAI_API_ENDPOINT",
      "models": [
        {
          "id": "gpt-5",
          "name": "GPT-5",
          "context_window": 400000,
          "default_max_tokens": 128000,
          "can_reason": true,
          "has_reasoning_efforts": true,
          "default_reasoning_effort": "high",
          "default_verbosity": "medium"
        },
        {
          "id": "gpt-5-mini",
          "name": "GPT-5 Mini",
          "context_window": 400000,
          "default_max_tokens": 128000,
          "can_reason": true,
          "has_reasoning_efforts": true,
          "default_reasoning_effort": "high",
          "default_verbosity": "medium"
        }
      ]
    }
  },
  "models": {
    "large": { "provider": "openai", "model": "gpt-5" },
    "small": { "provider": "openai", "model": "gpt-5-mini" }
  },
  "options": { "debug": true }
}
```
Hinweise:
- SelectedModel.{reasoning_effort|verbosity} kann optional gesetzt werden; ansonsten greifen die obigen Defaults.
- providers.extra_body["verbosity"] wird ignoriert (nicht senden).
- Modellwechsel setzt auf Defaults zurück.

Beispiele behobener Fehler (Before → After)
- Reasoning default im Request fehlte:
  - Before: UI zeigte „Reasoning High“ (Default), der Request enthielt kein reasoning_effort, wenn Selected leer war.
  - After: Wenn Selected leer und can_reason=true, wird default_reasoning_effort im Request gesetzt. UI == Request.
- Azure-Parität & Prompt:
  - Before: extra_headers/extra_body bei Azure wurden nicht angewendet; per-call Options verhielten sich anders als OpenAI. Zudem nutzte Azure anthropic.md statt v2.md.
  - After: Azure verhält sich analog zu OpenAI; extra_body["verbosity"] wird absichtlich ignoriert, um doppelte Quellen zu vermeiden. Azure verwendet jetzt denselben Prompt wie OpenAI (v2.md).
- Verbosity nicht injiziert/nicht sichtbar:
  - Before: Verbosity wurde weder in Requests injiziert noch in der Sidebar angezeigt.
  - After: Verbosity wird per-Call injiziert (nur can_reason) und in der Sidebar angezeigt – Precedence Selected > default_verbosity.
- Selected vs. Catalog Inkonsistenz beim Modelwechsel:
  - Before: Stale Overrides führten zu Mismatches bei Defaults.
  - After: Slot-Reset auf Defaults (Effort/MaxTokens/Think, Verbosity via Fallback) macht Verhalten deterministisch.

Tests/Verifikation
- Build/Tests grün: go build ./...; go test ./...
- ModelSelectedMsg setzt SelectedModel-Felder explizit zurück (ReasoningEffort="", Verbosity="", Think=false, MaxTokens=0) – deterministischer Reset.
- Debug-Logs (options.debug=true) zeigen:
  - Top-Level "verbosity": "<level>" nur wenn can_reason.
  - ReasoningEffort entsprechend SDK-Feld (enum) gesetzt.
  - Keine „verbosity“ aus providers.extra_body.
- Sidebar zeigt dieselben effektiven Werte wie der Request.

Changelog (Kurz, Englisch)
- Fix: Apply Reasoning Effort default in requests when SelectedModel is empty (UI and request aligned).
- Add: Per-model default_verbosity (providers.<id>.models[].default_verbosity); OpenAI/Azure parity; can_reason binding for both.
- Change: Ignore providers.extra_body["verbosity"] to avoid split-brain configuration.
- Add: TUI commands to set Reasoning Effort and Verbosity; model switch resets to defaults.

Pull Request (English, final – single PR)
Title
Reasoning Effort & Verbosity parity (OpenAI/Azure), can_reason binding, per-model default_verbosity (OOB), TUI controls, and UI alignment

Summary
This PR fixes inconsistencies and aligns OpenAI and Azure. Reasoning Effort and Verbosity now behave identically:
- Apply only when can_reason = true
- Precedence: SelectedModel override > per-model defaults > none
- Injected per call; sidebar shows the same effective value
- Per-model defaults: default_verbosity read OOB from providers.<id>.models[] (no provider-wide extra_body)
- Azure matches OpenAI behavior (headers/body/per-call options)
- TUI commands: set Reasoning Effort (minimal/low/medium/high) and Verbosity (low/medium/high)
- Model switch resets the slot to defaults (no prompts)

Examples of fixed issues
- Reasoning Effort default missing in requests while UI showed a default:
  - Before: UI displayed “Reasoning High” (from model default), but request did not include reasoning_effort when SelectedModel.ReasoningEffort was empty.
  - After: If Selected is empty and can_reason = true, the request includes the model default (ReasoningEffort). UI and request are aligned.
- Azure parity, prompt, and request customization:
  - Before: Azure did not apply provider extras (headers/body) and per-call options symmetrically to OpenAI; features depending on request customization (e.g., verbosity via per-call options) were ineffective. In addition, reasoning_effort suffered from the same default-missing behavior as OpenAI. Also, Azure used anthropic.md instead of v2.md for the coder prompt.
  - After: Azure now mirrors OpenAI for extras and per-call options; and reasoning_effort default is applied in requests (when Selected is empty and can_reason = true). Azure also uses the same coder prompt (v2.md) as OpenAI, ensuring consistent behavior and guidance across providers.
- Verbosity was neither injected nor visible:
  - Before: No request injection and no Sidebar badge.
  - After: Injected per call (only when can_reason), with precedence Selected > default_verbosity; Sidebar shows the same effective value.
- Selected vs. Catalog inconsistencies on model switch:
  - Before: Stale overrides caused mismatches with model defaults.
  - After: Forced reset to defaults (Effort/MaxTokens/Think; Verbosity via default_verbosity fallback) yields deterministic behavior.

Persistence
Identical to Anthropic “Think”: SelectedModel fields are persisted via config.UpdatePreferredModel(...) to ~/.local/share/crush/crush.json. No separate persistence layer.

Out-of-scope
Non-OpenAI-compatible Azure endpoints (would require a dedicated provider type).

Submission note
- This plan file is maintained internally for review and is not part of the submitted PR. It will be removed from the PR diff prior to opening.

Rollback
- If required to split PRs, TUI can be reverted in a single file (internal/tui/components/dialogs/commands/commands.go); the core fixes remain intact.

Optional follow-up (English)
Unify default_verbosity in Catwalk model metadata. Crush already reads default_verbosity from local provider models[] OOB; once Catwalk supplies it, behavior is unchanged.
