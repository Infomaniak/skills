---
name: loco-wording
description: Infomaniak translation and wording rules for Loco (localise.biz). Use whenever creating, translating, editing, or reviewing translations/assets through the Loco MCP (create_asset, update_translation, etc.) or any Infomaniak app string (kMail, kDrive, Calendar, SwissTransfer…). Mirrors the Infomaniak/loco_validator checks so generated strings pass validation.
---

# Loco wording rules (Infomaniak)

Apply these rules to **every** string you write to Loco. They mirror
[Infomaniak/loco_validator](https://github.com/Infomaniak/loco_validator) plus the
team's wording guide. If a rule conflicts with an existing translation provided by the user, the provided
translation wins (it is the source of truth) — but flag the conflict.

## Workflow

1. **Gather context** before writing: `Loco-MCP-get_asset` / `get_translations` for the key, and existing
   translations of similar keys (consistency of terminology beats creativity). Identify the app (Mail, Calendar,
   kDrive, SwissTransfer…) from tags/key/user — some rules are app-specific.
2. **Translate** following the translator brief below.
3. **Self-validate** every locale against the checklist (per-locale + cross-locale) before calling
   `Loco-MCP-update_translation` / `create_asset`.
4. Report which keys/locales you wrote. If a rule had to be broken on purpose, say why (the validator has
   per-key exceptions, e.g. a URL or a file-pattern containing `'`).

## Translator brief

- Preserve placeholders exactly (`%s`, `%d`, `%1$d`, `%@`, `{n}`…), punctuation style, line breaks and tone.
- Use **European** variants of each language (pt-PT not pt-BR, es-ES not LatAm, British English).
- German: aim for **Swiss German** usage (Swiss Standard German spelling).
- Use the most natural wording for each variant so it never reads as foreign.
- "transfer" / "transfert" = **file** transfer, never money transfer, unless stated otherwise.
- "attendee"/"guest" (EN) and "invité"/"participant" (FR): use the wording **"invité" (FR) / "attendee" (EN)**
  and their single equivalent consistently in all languages. Critical for Calendar; assume it applies when the app
  is unknown.
- Never pluralize product names (kDrive, kMeet, SwissTransfer, my kSuite, Infomaniak Mail, Infomaniak Calendar):
  not "your 3 kDrives" → rephrase ("your 3 kDrive spaces", "vos 3 espaces kDrive"…). Finnish inflections like
  "kDrivesta" are fine.

## Rules for all languages

| Rule | ✗ | ✓ |
|---|---|---|
| Typographic apostrophe | `'` | `’` |
| Ellipsis character | `...` | `…` |
| No escaped newline literal `\\n` | `\\n` | real newline |
| No pluralized product names | `kDrives` | see brief |

## Cross-locale rules (compare all locales of a key)

- **Same placeholders** in every locale (same set and count, e.g. `%1$s` + `%2$d` everywhere). For plurals, each
  plural form (`one`, `other`…) must match across locales. `%%` is a literal percent, not a placeholder.
- **Final period consistency**: either every locale ends with `.` or none does. Match the source.
- **Trailing newline consistency**: either every locale ends with a newline or none does.

## Per-language rules

### Français (fr)
- **e-mail**: the word is written `e-mail` (never `email`, `mail`, `mails` on its own).
- **Exception — "adresse mail"**: after *adresse*, and in compounds like *stockage mail*, *application mail*,
  *app mail*, *Infomaniak Mail*, write `mail` (never `adresse e-mail` / `adresse email`).
- Write **évènement** (not *événement*).
- Titles: verbs in the **infinitive** ("Supprimer le message").
- App **Mail**:
  - Other strings: verbs in the **2nd person plural** / vouvoiement ("Supprimez", "Vous pouvez…").

### English (en)
- **email** without hyphen (never `e-mail`).
- **British spelling**: `colour` (not `color`), and generally en-GB spelling.
- App **Mail**: the AI writing assistant is **the AI writing assistant**.

### German (de)
- Never `ẞ`/`ß` → use **`ss`** (Swiss German).
- **Speichern** = save (file, settings, draft…). **Aufnehmen** only for taking a photo/video.
- Never **Überweis…** (money/patients transfer) → use **Transfer** and derivatives.
- Conversation = **Unterhaltung**, never *Gespräch*.

### Italiano (it)
- Where French uses *vous*, Italian uses **tu** (2nd person **singular**), never *voi*/*Lei*.
- Light/dark theme: **tema**, **chiaro**, **scuro** (not *thema*, *claro*, *luce*, *oscuro*).

### Español (es) — Spain
- Trash = **papelera** (not *basura*).
- Disabled/deactivated = **desactivado** (not *discapacitado*).
- Conversation thread = **conversación** (not *hilo*).

### Português (pt) — Portugal
- Delete = **eliminar** (not *excluir*).
- Password = **palavra-passe** (not *senha*).
- User = **utilizador** (not *usuário*).
- Mobile phone = **telemóvel** (not *celular*).

### Ελληνικά (el)
- Question mark is the Greek **`;`** (U+037E), never `?`.

## Final checklist (run mentally on each string before writing)

- [ ] `’` not `'`; `…` not `...`
- [ ] No trailing space, no space around `\n`, no literal `\\n`
- [ ] Placeholders identical across locales; final `.` and trailing newline consistent across locales
- [ ] Product names not pluralized
- [ ] Language punctuation spacing (FR: space before `: ? !`; others: none; EL: `;` as `?`)
- [ ] Language vocabulary rules above (FR e-mail/adresse mail/évènement, EN email/colour, DE ss/Speichern/Unterhaltung/Transfer, IT tu/tema, ES, PT-PT)
- [ ] App-specific rules (Mail: FR infinitive titles / vous; assistant name; Calendar: invité/attendee)
