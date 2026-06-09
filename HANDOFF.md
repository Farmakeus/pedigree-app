# 📋 Pedigree Drawing App — Handoff Özeti

> **Önemli:** Bu dosya her session sonunda güncellenir. Yeni session'a başlarken **önce bunu oku**, sonra `git log --oneline -20` ile son commit'leri gözden geçir.

## 1. Proje Amacı ve Kapsamı

**Pedigri Çizim Uygulaması** — genetik soyağacı çizmek için interaktif web uygulaması. Klinik genetik standartlarına uygun (ISCN-benzeri) sembollerle pedigri diyagramları üretir.

**Hedef kullanıcılar:** Genetik danışmanlar, tıp öğrencileri, klinisyen, aile hastalık takibi yapan bireyler. (Geliştiren: tıp doktoru, yarın hocaya demo gösterecek.)

**İki ana mod:**
- 🧬 **Nadir Hastalık** — Mendel kalıtımı, tek hastalık alanı (DeNovo, Akraba evliliği template'leri)
- 🩸 **Kanser Pedigresi** — Onkogenetik; varsayılan minimal Otozomal Dominant template

**Ana özellikler:**
- 3 hazır şablon görünür (DeNovo, Akraba evliliği, ⭐ AD Kanser); BRCA/Lynch/LiFraumeni teaching template'leri kodda var ama UI'dan kaldırıldı
- Sembol araçlarıyla tıkla-yerleştir (place tool, hover preview)
- Çizgi-çek bağlantı (eş, ebeveyn-çocuk, kardeş) + konsanguinite tek-tıkla
- **Multi-select** (Shift/Ctrl/Cmd+click, drag rubber-band, Ctrl+A) + **grup taşıma**
- **Adaptive phenotype segments**: 1=tam dolu, 2=yarım, 3+=çeyrek/sektör (affected şartı YOK, hastalık girilince renkleniyor)
- Save/Load JSON, paylaşılabilir link, Export (SVG/PNG/PDF/CSV)
- TR/EN, karanlık tema, undo/redo, history timeline
- **38 otomatik test** (5 template + 8 layout + 25 regression)

---

## 2. Teknik Kararlar

| Konu | Karar |
|------|-------|
| **Stack** | Vanilla HTML/CSS/JS (framework yok) |
| **Tek dosya** | Tüm kod `index.html` içinde (~8200 satır) |
| **Bağımlılıklar** | jsPDF (CDN), sadece PDF export için |
| **Render** | SVG (viewBox + 1:1 mapping) |
| **State** | Plain JS objects (`familyData = {members, partnerships, parentChildLinks, mode, title, diseases, ...}`) |
| **Single source of truth** | `familyData` — popup / advanced panel / canvas / autosave hepsi aynı obje üzerinden |
| **Persistence** | localStorage (autosave + theme + lang + labelMode + leftPanelCollapsed + legendVisible + textScale), JSON file, URL hash |
| **i18n** | `data-i18n*` attribute + `t()`; `applyTranslations()` `textContent` yazıyor (⚠️ ikon+metin gereken yerlerde metin AYRI `<span>`'de olmalı, yoksa SVG silinir) |
| **Layout** | Sequential placement + multi-child symmetric centering + density-aware overlap + cluster padding |
| **Robustness** | `safeRender()` try/catch'le render hatalarını yutar |
| **Deploy** | GitHub Pages (`https://farmakeus.github.io/pedigree-app/`) |
| **Git** | `main` branch, direct push (single dev) |
| **Preview** | `.claude/launch.json` → "Pedigree App", port 5175, `python3 -m http.server` |

**Reddedilen yaklaşımlar:** Barycenter refinement, layout memory cache, edge crossing minimization sort (hepsi instability/stale bug nedeniyle).

---

## 3. Bu Session'da Yapılanlar (commit `856c771` ÜZERİNE — hepsi tek session)

### 🆕 Etkileşim & Navigasyon
- **Multi-select**: Shift+click, Ctrl/Cmd+click toggle, **drag rubber-band kutu** (`#rubberBand`), Ctrl+A select-all
- **Grup taşıma**: Move aracı + çoklu seçim → birini sürükle, hepsi birlikte kayar (ilişkiler korunur). Capture-phase handler + **`stopImmediatePropagation`** (kritik: aynı element üstündeki single-move handler'ı durdurmak için)
- **Klavye pan**: Ok tuşları (40px), Shift+ok (120px)
- **Wheel zoom**: Ctrl/Cmd+wheel cursor-anchored; trackpad pinch otomatik
- **Spacebar+drag pan**, **A−/A+ text scale** (0.6–2.2, post-render `<text>` ölçekleme)
- Shortcuts: Delete (multi-select aware), Ctrl+Z/Y/A

### 🆕 Sibling Symmetry (klinik standart)
- `computeLayout` içinde: `pos[i] = parentCenterX + (i − (n−1)/2) * stride`
- Tek n → orta çocuk merkez çizgide; çift n → orta çiftin arası merkezde
- `stride = max(NODE_W+childGap, density*baseCenterDist)` → overlap pass simetriyi bozmuyor
- customX artık çoklu çocukta override ediliyor (eski bug: template customX simetriyi kaydırıyordu)

### 🆕 Twin sibship termination fix
- Sibship çizgisi ikiz V-junction X'inde bitiyor (eskiden dış ikizin X'ine taşıyordu)
- `groupAttachX` = single→child.x, twin→V-junction midpoint

### 🆕 Phenotype görselleştirme
- Segmentler artık **`m.affected` şartı OLMADAN** çiziliyor (hastalık girilince renklenir)
- `hasPhenoSegments` varsa solid siyah dolgu atlanıyor (segment renkleri temiz kalsın)
- `renderPhenotypeSegments(pos, shape, r, items, opacity)`: 1=tam, 2=yarım, 3=3 sektör, 4=çeyrek, ≥5=+N overflow badge

### 🆕 Editing workflow
- **Advanced-edit panel auto-open YALNIZCA**: popup'taki "Detaylı Düzenle" butonu (`openAdvancedFromPopup`) veya manuel toggle. Düz node seçimi paneli AÇMAZ.
- **Internal ID** (`Birey N`/`Individual N`): name input'unun İÇİNDE sağda silik suffix (`#memberInternalIdSuffix`), ekstra satır yok, canvas'a yansımaz
- **Conditional disease display**: legend yalnızca kullanıcının eklediği hastalıkları gösterir (`familyData.diseases` + üyelerdeki canlı + cancers). Node altı etiket sadece veri girilince.
- **Bidirectional sync**: popup ↔ advanced panel ↔ legend (`onQuickEditField` → `updateDiseaseListUI` + `updateLegend`)

### 🆕 Connection line right-click menu (`#connContextMenu`)
- Çizgiye sağ tık → bağlam menü (8px sıkı hit-test, node sağ tıkını çalmaz)
- **Evlilik çizgisi**: Boşanma ✓ / Akraba evliliği ✓ toggle + Bağlantıyı kaldır
- **Ebeveyn-çocuk**: Bağı kaldır
- `findConnectionAtPos(pos)` (`_pointSegDist` helper), `openConnectionContextMenu`, `connMenuAction`
- Mevcut `togglePartnershipFlag`/`removePartnership`/`removeParentChild` kullanır (undo destekli)

### 🎨 Görsel Polish (12 madde)
1. **Header emoji → Lucide-style SVG ikon** (kaydet/yükle/paylaş/export/dil/tema/yardım/temizle). i18n stringlerinden eski emojiler temizlendi (TR+EN). Mod butonlarında ikon YOK — sadece sol üstte DNA logosu.
2. **Header sadeleştirme**: flat `#1f2a37` (gradient kaldırıldı) + ince border. Dark: `#141821`.
3. **Canvas dot-grid** zemin (`background-attachment:local`), dark için ayrı.
4. **Tipografi**: `-apple-system` font stack, font-smoothing; sol panel label'ları yumuşak gri + nefes alanı; input focus glow.
5. **Node derinliği**: `.pnode` class + CSS `drop-shadow` (çizgiler `<line>` olduğu için etkilenmez). Dark için ayrı.
6. **Yumuşak hastalık renkleri**: disease `#b23a48` (bordo), variant `#5b4b8a` (indigo) — `.lbl-disease/.lbl-variant/.lbl-cancer/.lbl-name` class'ları.
7. **Radius sistemi**: input/buton 8px, panel/kart 12px, chip 6px.
8. **Boş canvas karşılama**: soluk mini-pedigree illüstrasyonu + "Pedigri çizmeye başlayın" (TR/EN).
9. **Tutarlı geçişler**: tema fade (0.3s) major yüzeylerde + body.
10. **Dark tema kontrast fix**: genel `text{fill:#ddd}` kuralı `:not(.lbl-cancer):not(.lbl-disease):not(.lbl-variant)` ile sınırlandı; etiketler parlak renk korur (`#f0909c`, `#bca8ee`).
11. Legend → sağ toolbar'ın ALTINDA bağımsız floating card (`#legendPanel`), collapsible (`toggleLegendPanel`), **simetrik CSS grid** (ikon | label), `positionLegendPanel()` toolbar altına dinamik konumlar. Default kapalı.
12. **SVG export black-box fix**: `fill="transparent"` → `fill="none"` (bazı viewer transparent'ı siyah render ediyordu); export'a xmlns + beyaz bg rect + sanitize.

### ↩️ Geri Alınanlar
- **Yarım kardeş (`//` kırık sibship)** — eklendi sonra kullanıcı isteğiyle TAMAMEN geri alındı. (Veri modeli paylaşılan ebeveynli iki evliliği zaten destekliyor; sadece otomatik `//` notasyonu yok.)

---

## 4. Devam Eden İş — Şu An Nerede Kaldık?

### ✅ Son durum
- Tüm yukarıdaki iş **tamamlandı ve test edildi** (38/38 yeşil, preview'da TR+EN+dark doğrulandı).
- **⚠️ Henüz commit edilmedi** — bu handoff commit'i ile birlikte commit'lenecek. Yeni session `git log` ile görmeli.

### 🎯 Kullanıcının Son Konuştuğu
"Handoff oluştur, yeni chatte devam edeceğim." Yarın hocaya demo var.

### 🤔 Sıradaki Olası İşler (kullanıcı yönlendirsin)
Daha önce önerilen, henüz yapılmamış yüksek-değer işler:
1. **Otomatik aile öyküsü özeti** — pedigreeden klinik metin üretimi (TR/EN), klinik nota kopyalanır. (En yüksek demo etkisi, az kod.)
2. **Yazdırma/print modu** — temiz print stylesheet (paneller gizli, pedigree+başlık+lejand+tarih), Ctrl+P.
3. **Genetik test detay alanı** — ACMG sınıfı (Patojenik/VUS/Benign), zigosite, test tarihi/panel, VUS işareti.
4. **Validation motoru** — obligat taşıyıcı tespiti, kalıtım modu çakışması, yaş tutarsızlığı.
5. **Anonimleştirme modu** — tek tıkla isimleri kod ile değiştir (II-1, III-2). KVKK/yayın için.
6. ART/IVF göstergeleri, otomatik kaydetme göstergesi, klavye cheat-sheet modal.

---

## 5. Bilinen Sorunlar / Limitasyonlar

1. **PDF export raster kalitesinde** (vector değil).
2. **Help modal mobile uyumlu değil**; genel mobile/tablet optimize değil.
3. **Çok büyük pedigriler (50+ üye)** test edilmedi — demoyu 8-15 üye tut.
4. **Kompleks kuzen evliliği** template'lerinde nadir line crossing (sort comparator unstable — minimize edilmiş değil).
5. Export dropdown İÇİ menü öğeleri (SVG/PNG/PDF/CSV) hâlâ emoji'li (header butonları temizlendi; bunlar çift sembol oluşturmadığı için bırakıldı).
6. Dosya boyutu ~390KB HTML — GitHub Pages ilk yükleme biraz yavaş.

---

## 6. Önemli Dosya Yolları ve Yapı

### Dizin
```
/Users/ozanvural/claude code/pedigree-app/
├── index.html              ← TEK ANA DOSYA (~8200 satır)
├── HANDOFF.md              ← Bu dosya
├── .claude/launch.json     ← Preview: "Pedigree App", port 5175, python3 http.server
└── .git/
```

### `index.html` JS bölümleri (önemli fonksiyonlar)
```
- computeLayout()            → layout + sibling symmetry + clusters + overlap
- renderPedigree()           → ana render (safeRender wrapper'ı var; power-pack'te monkey-patch'li)
- renderPhenotypeSegments()  → adaptive segment fills
- findMemberAtPos / findConnectionAtPos / findTargetAtPos / findCoupleMidpointAtPos
- openStatusContextMenu (node sağ tık) / openConnectionContextMenu (çizgi sağ tık)
- editMember() → advanced panel doldur (panel AUTO-OPEN ETMEZ); openAdvancedFromPopup() → açar
- updateLegend() → power-pack'te override (canlı diseases + cancers, collapsible card)
- applyTheme() → ICON_SUN/ICON_MOON innerHTML; INIT'te her zaman çağrılır
- togglePartnershipFlag / removePartnership / removeParentChild (undo destekli)
- Power-pack bölümü (~satır 7000+): legend toggle, text scale, arrow pan, wheel zoom,
  spacebar drag, rubber-band select, group move, connection menu, regression tests
- runTemplateTests / runLayoutTests / runRegressionTests / runAllTests
```

### LocalStorage Anahtarları
```
pedigreeApp_data_v1            → Autosave (JSON)
pedigreeApp_theme              → 'dark' | 'light'
pedigreeApp_lang               → 'tr' | 'en'
pedigreeApp_tourSeen_v1        → '1'
pedigreeApp_labelMode          → 'off' | 'on' | 'proband'
pedigreeApp_leftPanelCollapsed → '0' | '1'
pedigreeApp_legendVisible      → '0' | '1'  (default kapalı)
pedigreeApp_textScale          → float (0.6–2.2)
```

### Repo
- **Owner:** Farmakeus · **Repo:** `Farmakeus/pedigree-app` · **Branch:** `main`
- **Live:** https://farmakeus.github.io/pedigree-app/
- **Latest commit (handoff zamanı):** bu handoff commit'i (856c771 üzerine)

---

## 🎬 Yeni Session Başlangıcı İçin Hızlı Komut

```bash
cd "/Users/ozanvural/claude code/pedigree-app"
git log --oneline -10
/opt/homebrew/bin/node -e "
const fs=require('fs');const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\\s\\S]*?)<\\/script>/);
if(m){try{new Function(m[1]);console.log('JS syntax OK');}catch(e){console.log('ERR:',e.message);}}
"
```
Preview: `mcp__Claude_Preview__preview_start` "Pedigree App" → http://localhost:5175/
Console'da: `runAllTests()` → 38/38 yeşil olmalı.

**Önemli notlar:**
- Kullanıcı **Türkçe** konuşur. UI çoğunlukla TR; EN tam çeviri var.
- Test sonrası **`Cmd+Shift+R`** ile cache temizle.
- Her commit `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` ile imzalanır.
- Değişiklik sonrası `runAllTests()` çağır — commit öncesi.
- **Single source of truth: `familyData`** — yeni feature eklerken bu kuralı bozma.
- **safeRender()** kullan — direct `renderPedigree()` fault-tolerant değil.
- **i18n tuzağı**: `applyTranslations()` `textContent` yazar → ikon+metin butonlarda metin AYRI `<span data-i18n>` içinde olmalı (yoksa SVG silinir).
- **Kullanıcı tercihi**: emoji yerine Lucide-style SVG ikon; basit/mevcut pattern'leri tercih eder.
