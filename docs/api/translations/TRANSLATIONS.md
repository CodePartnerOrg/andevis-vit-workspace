# Translation / Localization Architecture

## Overview

The API supports multi-language content for user-facing text (Estonian, English, Russian). Translations are stored in the database per company in a unified `i18n` JSONB column on each translatable entity.

## Supported Languages

Defined in `LocaleConstants` (`api/.../i18n/LocaleConstants.java`): **ET**, **EN**, **RU**.

Estonian (ET) is the default/base language.

Only supported languages are accepted when writing translations — unsupported language keys are filtered out.

---

## Locale Resolution

The API uses Spring Boot's built-in `AcceptHeaderLocaleResolver` configured in `LocaleConfig`:

- `Accept-Language: en` → `LocaleContextHolder.getLocale()` returns `en`
- Missing/unsupported header → defaults to `et`
- No manual language parameter passing needed — services read locale from `LocaleContextHolder`

**In controllers** — inject `Locale` as a method parameter:
```java
@GetMapping
public List<BreedDto> breeds(Locale locale) { ... }
```

**In services** — use `TranslatableHelper.currentLanguage()` or `LocaleContextHolder` directly:
```java
String lang = translatableHelper.currentLanguage(); // "et", "en", or "ru"
```

---

## Translation Storage — unified `i18n` JSONB column

Each translatable entity has a single `i18n` JSONB column with structure:

```json
{
  "name": { "en": "Annual leave", "ru": "Основной отпуск" },
  "title": { "en": "Leave request", "ru": "Заявление на отпуск" }
}
```

**Rules:**
- ET text stays in the primary column (`value`, `title`, `note`) — this is the base/default language
- The `i18n` map holds translations keyed by field name → language → text
- Adding a new translatable field requires **no schema change** — just add a new key to the JSON

---

## Resolution Priority

```
1. DB i18n override     (entity.i18n[fieldName][lang])
2. ET base value        (entity's primary column)
```

---

## `TranslatableHelper` (central service)

`api/.../i18n/TranslatableHelper.java`

Spring `@Component`. **Single entry point** for all translation operations — both low-level field resolution and entity-level `@Translatable` field handling. Reads the current request language automatically from `LocaleContextHolder`.

### Low-level methods

```java
// Resolve a field to the current request language
String resolve(String fieldName, String defaultValue,
               Map<String, Map<String, String>> i18n);

// All translations for a field
Translations allTranslations(String fieldName, String defaultValue,
                              Map<String, Map<String, String>> i18n);

// Current request language
String currentLanguage();
```

### Entity-level methods

```java
// Resolve all @Translatable fields
Map<String, Translations> resolveAll(Object entity);

// Resolve single @Translatable field
Translations resolve(Object entity, String fieldName);

// Write translations back to entity (sets primary column + i18n JSONB)
// Only SUPPORTED_LANGUAGES are stored; unsupported language keys are filtered out
void applyAll(Object entity, Map<String, Translations> fieldValues);
```

### Usage examples

```java
// Display endpoint — resolve to request language automatically
String title = translatableHelper.resolve("title", property.getTitle(), property.getI18n());

// Entity-level — resolve all @Translatable fields
Map<String, Translations> translations = translatableHelper.resolveAll(entity);

// Entity-level — resolve single field
Translations name = translatableHelper.resolve(reference, "name");

// Write direction — apply translations back to entity
translatableHelper.applyAll(entity, Map.of("title", titleTranslations));
```

---

## `@Translatable` Annotation

`api/.../i18n/Translatable.java`

Metadata annotation for entity fields. Declares which fields have translations.

```java
@Translatable("name")
@Column(columnDefinition = "TEXT")
private String value;  // ET name
```

---

## Adding a New Translatable Field to an Existing Entity

1. **Add `@Translatable` annotation** to the field on the entity.
2. **No schema migration needed** — just write the new field key into the existing `i18n` JSONB column.
3. **Use `TranslatableHelper.resolve()`** with the field name as the first argument.

---

## Key Files

| File | Role |
|---|---|
| `api/.../i18n/LocaleConfig.java` | `@Bean LocaleResolver` — AcceptHeaderLocaleResolver with ET default |
| `api/.../i18n/LocaleConstants.java` | Language constants (ET, EN, RU) |
| `api/.../i18n/Translatable.java` | `@Translatable` field annotation |
| `api/.../i18n/TranslatableHelper.java` | Central translation service (resolution + entity-level operations) |
| `api/.../reference/Reference.java` | Entity — `value`, `translations`, `i18n` (unified JSONB) |
| `api/.../application/entity/ApplicationProperty.java` | Entity — `title`, `note`, `i18n` (unified JSONB) |
| `api/.../vacation/service/VacationSettingsService.java` | Consumer of `TranslatableHelper` |
| `api/.../application/service/ApplicationPropertyService.java` | Consumer of `TranslatableHelper` |
| `api/src/main/resources/db/migration/V20__unify_i18n_columns.sql` | Adds `i18n` column to `_references` and `application_properties` |
