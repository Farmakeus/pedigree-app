# 📋 Pedigree Drawing App — Handoff Özeti

> **Önemli:** Bu dosya her session sonunda güncellenir. Yeni session'a başlarken **önce bunu oku**, sonra `git log --oneline -20` ile son commit'leri gözden geçir.

## 1. Proje Amacı ve Kapsamı

**Pedigri Çizim Uygulaması** — genetik soyağacı çizmek için interaktif web uygulaması. Klinik genetik standartlarına uygun (ISCN-benzeri) sembollerle pedigri diyagramları üretir.

**Hedef kullanıcılar:** Genetik danışmanlar, tıp öğrencileri, klinisyen, aile hastalık takibi yapan bireyler.

**İki ana mod:**
- 🧬 **Nadir Hastalık** — Mendel kalıtımı, tek hastalık alanı (DeNovo, Akraba evliliği template'leri)
- 🩸 **Kanser Pedigresi** — Onkogenetik; **şu an varsayılan: minimal Otozomal Dominant** template (sadece proband detaylanır, kalan üyelerde klinik veri yok)

**Ana özellikler:**
- 3 hazır şablon görünür (DeNovo, Akraba evliliği, ⭐ AD Kanser); BRCA/Lynch/LiFraumeni teaching template'leri kodda var ama UI'dan kaldırıldı
- Sembol araçlarıyla tıkla-yerleştir (place tool, hover preview ile hedef nesil gösterimi)
- Çizgi-çek bağlantı (eş, ebeveyn-çocuk, kardeş)
- **Konsanguinite tek-tıkla doğrudan** (Cmd+Shift drag veya consanBtn aktifken connect)
- Sürükle-taşı + Y otomatik nesil snap
- Akıllı çizgi rotalama (multi-level lanes)
- **Family-cluster aware spacing**: bağımsız aileler arası ekstra padding
- Save/Load JSON, paylaşılabilir link, Export (SVG/PNG/PDF/CSV) — proband-aware filename
- TR/EN dil desteği, karanlık tema, undo/redo, history timeline
- 13 otomatik test (5 template + 8 layout invariant)

---

## 2. Teknik Kararlar

| Konu | Karar |
|------|-------|
| **Stack** | Vanilla HTML/CSS/JS (framework yok) |
| **Tek dosya** | Tüm kod `index.html` içinde (~7000 satır) |
| **Bağımlılıklar** | jsPDF (CDN), sadece PDF export için |
| **Render** | SVG (viewBox + 1:1 mapping) |
| **State** | Plain JS objects (`familyData = {members, partnerships, parentChildLinks, mode, title, ...}`) |
| **Single source of truth** | `familyData` — popup / advanced panel / canvas / autosave hepsi aynı obje üzerinden |
| **Persistence** | localStorage (autosave + theme + lang + labelMode + leftPanelCollapsed), JSON file, URL hash |
| **i18n** | `data-i18n*` attribute sistemi + `t()` helper |
| **Layout** | Sequential placement + multi-child centering + density-aware overlap + cluster padding |
| **Routing** | Orthogonal (Manhattan), multi-level lanes |
| **Robustness** | `safeRender()` wrapper try/catch'le render hatalarını yutar; left panel her zaman editable kalır |
| **Deploy** | GitHub Pages (`https://farmakeus.github.io/pedigree-app/`) |
| **Git** | `main` branch, direct push (single dev) |

**Reddedilen yaklaşımlar:**
- ❌ Barycenter refinement (center bias)
- ❌ Layout memory cache (stale position bugları)
- ❌ Edge crossing minimization sort (unstable comparator)

---

## 3. Tamamlanan Modüller (Bu session ekledikleri **kalın**)

### 🏗️ Çekirdek Mimari
- `familyData` state model (artık `title` + sync alanları içerir)
- `addMemberDirect()`, `addOrUpdateMember()`, `deleteMember()`
- **`bulkAddSiblings()`** — N kardeş tek seferde
- **`normalizeGenerationsBelowOne()`** — gen<1 inserted above ise tüm üyeleri shift
- Undo/redo, change history, autosave

### 📐 Layout Engine
- `computeLayout()` — sequential + multi-child centering + overlap resolution
- **Family-cluster awareness**: connected-component graph; farklı aileler arası `INTER_CLUSTER_PAD = 120px` ekstra
- **Y formülü gen numarasından**: `genY = MARGIN + (gen-1) * (NODE_H + V_GAP) + NODE_H/2` (eskiden genIdx kullanılıyordu — bug)
- **customY drift refresh**: `|customY - canonicalGenY| > V_GAP/2` ise resnap
- `findIntermediateNodes()`, multi-level lane assignment
- Density-aware spacing (densityFactor 1.0-1.6)
- **Geniş kardeş grupları için child gap**: 80/60/45/35/30 (n ≤ 2/4/6/8/9+)

### 🎨 Render Pipeline
- `renderPedigree()` — ana render
- **`safeRender()`** wrapper — try/catch + fallback toast
- Partnership lines (normal + top-routed + consanguineous + divorced)
- **Twin V-junction integrated with sibship line** (önceden çift vertical bug'ı vardı)
- Member nodes (square/circle/diamond + carrier dot + cancer sectors)
- Pregnancy symbols, adoption brackets, deceased slash, proband arrow
- Cancer multi-sector rendering
- **Gen row labels clickable** (selectGenerationRow ile target gen seçimi, mavi highlight band)
- Generation guide lines (place mode'da otomatik, member-less satırlarda label gösterir — duplicate önleyici)
- **Pedigree title** SVG'nin üstünde Georgia serif

### 🛠️ Araç Sistemi
- Place / Connect / Move / Eraser
- **Place mode hover preview**: gri band + "II'e ekle" / turuncu band + "↑ Yeni nesil eklenecek"
- Auto-layout, grid toggle, consanguinity toggle
- Multi-select (Shift+click), batch ops

### 🆕 Quick-Edit Popup (Bu session)
- **Sol tık node → küçük popup**: Name / Disease / Variant
- Real-time write-through to `familyData` + advanced panel sync
- "⚙ Detaylı Düzenle" butonu — advanced panel'e geçiş
- ESC / outside click / başka node'a tıklama → kapanır

### 🆕 Right-click Status Menu (Bu session)
- **Sağ tık node → context menu**: affected/carrier/deceased/proband (exclusivity), sex (M/F/U), pregnancy (P/SAB/clear)
- ✓ check ile güncel durum, advanced panel checkboxes anında senkron

### 🆕 Sol Panel Collapsible (Bu session)
- **`<<` / `>>` butonu**: parlak mavi, panel sağ kenarına yapışık (kapalıyken sol kenarda)
- localStorage `pedigreeApp_leftPanelCollapsed` ile state persist
- 250ms smooth slide animasyonu

### 💾 Veri Yönetimi
- `serializeFamilyData()` / `loadFamilyData()` (artık `title` da serialize ediliyor)
- Smart export filename: `<title|proband>_pedigree.<ext>`, Türkçe diakritik temizliği
- Optional "İndirmeden önce dosya adını sor" toggle export menüsünde

### 🎨 UI / UX
- **Tab adı** "Üye Ekle" → "**Ekle / Düzenle**"
- **Pedigree Title input bar** canvas üstünde
- **Generation toggle dropdown** dinamik (max+1 her zaman mevcut)
- **Shift-down toast** (üst nesil eklendiğinde "Tüm nesiller bir aşağı kaydırıldı")
- **Toplu Kardeş Ekle** UI (İlişkiler tab): sayı 1-20, cinsiyet (M/F/U/Karışık)
- **Live disease/variant sync**: oninput ile real-time render
- **Variant strict + rule**: " +" suffix daima
- **Auto labels OFF default**: "Birey N" placeholder'ları render edilmez
- **Label mode toggle** (Aa butonu): Off / On / Proband-only (3-state cycle)
- Modal sistem (Help), toast notifications, custom UI tooltips, tutorial tour
- Karanlık tema, mini-map, search/filter, stats panel, validation panel
- **Cancel/empty-canvas-click** → editingMemberId + focusedMemberId temizlenir

### 🧪 Test Altyapısı
- `runTemplateTests()` — 5 template (member count, mode, lock, NaN check)
- `runLayoutTests()` — 8 invariant
- `runAllTests()` — master runner
- **13/13 hala yeşil** her commit sonrası

---

## 4. Devam Eden İş — Şu An Nerede Kaldık?

### ✅ Son Tamamlanan (commit `204cde6`)
**Quick-edit popup + Right-click status menu** — node click workflow'u kökten değişti:
- Sol tık → küçük popup (Name/Disease/Variant) cursor yanında
- Sağ tık → status flag context menu
- Eski "click → advanced panel" davranışı yerine, popup içindeki "⚙ Detaylı Düzenle" butonu advanced panel'i açıyor
- Bütün UI'lar (popup, advanced panel, canvas, autosave) aynı `familyData` üstünde — single source of truth

### 🎯 Kullanıcının Son Konuştuğu
Context dolmak üzere; **yeni session'a geçiş** istedi. Test/onay yok henüz — yeni session'da kullanıcı şunları test edebilir:
1. Sol tık node → popup açılıyor mu?
2. Sağ tık → context menu çıkıyor mu?
3. Popup'taki yazılar advanced panel'e ve canvas'a anında yansıyor mu?
4. Sol panel collapsible toggle butonu (mavi `<<`) görünür mü?

### 📋 Son Faz Özeti (sırayla, bu session'da)
1. ✅ Pedigree title + smart export naming
2. ✅ Auto label OFF default (clinical clean view) + Aa toggle
3. ✅ Single-click node → editor (sonra popup'a evrildi)
4. ✅ Generation flexibility (any gen any time, dynamic dropdown, shift-down toast, hover preview)
5. ✅ Layout Y formülü gen# bazlı, customY drift refresh, duplicate gen label fix, selectable gen rows
6. ✅ Live disease/variant sync + variant " +" rule + safeRender resilience
7. ✅ Family-cluster spacing (auto-layout artık aileleri ayırıyor)
8. ✅ Twin V-junction fix + bulk-add siblings + large-sibship spacing
9. ✅ Sol panel collapsible (`<<` / `>>` toggle)
10. ✅ Konsanguinite direct action (eski 2-step de korundu)
11. ✅ Minimal AD cancer template default (BRCA/Lynch/LiFr UI'dan kaldırıldı)
12. ✅ Quick-edit popup + Right-click status menu

### 🤔 Sıradaki Olası İşler (kullanıcı yönlendirsin)
- 📸 Üye fotoğrafları (drag-drop)
- 🩻 ART/IVF göstergeleri (sperm donörü, taşıyıcı anne)
- 👯 Üvey kardeş gösterimi (yarı bağlantı çizgisi)
- 🖨️ Yazdırma modu / print preview
- 🧪 Genetik test sonuçları detaylı alanı (test tarihi, panel adı, VUS işareti, ACMG sınıflandırması)
- 📋 Daha fazla şablon (X-linked, Mitokondriyal, Yarı kardeş)
- 📱 Mobile/tablet optimizasyonu
- 📄 Aile öyküsü auto-generated klinik özet metni

---

## 5. Bilinen Sorunlar / Çözülmemiş Kararlar

### ⚠️ Bilinen Limitasyonlar (acil değil)
1. **Kompleks kuzen evliliği template'lerinde line crossing** — sort comparator unstable.
2. **Help modal mobile uyumlu değil**.
3. **Çok büyük pedigriler (50+ üye)** test edilmedi.
4. **PDF export raster kalitesinde** — vector değil.
5. **safeRender error toastları** birikebilir (henüz gerçek crash görmedik ama diagnostik için duruyor).

### 🔧 Geri Çekilen / Sandbox'ta Olan
- ❌ **Layout memory** — stale position riski
- ❌ **Edge crossing minimization** — sort instability
- ❌ **BRCA/Lynch/LiFraumeni template UI buttons** — clutter (functions hâlâ test için kodda)

### 💡 Mimari Borçlar
1. **7000+ satır tek dosyada** — modülerizasyon önerildi ama reddedildi (basit deploy için)
2. **No CSS preprocessor**, no build step
3. **Console-based test runner** otomatize değil
4. Dosya boyutu büyük (~330KB HTML) — GitHub Pages ilk yüklemede biraz yavaş

---

## 6. Önemli Dosya Yolları ve Yapı

### Dizin Yapısı
```
/Users/ozanvural/claude code/pedigree-app/
├── index.html              ← TEK ANA DOSYA (~7000 satır)
├── HANDOFF.md              ← Bu dosya
├── .claude/launch.json     ← Preview server config (port 5175, python http.server)
└── .git/
```

### `index.html` Bölümleri
```
1.    <head>
      ├─ <style> CSS (~700 satır)
      │   ├─ Layout, panels, toolbars
      │   ├─ Pedigree title bar, label-toggle button
      │   ├─ Quick-edit popup (.quick-edit-popup), context menu (.status-context-menu)
      │   ├─ Panel toggle button (.panel-toggle)
      │   ├─ Tooltips, modal, tour, batch toolbar, toast
      │   └─ Dark theme overrides
      └─ jspdf CDN

2.    <body>
      ├─ App header (mode toggle + actions + title input)
      ├─ Main container
      │   ├─ Panel toggle button (<<>>)
      │   ├─ Left panel (3 tabs: Ekle/Düzenle, Aile Listesi, İlişkiler)
      │   └─ Right panel (canvas + toolbars + minimap + zoom + place-toolbar)
      ├─ Legend overlay
      ├─ Pedigree tooltip (hover)
      ├─ UI tooltip (custom)
      ├─ Quick-edit popup (#quickEditPopup)        ← yeni
      ├─ Status context menu (#statusContextMenu)  ← yeni
      ├─ Toast container, Help modal, Tour tooltip

3.    <script> JS (~5500 satır)
      ├─ Constants (DISEASE_COLORS, CANCER_TYPES, CANCER_COLORS)
      ├─ State (familyData, lastLayout, currentLang, selectedGeneration, ...)
      ├─ Member management (add/edit/delete)
      ├─ bulkAddSiblings()
      ├─ Mode toggle (rare ↔ cancer)
      ├─ Relationships
      ├─ UI updates (lists, panels, legend, stats, validation, history)
      ├─ Pedigree layout engine (computeLayout, family clusters, drift refresh)
      ├─ Render pipeline (renderPedigree, safeRender, twin V-junction)
      ├─ Quick edit popup + Status context menu
      ├─ Generation row click handling (selectGenerationRow)
      ├─ Place mode hover preview (renderPlacePreview)
      ├─ Export (SVG, PNG, PDF, CSV, JSON) — smart filename
      ├─ Template system (lockTemplatePositions, currentLoadedTemplate)
      ├─ Sample data (DeNovo, Consang, AD Cancer, BRCA/Lynch/LiFr internal)
      ├─ I18N (TR + EN ~330 keys)
      ├─ Toast notifications, mini-map, custom tooltips
      ├─ Save/Load/Share/Autosave
      ├─ Undo/Redo
      ├─ Tools (place, connect, move, eraser) + touch handlers
      ├─ Multi-select / batch operations
      ├─ Keyboard shortcuts
      ├─ Tutorial tour
      ├─ Left panel collapse toggle (toggleLeftPanel)
      ├─ Test suites (runTemplateTests, runLayoutTests, runAllTests)
      └─ INIT (theme, lang, label mode, panel collapse, autosave load, hash load, render)
```

### LocalStorage Anahtarları
```
pedigreeApp_data_v1            → Autosave verisi (JSON)
pedigreeApp_theme              → 'dark' | 'light'
pedigreeApp_lang               → 'tr' | 'en'
pedigreeApp_tourSeen_v1        → '1'
pedigreeApp_labelMode          → 'off' | 'on' | 'proband'
pedigreeApp_leftPanelCollapsed → '0' | '1'
```

### URL Hash Format (Paylaşım)
```
https://farmakeus.github.io/pedigree-app/#data=<base64-encoded-json>
```

### Repo
- **Owner:** Farmakeus
- **Repo:** `Farmakeus/pedigree-app`
- **Branch:** `main`
- **Live URL:** https://farmakeus.github.io/pedigree-app/
- **Latest commit (handoff zamanı):** `204cde6` (Quick-edit popup + right-click status menu)

---

## 🎬 Yeni Session Başlangıcı İçin Hızlı Komut

```bash
cd "/Users/ozanvural/claude code/pedigree-app"
git log --oneline -20
/opt/homebrew/bin/node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const m = html.match(/<script>([\\s\\S]*?)<\\/script>/);
if (m) { try { new Function(m[1]); console.log('JS syntax OK'); } catch(e) { console.log('ERR:', e.message); } }
"
```

Tarayıcıda console'da:
```javascript
runAllTests()  // 13/13 yeşil olmalı
```

**Önemli notlar:**
- Kullanıcı **Türkçe** konuşur. UI çoğunlukla TR; EN tam çeviri var.
- Test sonrası **`Cmd + Shift + R`** ile cache temizletmek zorunlu.
- Her commit `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>` ile imzalanır.
- Push otomatik (single dev, no PR review).
- Değişiklik sonrası `runAllTests()` çağrılmalı — commit etmeden önce.
- Preview server: `mcp__Claude_Preview__preview_start` "Pedigree App" → port 5175 (Python http.server).
- **Single source of truth: `familyData`** — popup, advanced panel, canvas, autosave hepsi aynı obje üzerinde çalışmalı; yeni feature eklerken bu kuralı bozmamak kritik.
- **safeRender()** kullan — direct `renderPedigree()` çağrıları artık fault-tolerant değil; render hatası UI'yi bozmamalı.
