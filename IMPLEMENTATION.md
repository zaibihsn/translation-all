# Implementation Guide

This guide provides comprehensive details on how the translation data is structured and how to implement it across various programming languages and tech stacks. 

## 1. Data Structure

The repository organizes Quranic translations into an extremely lightweight, page-based format.

### Global Metadata
At the root of the repository, you'll find an index of all available translations in two formats: `translations.json` and `translations.toon`. 
These contain a list of objects with the following fields:
- `id`: The unique identifier for the translation (e.g., `english-hilali-khan`).
- `name`: The display name of the translation.
- `author`: The author/translator's name.
- `lang`: The language code (e.g., `english`, `urdu`).
- `script`: The writing script (e.g., `Latin`, `Arabic`).
- `dir`: Text direction (`ltr` or `rtl`).
- `has_footnotes`: Whether the translation contains footnote data.

### Translation Archives (.zip)
Each translation is compressed into a `.zip` file named after its `id` (e.g., `english-hilali-khan.zip`). Compressing the data allows fetching an entire translation (often ~500 KB to 1 MB) in a single rapid network request.

Inside each `.zip` file:
- `meta.json`: Local metadata specific to this translation.
- `pages/`: A directory containing exactly 604 files, named `1.toon` to `604.toon`. Each file corresponds to a standard Madani Quran page.

### The `.toon` File Format
To optimize for size and parsing speed, we use a custom `.toon` schema instead of JSON for the translation data. A `.toon` file represents a table of tabular data.

**Example `pages/100.toon`:**
```text
quran[6]{c,v,t,f}:
4,135,"O you who believe! Stand out firmly for justice...",footnotes:1:(V.135) Narrated Anas...
4,136,"O you who believe! Believe in Allâh..."
```

**Schema Breakdown:**
- **Header (`quran[6]{c,v,t,f}:`)**: Indicates the schema. The characters in the braces define the columns:
  - `c` = Chapter (Surah) number
  - `v` = Verse (Ayah) number
  - `t` = Translation text (enclosed in quotes `""`)
  - `f` = Footnotes (optional, appears if `has_footnotes` is true)
- **Data Rows**: Comma-separated values corresponding to the columns.

---

## 2. Implementation by Tech Stack

Because the data is zipped and uses a custom `.toon` format, you'll need to fetch the zip, extract it in memory, and parse the text.

### JavaScript / TypeScript (Web / Node.js)
Using `jszip` for extraction.

```javascript
import JSZip from 'jszip';

async function fetchAndParseTranslation(translationId, pageNumber) {
    // 1. Fetch the zip file
    const url = `https://cdn.jsdelivr.net/gh/zaibihsn/translation-all@1.0/${translationId}.zip`;
    const response = await fetch(url);
    const blob = await response.blob();

    // 2. Unzip & extract specific page
    const zip = await JSZip.loadAsync(blob);
    const pageFile = zip.file(`pages/${pageNumber}.toon`);
    if (!pageFile) throw new Error("Page not found");
    
    const toonText = await pageFile.async("string");

    // 3. Parse .toon format
    const lines = toonText.trim().split('\n');
    const verses = [];
    
    // Skip the header (index 0)
    for (let i = 1; i < lines.length; i++) {
        const line = lines[i];
        
        // Match: chapter, verse, "translation", footnotes (optional)
        // Using a regex to properly handle commas inside quotes
        const match = line.match(/^(\d+),(\d+),"([\s\S]*?)"(?:,footnotes:(.*))?$/);
        
        if (match) {
            verses.push({
                chapter: parseInt(match[1]),
                verse: parseInt(match[2]),
                text: match[3],
                footnotes: match[4] || null
            });
        }
    }
    
    return verses;
}
```

### Dart / Flutter
Using the `archive` package and `http`.

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'package:archive/archive.dart';

Future<List<Map<String, dynamic>>> fetchAndParseTranslation(String translationId, int pageNumber) async {
  // 1. Fetch the zip file
  final url = Uri.parse("https://cdn.jsdelivr.net/gh/zaibihsn/translation-all@1.0/$translationId.zip");
  final response = await http.get(url);
  
  // 2. Unzip & extract specific page
  final archive = ZipDecoder().decodeBytes(response.bodyBytes);
  final file = archive.findFile('pages/$pageNumber.toon');
  
  if (file == null) throw Exception("Page not found");
  
  final toonText = utf8.decode(file.content as List<int>);
  
  // 3. Parse .toon format
  final lines = toonText.trim().split('\n');
  final verses = <Map<String, dynamic>>[];
  
  // Regex to extract chapter, verse, text, and optional footnotes
  final regex = RegExp(r'^(\d+),(\d+),"([\s\S]*?)"(?:,footnotes:(.*))?$');
  
  for (int i = 1; i < lines.length; i++) {
    final match = regex.firstMatch(lines[i]);
    if (match != null) {
      verses.add({
        'chapter': int.parse(match.group(1)!),
        'verse': int.parse(match.group(2)!),
        'text': match.group(3),
        'footnotes': match.group(4),
      });
    }
  }
  
  return verses;
}
```

### Python
Using the built-in `zipfile`, `io`, and `re` modules.

```python
import urllib.request
import zipfile
import io
import re

def fetch_and_parse_translation(translation_id, page_number):
    # 1. Fetch the zip file
    url = f"https://cdn.jsdelivr.net/gh/zaibihsn/translation-all@1.0/{translation_id}.zip"
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
    
    with urllib.request.urlopen(req) as response:
        zip_data = response.read()

    # 2. Unzip & extract specific page
    with zipfile.ZipFile(io.BytesIO(zip_data)) as z:
        with z.open(f"pages/{page_number}.toon") as f:
            toon_text = f.read().decode('utf-8')

    # 3. Parse .toon format
    lines = toon_text.strip().split('\n')
    verses = []
    
    regex = re.compile(r'^(\d+),(\d+),"([\s\S]*?)"(?:,footnotes:(.*))?$')
    
    for line in lines[1:]: # Skip header
        match = regex.match(line)
        if match:
            verses.append({
                'chapter': int(match.group(1)),
                'verse': int(match.group(2)),
                'text': match.group(3),
                'footnotes': match.group(4)
            })
            
    return verses
```

---

## 3. AI Code Generation Prompt

If you're using an AI Assistant (like ChatGPT, Claude, GitHub Copilot) to help you build an app that utilizes this repository, you can simply paste the prompt below into the chat to get instant boilerplate code tailored to your exact stack:

> **AI Prompt Template:**
>
> "Act as a senior software engineer. I am building a Quran application using `[INSERT YOUR TECH STACK HERE, e.g., React Native/Flutter/Next.js]`. 
> 
> I need to implement a feature to fetch and parse Quran translation data from a remote GitHub CDN repository (`https://cdn.jsdelivr.net/gh/zaibihsn/translation-all@1.0`). 
> 
> **Data Details:**
> 1. The translations are stored in `.zip` files named by their ID (e.g., `english-hilali-khan.zip`). I want to fetch this ZIP file and extract it in memory.
> 2. Inside the ZIP, there is a `pages/` directory containing 604 files named `1.toon` to `604.toon`.
> 3. A `.toon` file uses a custom CSV-like format. The first line is a header like `quran[6]{c,v,t,f}:`. The subsequent lines are rows formatted strictly as: `{chapter_number},{verse_number},"{translation_text}"` OR `{chapter_number},{verse_number},"{translation_text}",footnotes:{footnote_text}`.
>
> **Requirements**:
> - Write a robust, production-ready function to fetch the zip, unzip it in memory, extract a specific `pageNumber.toon`, and parse the contents into a clean array/list of objects/models containing `chapter`, `verse`, `text`, and optionally `footnotes`. 
> - Handle network errors and parsing exceptions gracefully.
> - Ensure regex or standard parsing safely handles commas that exist *inside* the quoted translation text.
> - Provide recommendations on caching the unzipped contents locally so it doesn't need to be downloaded multiple times."
