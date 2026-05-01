# 📋 Pedigree Drawing App — Handoff Özeti

## 1. Proje Amacı ve Kapsamı

**Pedigri Çizim Uygulaması** — genetik soyağacı çizmek için interaktif web uygulaması. Klinik genetik standartlarına uygun (ISCN-benzeri) sembollerle pedigri diyagramları üretir.

**Hedef kullanıcılar:** Genetik danışmanlar, tıp öğrencileri, klinisyen, aile hastalık takibi yapan bireyler.

**İki ana mod:**
- 🧬 **Nadir Hastalık** — Mendel kalıtımı, tek hastalık alanı (DeNovo, Akraba evliliği template'leri)
- 🩸 **Kanser Pedigresi** — Onkogenetik, çoklu kanser/bilateral/yaş takibi (BRCA, Lynch, Li-Fraumeni)

**Ana özellikler:**
- 5 hazır şablon (template lock'lu)
- Sembol araçlarıyla tıkla-yerleştir
- Çizgi-çek bağlantı (eş, ebeveyn-çocuk, kardeş)
- Sürükle-taşı + Y otomatik nesil snap
- Akıllı çizgi rotalama (multi-level lanes)
- Save/Load JSON, paylaşılabilir link, Export (SVG/PNG/PDF/CSV)
- TR/EN dil desteği
- Karanlık tema
- Undo/Redo, batch operations, history timeline
- 13 otomatik test (5 template + 8 layout invariant)

---

## 2. Teknik Kararlar

| Konu | Karar |
|------|-------|
| **Stack** | Vanilla HTML/CSS/JS (framework yok) |
| **Tek dosya** | Tüm kod `index.html` içinde (~5300 satır) |
| **Bağımlılıklar** | jsPDF (CDN üzerinden, sadece PDF export için) |
| **Render** | SVG (viewBox + 1:1 mapping) |
| **State** | Plain JS objects (`familyData = {members, partnerships, parentChildLinks, mode, ...}`) |
| **Persistence** | localStorage (autosave), JSON file, URL hash (paylaşım) |
| **i18n** | `data-i18n`, `data-i18n-title`, `data-i18n-placeholder` attribute sistemi + `t()` fonksiyonu |
| **Layout** | Sequential placement + multi-child centering + density-aware overlap resolution |
| **Routing** | Orthogonal (Manhattan), multi-level lanes for distant relationships |
| **Deploy** | GitHub Pages (`https://farmakeus.github.io/pedigree-app/`) |
| **Git** | `main` branch, direct push (single developer) |

**Reddedilen yaklaşımlar:**
- ❌ Barycenter refinement (center bias yarattı)
- ❌ Layout memory cache (stale position bugları)
- ❌ Edge crossing minimization sort (unstable comparator)
- ❌ PedigreeJS gibi external library (basit tutmak için)

---

## 3. Tamamlanan Modüller/Fonksiyonlar

### 🏗️ Çekirdek Mimari
- `familyData` state model
- `addMemberDirect()`, `addOrUpdateMember()`, `deleteMember()`
- `addPartnership()`, `addParentChild()`, `removePartnership()`, `removeParentChild()`
- Undo/redo stack (`saveState()`, `undo()`, `redo()`)
- Change history (`logChange()`, `updateHistoryPanel()`)

### 📐 Layout Engine
- `computeLayout()` — sequential + multi-child centering + overlap resolution
- `findIntermediateNodes()` — obstacle detection
- Multi-level lane assignment (greedy interval scheduling)
- `getEdgeAnchor()`, `topEdgeAnchor()` — şekil sınır anchor'ları
- Density-aware spacing (densityFactor 1.0-1.6)
- Single child auto-centering

### 🎨 Render Pipeline
- `renderPedigree()` — ana render
- Partnership lines (normal + top-routed + consanguineous + divorced)
- Parent-child lines (orthogonal, sibship line)
- Member nodes (square/circle/diamond + carrier dot + cancer sectors)
- Pregnancy symbols (P, SAB, TOP, SB)
- Adoption brackets, deceased slash, proband arrow
- Cancer multi-sector rendering
- Twin connectors (V-shape + monozygotic bar)
- Generation guide lines (place mode'da otomatik)

### 🛠️ Araç Sistemi
- Place tool (8 sembol türü)
- Connect tool (siyah + yeşil noktalar, tıkla-çek)
- Move tool (Y-snap)
- Eraser tool (toast undo)
- Auto-layout
- Grid toggle, consanguinity toggle
- Multi-select (Shift+click)
- Batch operations (delete, set affected/carrier/deceased)

### 💾 Veri Yönetimi
- `serializeFamilyData()` / `loadFamilyData()`
- `saveToFile()` / `loadFromFile()` (JSON)
- `tryLoadAutosave()` (localStorage)
- `generateShareLink()` (URL hash + base64)
- Export: SVG, PNG, PDF, CSV
- Template lock (`lockTemplatePositions()`, `currentLoadedTemplate`)
- Reset to template button

### 🎨 UI / UX
- Modal sistem (Help)
- Toast notifications (undo support)
- Custom UI tooltips (500ms delay, smart positioning)
- Form tooltips (klinik açıklamalar)
- Tutorial tour (6 adım, progress + back button)
- Karanlık tema toggle
- Mini-map
- Search/filter (member list)
- Stats panel (mode-aware)
- Validation panel (logical conflicts)
- Inline relationship toggle chips (akraba, boşanma)

### 🧪 Test Altyapısı
- `runTemplateTests()` — 5 template (member count, mode, lock, NaN check)
- `runLayoutTests()` — 8 invariant (single child centered, no overlap, valid pos, etc.)
- `runAllTests()` — master runner
- Console hint message at startup

---

## 4. Devam Eden İş — Şu An Nerede Kaldık?

### ✅ Son Tamamlanan
**Place mode auto-grid** (commit `f4b1428`) — Şekil ekleme modu açıkken **nesil kılavuz çizgileri** (I, II, III, IV, V Roma rakamlarıyla) otomatik olarak görünüyor. Mod kapanınca kayboluyor.

### 🎯 Kullanıcının Son Sorduğu
Kullanıcı bu özelliğin çalışıp çalışmadığını henüz **test edip onaylamadı**. Bir sonraki session başında:
1. Kullanıcıya `Cmd + Shift + R` ile cache temizleyip test ettirilmeli
2. Sonuç onaylandıktan sonra başka bir özelliğe geçilmeli

### 📋 Tamamlanan Son Faz Listesi (sırayla)
1. ✅ Help modal İngilizce çevirisi (50+ key)
2. ✅ Tutorial geliştirmesi (progress dots, back button, smart positioning, pulse animation)
3. ✅ Replay tutorial butonu (Help modal'da)
4. ✅ Place mode otomatik nesil kılavuz çizgileri

### 🤔 Sıradaki Olası İşler (kullanıcı yönlendirsin)
- 📸 Üye fotoğrafları (drag-drop)
- 🩻 ART/IVF göstergeleri (sperm donörü, taşıyıcı anne)
- 👯 Üvey kardeş gösterimi (yarı bağlantı çizgisi)
- 🖨️ Yazdırma modu / print preview
- 📋 Daha fazla şablon (X-linked, Mitokondriyal)
- 🧪 Genetik test sonuçları (her bireye)
- 📱 Mobile/tablet optimizasyonu

---

## 5. Bilinen Sorunlar / Çözülmemiş Kararlar

### ⚠️ Bilinen Limitasyonlar (acil değil)
1. **Kompleks kuzen evliliği template'lerinde line crossing** — BRCA gibi templates'te 2 couple'ın siblings olduğu durumlarda çizgiler kesişebilir. Sort comparator'lı crossing minimization denendi ama unstable çıktı, geri çekildi.
2. **Help modal mobile uyumlu değil** — küçük ekranlarda dağılabilir.
3. **Tutorial pulse animation Safari'de yavaş** olabilir (test edilmedi).
4. **Çok büyük pedigriler (50+ üye)** test edilmedi, performans bilinmiyor.
5. **PDF export print kalitesi** — basit raster export, vector quality değil.

### 🔧 Geri Çekilen / Sandbox'ta Olan Özellikler
- ❌ **Layout memory** — stale position riski (commit `300ab61` ile geri çekildi)
- ❌ **Edge crossing minimization** — sort instability (aynı commit)

### 💡 Mimari Borçlar
1. **5300+ satır tek dosyada** — modülerizasyon önerildi ama reddedildi (basit deploy için)
2. **No CSS preprocessor** — değişkenler manuel yönetiliyor
3. **No build step** — direct edit + commit + GitHub Pages
4. **No proper test framework** — console-based test runner var ama otomatize değil
5. **Dosya boyutu büyük** (~250KB HTML) — GitHub Pages ilk yüklemede biraz yavaş

### 🚧 Yarım Kalmış Düşünceler
- Mobile/tablet optimization önerildi, henüz yapılmadı
- Print preview önerildi, yapılmadı
- "Daha fazla şablon" önerildi, yapılmadı

---

## 6. Önemli Dosya Yolları ve Yapı

### Dizin Yapısı
```
/Users/ozanvural/claude code/pedigree-app/
├── index.html              ← TEK ANA DOSYA (~5300 satır)
├── HANDOFF.md              ← Bu dosya
└── .git/                   ← Git repo
```

### `index.html` İç Yapı (Bölümler)
```
1.    <head>
      ├─ <style> CSS (~470 satır)
      │   ├─ App layout (header, panels)
      │   ├─ Form styles
      │   ├─ Toolbar styles (.place-toolbar, .zoom-controls)
      │   ├─ Member list, stats, validation, history panels
      │   ├─ Tooltips (custom + pedigree tooltip)
      │   ├─ Help modal, tour tooltip, batch toolbar, toast
      │   ├─ Dark theme overrides
      │   └─ Mode toggle, cancer section, autocomplete
      └─ jspdf CDN

2.    <body>
      ├─ App header (mode toggle + actions)
      ├─ Main container
      │   ├─ Left panel (3 tabs: Üye Ekle, Aile Listesi, İlişkiler)
      │   └─ Right panel (canvas + toolbars + minimap)
      ├─ Legend overlay
      ├─ Pedigree tooltip (hover)
      ├─ UI tooltip (custom)
      ├─ Toast container
      ├─ Help modal
      └─ Tour tooltip

3.    <script> JS (~3700 satır)
      ├─ Constants (DISEASE_COLORS, CANCER_TYPES, CANCER_COLORS)
      ├─ State (familyData, lastLayout, currentLang, etc.)
      ├─ Member management (add/edit/delete)
      ├─ Disease + Cancer management
      ├─ Mode toggle (rare ↔ cancer)
      ├─ Relationships (partnerships, parent-child)
      ├─ UI updates (lists, panels, legend, stats, validation, history)
      ├─ Pedigree layout engine (computeLayout)
      ├─ Render pipeline (renderPedigree)
      ├─ Export (SVG, PNG, PDF, CSV)
      ├─ Template system (lock, reset, 5 templates)
      ├─ Sample data / cancer templates (BRCA, Lynch, Li-Fraumeni)
      ├─ I18N system (TR + EN ~250 keys)
      ├─ Toast notifications
      ├─ Custom tooltips (initUITooltips)
      ├─ Mini-map
      ├─ Save/Load/Share/Autosave
      ├─ Undo/Redo
      ├─ Tools (place, connect, move, eraser)
      ├─ Multi-select / batch operations
      ├─ Keyboard shortcuts
      ├─ Tutorial tour
      ├─ Test suites (runTemplateTests, runLayoutTests, runAllTests)
      └─ INIT (theme, lang, autosave load, hash load, render)
```

### Anahtar Fonksiyonlar (hızlı bulma)
| Fonksiyon | Yaklaşık Satır | Görev |
|-----------|---------------|-------|
| `computeLayout()` | ~2680 | Pozisyonları hesaplar |
| `renderPedigree()` | ~2860 | SVG'yi render eder |
| `addOrUpdateMember()` | ~1065 | Üye ekle/güncelle |
| `lockTemplatePositions()` | ~3715 | Template pozisyonlarını kilitler |
| `setPlaceMode()` | ~4297 | Place tool aktif et |
| `getEdgeAnchor()` | ~2950 | Şekil sınır anchor noktası |
| `findIntermediateNodes()` | ~2996 | Obstacle detection |
| Multi-level lane assignment | ~3018 | Top routing lane'leri |
| `t()` | i18n helper | Çeviri lookup |
| `runAllTests()` | ~5160 | Tüm testleri çalıştır |

### LocalStorage Anahtarları
```
pedigreeApp_data_v1      → Autosave verisi (JSON)
pedigreeApp_theme        → 'dark' | 'light'
pedigreeApp_lang         → 'tr' | 'en'
pedigreeApp_tourSeen_v1  → '1' (tour gösterildi mi)
```

### URL Hash Format (Paylaşım)
```
https://farmakeus.github.io/pedigree-app/#data=<base64-encoded-json>
```

### Git Commit Pattern
Çoğu commit:
```
[Konu] [kısa özet]

[Detay açıklama]

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
```

### Repo
- **Owner:** Farmakeus
- **Repo:** `Farmakeus/pedigree-app`
- **Branch:** `main`
- **Live URL:** https://farmakeus.github.io/pedigree-app/
- **Latest commit (handoff zamanı):** `f4b1428` (Place mode auto-grid)

---

## 🎬 Yeni Session Başlangıcı İçin Hızlı Komut

```bash
cd "/Users/ozanvural/claude code/pedigree-app"
git log --oneline -10                          # Son 10 commit
/opt/homebrew/bin/node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script>([\\s\\S]*?)<\\/script>/);
if (match) { try { new Function(match[1]); console.log('JS syntax OK'); } catch(e) { console.log('JS syntax error:', e.message); } }
"                                              # JS syntax check
```

Tarayıcıda konsolda test:
```javascript
runAllTests()  // 13/13 yeşil olmalı
```

**Önemli notlar:**
- Kullanıcı **Türkçe** konuşur, tooltip ve UI mesajları TR/EN her ikisinde de var
- Cache temizleme zorunlu (`Cmd + Shift + R`)
- Her commit `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>` ile imzalanır
- Push otomatik (single developer, no PR review)
- Test öncelikli: değişiklik sonrası `runAllTests()` çağrılmalı
