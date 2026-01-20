
# Project: Platform Localization via DeepL API

Following the launch of **Alfred**, a promising first partner has requested the platform in their local language. This document outlines the A-Z workflow to implement a professional, scalable translation system.

## 1. @Sarah (Technical Writer) – Glossary & Quality Control

Your goal is to ensure the AI uses industry-specific terminology.

1. **Create Glossary Files**: In the `/server/glossaries/` folder, create CSV files (UTF-8, no BOM) for each language.
* Format: `SourceTerm,TargetTerm` (No headers).
* Example `glossary-en-it.csv`:
```csv
Butler,AI Butler
Property,Structure
Host,Manager
Troubleshooting,Problem Resolution
Amenities,Included Services

```




2. **Review Output**: After @Michele runs the script, review `src/locales/[lang].json` to ensure the tone matches our "AI Butler" persona.

---

## 2. @Abdullah (Frontend Developer) – i18n Refactoring

Your goal is to **move away from hard-coded text**. This means no English strings should remain directly in `.jsx` or `.tsx` files.

1. **Install Dependencies**:
```bash
npm install i18next react-i18next i18next-browser-languagedetector

```


2. **Setup Config (`src/i18n.js`)**:
```javascript
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import en from './locales/en.json';
import it from './locales/it.json';

i18n.use(LanguageDetector).use(initReactI18next).init({
  resources: {
    en: { translation: en },
    it: { translation: it }
  },
  fallbackLng: 'en',
  interpolation: { escapeValue: false }
});
export default i18n;

```


3. **The Refactoring Process (The "How-To")**:
* **Step A**: Find a string in a component: `<h1>Accommodation address</h1>`.
* **Step B**: Cut the text and move it to `src/locales/en.json` under a logical key.
* **Step C**: Replace the text in the component with the `t()` function.


```jsx
// src/components/PropertyForm.jsx
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();
// Old: <h1>Accommodation address</h1>
// New:
<h1>{t('property.address')}</h1>

```


4. **Source File Structure (`src/locales/en.json`)**:
```json
{
  "sidebar": { "dashboard": "Dashboard" },
  "property": { "address": "Accommodation address" }
}

```



---

## 3. @Michele (Backend Developer) – DeepL Automation Script

Your goal is to translate Abdullah's `en.json` automatically while respecting Sarah's glossaries.

1. **Install Library**: `npm install deepl-node`.
2. **Implementation Script (`scripts/translate-ui.js`)**:
```javascript
const deepl = require('deepl-node');
const fs = require('fs');
const path = require('path');

const translator = new deepl.Translator(process.env.DEEPL_AUTH_KEY);

async function translate() {
  const enData = JSON.parse(fs.readFileSync('../src/locales/en.json', 'utf8'));

  // Example for Italian - Repeat for other languages
  const entries = new deepl.GlossaryEntries({ /* Load from Sarah's CSV */ });
  const glossary = await translator.createGlossary('hosp-en-it', 'en', 'it', entries);

  const itData = await translateRecursive(enData, 'it', glossary);
  fs.writeFileSync('../src/locales/it.json', JSON.stringify(itData, null, 2));
}

async function translateRecursive(obj, targetLang, glossary) {
  const result = {};
  for (const [key, value] of Object.entries(obj)) {
    if (typeof value === 'object') {
      result[key] = await translateRecursive(value, targetLang, glossary);
    } else {
      const res = await translator.translateText(value, 'en', targetLang, { glossary });
      result[key] = res.text;
    }
  }
  return result;
}
translate();

```



---

## 4. Maintenance Workflow (Summary)

1. **@Abdullah**: Adds new English keys to `en.json` during feature dev.
2. **@Sarah**: Updates CSV glossaries if new technical terms arise.
3. **@Michele**: Runs the script to sync all language JSONs.
4. **@Abdullah**: Commits all updated JSON files to the repo.

## Official Documentation

* **DeepL API (Setup & Auth):** [https://www.deepl.com/docs-api/accessing-the-api/](https://www.google.com/search?q=https://www.deepl.com/docs-api/accessing-the-api/)
* **DeepL Node Library:** [https://github.com/DeepLcom/deepl-node](https://github.com/DeepLcom/deepl-node)
* **React i18next (Tutorial):** [https://react.i18next.com/latest/usetranslation-hook](https://react.i18next.com/latest/usetranslation-hook)
