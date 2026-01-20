Understood. Here is the complete technical specification translated into English for your team.

---

### 1. @Sarah (Technical Writer) – Glossary Configuration

Your task is to prepare the source files to instruct the AI on specific terminology.

1. **Create a local folder** named `translations-master`.
2. **Create CSV files** inside this folder. You need one file for each target language. Files must be **UTF-8** encoded (no BOM).
* `glossary-en-it.csv`
* `glossary-en-es.csv`


3. **Fill the files** using this structure (no headers):
```csv
Butler,AI Butler
Property,Property Structure
Host,Manager
Troubleshooting,Problem Resolution

```


4. **Send the files to @Michele** via your messaging platform or upload them to the repository in the `/server/glossaries/` folder.

---

### 2. @Abdullah (Frontend Developer) – React Refactoring

Your task is to remove static text from components and link them to the i18n library.

1. **Install the packages** in the React project:
`npm install i18next react-i18next i18next-browser-languagedetector`
2. **Create the configuration file** in `src/i18n.js`:
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


3. **Import the configuration** in `src/index.js` (or `main.jsx`):
`import './i18n';`
4. **Create the language folder**: `src/locales/`.
5. **Create the source file** `src/locales/en.json` and insert all platform strings:
```json
{
  "sidebar": { "dashboard": "Dashboard", "settings": "Butler Settings" },
  "property": { "address": "Accommodation address", "video": "Dishwasher troubleshooting video" }
}

```


6. **Modify React components**:
```jsx
import { useTranslation } from 'react-i18next';
const { t } = useTranslation();
return <h1>{t('property.video')}</h1>;

```



---

### 3. @Michele (Backend Developer) – Automatic Translation Script

Your task is to create the script that reads @Abdullah's file and uses @Sarah's glossaries.

1. **Install the DeepL library** (in the backend):
`npm install deepl-node`
2. **Create the script** `scripts/translate-ui.js`:
```javascript
const deepl = require('deepl-node');
const fs = require('fs');
const authKey = "YOUR_DEEPL_AUTH_KEY"; // Fetch from environment variables
const translator = new deepl.Translator(authKey);

async function run() {
  // 1. Load @Sarah's glossary
  const entries = new deepl.GlossaryEntries({
    entries: { "Butler": "AI Butler", "Property": "Property Structure" } // Or read from CSV
  });
  const glossary = await translator.createGlossary('hospitality-it', 'en', 'it', entries);

  // 2. Read @Abdullah's file
  const enJson = JSON.parse(fs.readFileSync('../src/locales/en.json', 'utf8'));
  const itJson = {};

  // 3. Translate
  for (const [key, section] of Object.entries(enJson)) {
    itJson[key] = {};
    for (const [subKey, text] of Object.entries(section)) {
      const result = await translator.translateText(text, 'en', 'it', { glossary });
      itJson[key][subKey] = result.text;
    }
  }

  // 4. Save the file for @Abdullah
  fs.writeFileSync('../src/locales/it.json', JSON.stringify(itJson, null, 2));
}
run();

```


3. **Run the script**: `node translate-ui.js` whenever @Sarah or @Abdullah update their files.

---

### 4. Maintenance Workflow

For every new feature added to the platform:

1. **@Abdullah** adds the English key to `en.json` and uses `t('key')` in the code.
2. **@Sarah** checks if new terms are needed in the glossary CSVs.
3. **@Michele** runs the `node translate-ui.js` script.
4. **@Abdullah** commits the updated `en.json` and `it.json` files.

---

### Official Documentation Links

* **DeepL API (Setup & Auth):** [https://www.deepl.com/docs-api/accessing-the-api/](https://www.google.com/search?q=https://www.deepl.com/docs-api/accessing-the-api/)
* **DeepL Node Library:** [https://github.com/DeepLcom/deepl-node](https://github.com/DeepLcom/deepl-node)
* **React i18next (Tutorial):** [https://react.i18next.com/latest/usetranslation-hook](https://react.i18next.com/latest/usetranslation-hook)
