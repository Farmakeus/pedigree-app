# 📋 Pedigree Drawing App — Handoff Özeti

> **Önemli:** Bu dosya her session sonunda güncellenir. Yeni session'a başlarken **önce bunu oku**, sonra `git log --oneline -15` ile son commit'leri gözden geçir.

## 1. Proje Amacı ve Kapsamı

**Pedigri Çizim Uygulaması** — genetik soyağacı çizmek için interaktif web uygulaması. Klinik genetik standartlarına uygun (ISCN-benzeri) sembollerle pedigri diyagramları üretir.

**Hedef kullanıcılar:** Genetik danışmanlar, tıp öğrencileri, klinisyen, aile hastalık takibi yapan bireyler. (Geliştiren: tıp doktoru.)

**İki ana mod:** 🧬 Nadir Hastalık (Mendel) · 🩸 Kanser Pedigresi (onkogenetik).

**Ana özellikler:** 3 şablon · tıkla-yerleştir semboller · çizgi-çek bağlantı · multi-select + grup taşıma · adaptive fenotip segmentleri · Save/Load JSON · paylaşılabilir link · Export (SVG/PNG/PDF/CSV) · TR/EN · dark tema · undo/redo · klinik validation + olasılık/Bayes motoru (penetrans·recurrence·overlay WIP) · **73 otomatik test**.

---

## 2. Teknik Kararlar

| Konu | Karar |
|------|-------|
| **Stack** | Vanilla HTML/CSS/JS (framework yok) |
| **Tek dosya** | Tüm kod `index.html` içinde (~9100 satır) |
| **Bağımlılıklar** | jsPDF (CDN, PDF export); **AI özelliği için Anthropic API (BYOK, tarayıcıdan fetch)** |
| **Render** | SVG (viewBox + 1:1 mapping) |
| **Single source of truth** | `familyData = {members[], partnerships[], parentChildLinks[], diseases(Set), nextId, mode, title}` |
| **Persistence** | localStorage (autosave, theme, lang, ... + **`pedigreeApp_anthropicKey`**), sessionStorage (`pedigreeApp_aiPrivacyAck`), JSON file, URL hash |
| **i18n** | `data-i18n*` + `t()`; `applyTranslations()` `textContent` yazar (⚠️ ikon+metin butonlarda metin AYRI `<span>`'de olmalı, yoksa SVG silinir) |
| **Robustness** | `safeRender()` try/catch'le render hatalarını yutar |
| **Deploy** | GitHub Pages — https://farmakeus.github.io/pedigree-app/ |
| **Git** | `main` branch, direct push (single dev) |
| **Preview** | `.claude/launch.json` → "Pedigree App", port 5175, python3 http.server |

**Çalışma deseni (bu oturumda kullanıldı):** Her büyük özellik için **keşif workflow → ben implement → adversarial review workflow (3 lens) → gerçek bulguları süz/düzelt → preview'da doğrula → 38/38 test → commit → push → Pages doğrula**. Ultracode açık; Workflow tool ile çalışıldı.

---

## 3. Bu Session'da Yapılanlar (commit `f44198a` ÜZERİNE)

> Canlıda: `38c0321`'e kadar (olasılık hesaplayıcı dahil). Lokalde **WIP checkpoint** = penetrans/fenokopi + recurrence + overlay (push EDİLMEDİ, 5 review bulgusu BEKLİYOR — bkz. bölüm 4).

### 🚧 #6 Eksik penetrans + fenokopi + recurrence + canvas overlay (WIP — lokal checkpoint, push'lanmadı)
- **Eksik penetrans `f` + fenokopi `c`** (kontrol çubuğunda %, varsayılan 100/0): `penetrance(g,sex,pattern,f,c)` = risk-altı→`f`, değilse→`c`. "Etkilenmiş" artık **yumuşak kanıt** (genotip pin'i DEĞİL); "taşıyıcı" sabit pin kalır. **Geri uyumluluk:** affected hard-pin yalnızca `fullPen=(f===1&&c===0)` dalında korunur (q=0 köşe durumları için). `priorDistribution`/`fullJointClamped`→`buildDomains`→`pinnedDomain` zincirine `fullPen` threaded.
- **Recurrence (`analyzeRecurrence`)**: çifte **fantom çocuk** ekle (id=maxId+1000001), tüm gerçek fenotiplere koşulla, fantomu marjinalleştir (makeFactor `id!==phantomId` ile fantoma kanıt uygulamaz). X-bağlıda cinsiyete göre ayrı. Panelde "Sonraki çocuk riski" bölümü (`renderRecurrenceSection`, seçili bireyin eşi varsa).
- **Canvas overlay**: olasılık modunda her düğüm altında posterior P(taşıyıcı)% (resesif) / P(etkilenmiş)% (dominant). `probOverlay`/`probOverlayMetric` state; ≤30 üye guard'ı; `onProbControlChange` `renderPedigree` çağırır.
- **Doğrulama:** 2. tasarım+türetme workflow'u (penetrans/recurrence modeli + RP1–RP11 oracle'ları, arbitrajlı) → motor **Node'da** doğrulandı → **14 yeni RP testi eklendi, tüm paket 73/73** → tarayıcıda teyit (Huntington Aa=1/6, recurrence 1/4&1/2, overlay).
- **Yeni motor fonksiyonları:** `analyzeRecurrence`, `penetrance(...,f,c)`, `pinnedDomain(...,fullPen)`. UI: `probF/probC/probOverlay/probOverlayMetric`, `renderRecurrenceSection`, `probFindPartner`.

### ✅ #5 Genotip Olasılık Hesaplayıcı — prior + Bayesçi posterior (`10b8efa`/`38c0321`, CANLIDA)
- Header'da **"Olasılık" modu** (Lucide ikon) → `toolMode='probability'` → **tıkla-panel-aç**: bir bireye tıkla, sağda kontrol çubuğu (kalıtım AR/AD/XLR/XLD + allel frekansı `q`) + **prior** (genotip dağılımı, P(etkilenmiş), P(taşıyıcı)) + **denetlenebilir posterior tablosu** (`prior × koşullu = ortak → posterior`).
- **Motor: `PedigreeProb` IIFE — saf, `familyData`'yı ASLA değiştirmez.** Pedigri Bayes ağı üzerinde **tam ortak dağılım enumerasyonu**. Prior = atalardan aşağı segregasyon (hedefin kendi fenotipini yok sayar). Posterior = tüm fenotipleri koşullayan tam Bayes. Bilinmeyen founder → HWE(q); `q=0` deterministik textbook. X-bağlı hemizigot doğru (baba X→kız, Y→oğul). **Konsanguinite döngüleri TAM** (ortak ata bir kez, 1/16 doğrulandı) + tespit/uyarı. Büyük pedigri için boyut guard'ı.
- **Geliştirme deseni:** çok-ajanlı **tasarım+türetme workflow** (3 bağımsız türetme → oracle çapraz-kontrol) → motoru **standalone Node'da** oracle'lara karşı doğrula → entegre et → **adversarial review workflow (8 bulgu, hepsi düzeltildi)**.
- **Review'da bulunan KRİTİK bug (düzeltildi):** founder-hedef posterior'u popülasyon prior'unu atıyordu (singleton clamp → factor 1.0) → düz 0.5/0.5; artık prior-ağırlıklı. Ayrıca: hedefin kendi carrier/affected pin'i posterior'u kısıtlar · panel her veri değişiminde yenilenir (`renderPedigree` hook) · X-bağlı aynı-cinsiyet ebeveyn uyarısı · mod açıkken araç paleti gizlenir · q clamp geri yazılır.
- **15 olasılık testi** (O1–O10 + founder-posterior + carrier-pin). **Tüm paket 59/59.** TR/EN, dark tema, isimler `fhEscapeHtml`'li. Salt-okunur (şemaya dokunmaz).

### 🔒 Güvenlik sertleştirme — cross-user XSS / API anahtar hırsızlığı kapatıldı (`d3f8a53`)
- **Çok-ajanlı adversarial denetim** (6 yüzey × doğrulama; 18 teyit / 9 red) ile **gerçek, ulaşılabilir stored-XSS kümesi** bulundu: saldırganın hazırladığı pedigri (paylaşım linki `#data=` veya JSON dosyası) **başka bir klinisyende açılınca** onun origin'inde JS çalıştırıp **localStorage'daki Anthropic API anahtarını** çalabiliyordu. Güvenilmeyen isim/hastalık/not birçok panelde ham `innerHTML`'e gidiyordu; sayısal alanlar coerce edilmiyordu.
- **İki katmanlı düzeltme:**
  1. **Trust boundary — `loadFamilyData()` artık `sanitizeImportedFamily()` çağırıyor** (dosya · link · autosave hepsi buradan geçiyor). Üye/eş/bağ alanları **allowlist + tip coercion** ile yeniden kuruluyor: id/yaş/kuşak→sayı, customX/customY→sonlu sayı (±100000 clamp; SVG koordinat-attribute injection + viewBox DoS kapanır), string'ler uzunluk-capli, diziler boyut-capli (`Math.max(...ids)` RangeError DoS düzeldi), kopuk referanslar atılıyor, bilinmeyen/`__proto__` anahtarlar düşürülüyor (prototype pollution yok).
  2. **Çıktı escaping (`fhEscapeHtml`)** kalan tüm sink'lere eklendi: üye listesi (isim/hastalık), `getMemberLabel` (`<select>`), eş `<option>`, ilişkiler listesi, değişiklik geçmişi, hastalık-tag editörü, metin-import toast'ları.
- **CSV formül enjeksiyonu** nötralize edildi (baş `= + - @ / TAB / CR` → `'` öneki). **jsPDF CDN'i SRI (sha512) + crossorigin ile pinlendi** (CDN ele geçse bile keyfi JS çalışamaz).
- Tarayıcıda **gerçek payload ile doğrulandı**: hiçbir script çalışmıyor, canlı handler elementi oluşmuyor, prototype kirlenmiyor, her şey inert escaped metin olarak render. Normal pedigriler / `&`'li isimler / PDF export bozulmadı. **+1 regresyon testi**, **44/44 geçiyor**.

### 🆕 #2 Validation / Klinik Muhakeme Motoru (`6d4a0ff`, canlıda)
- Eski **Türkçe-sabit** `validatePedigree()` → **yapısal, iki dilli `computeValidation()`** ile değiştirildi. Bulgular `{errors, warnings, observations}` olarak gruplanır; her bulguda dile bağımsız `code` (test için) + `currentLang`'a göre yerelleştirilmiş `text` var. Mevcut `#validationPanel` render eder.
- **Obligat taşıyıcı tespiti** (klinik çıkarım): iki **sağlam biyolojik ebeveynin etkilenmiş çocuğu** → otozomal resesif varsayımıyla her iki ebeveyn obligat taşıyıcı. **Dikey geçiş (etkilenmiş ebeveyn) ve evlatlık (adopted='in') çocuk çıkarımı bastırır** (yanlış pozitif önleme). Mavi, **bloklamayan** gözlem olarak gösterilir + tek tık **"Taşıyıcı işaretle"** (`applyObligateCarriers`): tek undo adımı, **klinisyen başlatır — asla otomatik değiştirmez** (zaten taşıyıcı/etkilenmiş olanları atlar).
- **Yeni tutarsızlık kontrolleri**: aynı bireyde `carrier`+`affected` çelişkisi · **izole/kopuk düğüm** (hiç eş/ebeveyn-çocuk yok). Mevcut kuşak-sırası, yaş, çoklu-proband, eksik-referans kontrolleri korundu ve artık **II-1/III-2 kodlu, TR/EN yerel**.
- **Panel durumu**: yalnız gözlem varsa **mavi (`info-only`)**, uyarı varsa amber, temizse yeşil. Tüm kullanıcı verisi **`fhEscapeHtml`** ile escape. Başlık "Pedigri Uyarıları" → **"Pedigri Kontrolleri / Pedigree Checks"**.
- **+5 regresyon testi** (obligat tespit · dikey-geçiş bastırma · çelişki · izole düğüm · apply). **43/43 geçiyor**, JS syntax OK. Preview'da TR+EN, açık+dark, apply butonu doğrulandı.

### Önceki özellikler (önceki session'lar, canlıda `f7d1c53`)

### `f44198a` — (önceki session'dan kalan, push'landı)
Visual polish + multi-select + bağlantı sağ-tık menüsü + fenotip/ikiz/kardeş simetri düzeltmeleri. Bu oturumun başında push edildi (local 1 commit öndeydi).

### 🆕 #1 Aile Öyküsü Özeti (`d4e0829` + review `3b628c6`)
- Header'da **📄 Özet** butonu → modal → pedigreeden klinik metin üretir (TR/EN) → **Panoya Kopyala**.
- `generateFamilyHistory()`: genel · proband · etkilenen bireyler (kuşağa göre, II-1/III-2 kodlu) · akrabalık · üreme öyküsü · genetik bulgular · kalıtım paterni gözlemi + disclaimer.
- `fhInheritancePattern()`: **temkinli** kalıtım modu çıkarımı (dikey geçiş→OD, tek-kuşak+konsanguinite→OR, allMale→X-linked ipucu) — her dalda alternatifleri ve "tek başına tanı koydurmaz" uyarısını içerir.
- Review düzeltmeleri: **evlatlık bireyler kalıtım çıkarımından çıkarılır**; "affected ama fenotip yok" ayrımı; güçlü disclaimer; bozuk-veri guard'ları (Array.isArray, Number coerce); modal açıkken dil değişince metin yenilenir.

### 🆕 Metinden Oluştur — deterministik parser (`48866e8`)
- Header'da **📄 Metinden Oluştur** butonu → modal. **İngilizce, satır başına bir kişi** ("Mother, Jane, age 60, breast cancer age 45") → otomatik pedigree.
- `parseTextToFamily()`: proband-merkezli, **slot tabanlı graf** kurucu. İngilizce akrabalık terimleri (mother/maternal grandmother/paternal uncle/sister/son/spouse...), eksik ebeveyn/eş **placeholder ile tamamlanır**, **level→generation normalize** (en üst kuşak=1), rol→cinsiyet + cinsiyete-özgü kanserden cinsiyet çıkarımı (ovarian→female, prostate→male; meme HARİÇ).
- Çıktı düzenlenebilir textarea'ya yazılır → "Pedigri Oluştur" → `saveState()` (tek undo) → `loadFamilyData()`.

### 🆕 AI ile yapılandır — BYOK (`f7d1c53`)
- Aynı modalda **✨ AI ile yapılandır** butonu + "AI ayarları" (details): API anahtarı (password), anonimleştirme toggle, gizlilik uyarısı.
- `aiStructureText()` / `aiConvertToLines()`: kullanıcının **kendi** Anthropic API anahtarıyla (localStorage) serbest anamnez metnini (TR veya EN) tarayıcıdan **doğrudan** Anthropic API'ye gönderir (`anthropic-dangerous-direct-browser-access` header, **model `claude-opus-4-8`**, `output_config.format` JSON schema → `{lines:[...]}`), satır formatı döndürür → textarea → **mevcut deterministik parser** kurar (klinisyen onayı korunur).
- **AI sadece metni normalize eder; grafiği test edilmiş `parseTextToFamily` kurar.**
- Review sertleştirmeleri: anahtar DOM'a auto-populate edilmez · `sk-ant-` format kontrolü · "anahtarı sil" · oturum-başına onay (sessionStorage) · "anonim yalnızca ÇIKTIYI gizler" uyarısı · 45s timeout · max_tokens 4000.

### 🔒 Güvenlik — XSS sertleştirme (`f7d1c53` içinde)
- **Gerçek aktif XSS bulundu ve kapatıldı**: kanser tipi SVG label'ında (`<text>● ${shortName}`) `<svg onload=...>` çalışıyordu. Bu **AI'a özel değil, elle girişi de etkileyen önceden var olan açıktı.**
- `fhEscapeHtml()` (global helper) ile tüm kullanıcı-girişi alanları (**isim, not, variant, hastalık, kanser tipi**) SVG + tooltip innerHTML + advanced panel + silme toast'ında escape edilir.
- Doğrulandı: kötü-niyetli payload render'ında hiç `on*` attribute / enjekte element hayatta kalmıyor.

---

## 4. Devam Eden İş — Şu An Nerede Kaldık?

### ✅ Son durum
- **#6 Penetrans/fenokopi + recurrence + overlay** kodlandı, **73/73 test geçiyor**, tarayıcıda doğrulandı. ⚠️ **Lokal WIP checkpoint commit'li, PUSH EDİLMEDİ.** Adversarial review **5 bulgu** çıkardı, **henüz düzeltilmedi** (aşağıda). Özellik çalışıyor; bulgular köşe durumlar.
- Önceki: `38c0321`'e kadar **canlıda** (olasılık hesaplayıcı dahil).

### 🎯 SIRADAKİ İŞ — yarın: önce #6 review bulgularını düzelt, sonra push
**Düzeltilecek review bulguları (önem sırasına göre):**
1. **HIGH — recurrence q=0 + (f<1, c=0) süreksizliği:** etkilenmiş bir FOUNDER'lı çiftte, `q=0` ve penetrans<%100 (fenokopi %0) iken `analyzeRecurrence` "Hesaplanamadı" döndürüyor (q=1e-10'da doğru: 0.40). Sebep: `fullPen=false` → affected pin kapalı; `founderFactorMap` q=0'da kütleyi AA'ya yığıyor; affected-kanıtı `penetrance(AA)=c=0` → joint sıfır. **Fix (a, önerilen):** `pinnedDomain`'e `c` thread et; **`c===0 && m.affected`** ise (fullPen'den BAĞIMSIZ) dom’i risk-altı genotiplere kısıtla — q=0 collapse {Aa,aa} içinde min-mutant=Aa seçer, `penetrance(Aa)=f` → doğru. (RP9 c=0.10 etkilenmez.) Regresyon testi ekle: AD q=0 f=0.8 etkilenmiş-baba×normal = 0.40; AR q=0 f=0.5; AD q=0 f=0.99.
2. **MEDIUM — prior, f<1/c>0'da etkilenmiş ataları yok sayıyor:** `priorDistribution` `withEvidence=false` çağrılıyor; affected pin de `fullPen` kapalıyken yok → prior az-koşullanıyor (süreksiz). **Fix:** prior'ı `withEvidence=true` ile (hedefin KENDİ fenotipi hariç, `id!==target` skip — mevcut `phantomId` skip mekanizmasını yeniden kullan) enumerate et ki ata fenotipleri her f/c'de yumuşak kanıt olarak girsin. (Posterior zaten doğru; bu yalnızca PRIOR satırını etkiler.)
3. **LOW — (önceden var olan) q=0 + tam penetrans + etkilenmiş founder:** clamp'lı founder hedefi q=0'da posteriorValid=false. Extension'ın getirdiği YENİ hata değil; #1 fix'i (b şıkkı / epsilon floor) ile birlikte ele alınabilir.
4. **LOW — kendiyle-eş recurrence:** `momId===dadId` reddedilmiyor. **Fix:** `analyzeRecurrence`'a `if (mom.id===dad.id) return {error:'self-couple'}`; renderRecurrenceSection'da ele al; import sanitizer'da self-partnership düş.
5. **LOW — `probFindPartner` boşanmış eşi seçebiliyor:** önce `!p.divorced` eşi seç; çoklu eşte hepsini listelemek daha iyi.
> Bulgu ayrıntısı: `tasks/wi0a6odk0.output`. Reddedilen 4 bulgu (overlay debounce, tooLarge cross-contam, vb.) gerçek değildi.

**Sonra:**
6. #6 düzeltilince push et (`gh api .../pages/builds/latest` ile Pages doğrula; sandbox DNS → `dangerouslyDisableSandbox:true`, birkaç retry).
7. **Bekleyen 3 ürün kararı** (güvenlik denetiminden, kod açığı değil — kullanıcıya soruldu, henüz seçmedi):
   - **API anahtarı localStorage vs sessionStorage**: XSS kapandığı için risk çok düştü; yine de sessionStorage/oturum-içi bellek daha güvenli ama her oturum anahtar yeniden girilir (UX maliyeti).
   - **AI "anonim" toggle yanıltıcı**: serbest metin (isimler dâhil) Anthropic'e olduğu gibi gider; toggle yalnızca ÇIKTIYI gizler. İstenirse gönderilmeden önce gerçek anonimleştirme eklenebilir.
   - **CSP meta yok**: inline-script mimarisi `'unsafe-inline'` gerektirdiğinden tam CSP zor; ama `connect-src`/`img-src` kısıtı, sızıntı kanalını (key exfil) kapatan derinlemesine savunma olabilir — dikkatli denenmeli (export/AI'yi bozmadan).
3. Opsiyonel #2 genişletme (kullanıcı isterse): X'e bağlı resesif obligat taşıyıcı · iki etkilenmiş ebeveynin sağlam çocuğu (AR'da beklenmez) · konsanguinite + tek-kuşak vurgusu. `fhInheritancePattern()` (Özet) ile `computeValidation()` ayrı duruyor.

### 🤔 İptal Edilen / İstenmeyenler
- **#3 Genetik test detay alanı (ACMG)** — kullanıcı istemedi.
- **#4 Print modu + anonimleştirme** — kullanıcı istemedi (not: gizlilik anonimleştirmesi AI özelliğinde kısmen ele alındı).
- **Yol 2 backend proxy** — şimdilik BYOK seçildi; çok-kullanıcı olursa backend düşünülebilir.

---

## 5. Bilinen Sorunlar / Limitasyonlar

1. **PDF export raster** (vector değil).
2. **Help/Özet/Import modalları dark temada beyaz kalır** — metin koyu olduğu için OKUNUR ve kendi içinde tutarlı (3 review'da doğrulandı). İstenirse 3 modal birlikte dark yapılabilir (ayrı iş).
3. Mobile/tablet optimize değil.
4. **AI özelliği gizlilik**: serbest metin Anthropic API'ye gider (üçüncü taraf). Anonim toggle yalnızca ÇIKTIYI gizler, gönderilen metni DEĞİL — uyarı UI'da net. Hasta tanımlayıcı veri girilmemeli. (Bölüm 4'teki ürün kararı.)
5. **AI halüsinasyon riski**: AI çıktısı düzenlenebilir textarea'da gösterilir (klinisyen onayı zorunlu adım), doğrudan pedigree'ye dönüşmez. Sistem prompt "faithful, do not invent" diyor.
6. ~~CSV `c.type` escape~~ ve ~~stored-XSS / coordinate injection / proto pollution / DoS~~ → **`d3f8a53` ile KAPATILDI** (bkz. bölüm 3 güvenlik). Güvenilmeyen veri artık `sanitizeImportedFamily()` (loadFamilyData) + tüm sink'lerde `fhEscapeHtml` ile temizleniyor; CSV formül-enjeksiyonu nötr.
7. **Kalan güvenlik ürün kararları**: API anahtarı localStorage'da · AI gönderdiği metni anonimleştirmiyor · CSP meta yok. (Detay: bölüm 4, madde 2.)
7. Dosya boyutu ~9100 satır / ~440KB.

---

## 6. Önemli Dosya Yolları ve Yapı

### Dizin
```
/Users/ozanvural/claude code/pedigree-app/
├── index.html          ← TEK ANA DOSYA (~9100 satır)
├── HANDOFF.md          ← Bu dosya
├── .claude/launch.json ← Preview: "Pedigree App", port 5175
└── .git/
```

### Yeni eklenen JS fonksiyonları (bu oturum, script sonuna yakın)
```
# Aile öyküsü özeti
generateFamilyHistory, fhInheritancePattern, fhBuildIdMap, fhRomanGen,
fhSexWord, fhIsAffected, fhPhenotypeText, fhLabel,
openSummaryModal / closeSummaryModal / copySummary

# Metinden pedigri (deterministik)
parseTextToFamily, fhResolveRelationship, fhParseImportLine, fhApplyAttribute,
fhParseCancer, fhIsAttributePart, FH_CANCER_RE,
openImportModal / closeImportModal / buildFamilyFromText / loadImportExample / importShowStatus

# AI ile yapılandır (BYOK)
aiStructureText, aiConvertToLines,
fhGetAnthropicKey / fhSaveAnthropicKey / fhClearAnthropicKey,
FH_AI_KEY_LS='pedigreeApp_anthropicKey', FH_AI_ACK_LS='pedigreeApp_aiPrivacyAck'

# Güvenlik
fhEscapeHtml(s)  ← global; SVG/tooltip/panel/toast'ta kullanıcı verisi basmadan önce ÇAĞIR
sanitizeImportedFamily(data) → loadFamilyData içinde çağrılır; güvenilmeyen JSON'u
                               allowlist + tip coercion ile yeniden kurar (XSS/DoS/proto pollution kalkanı)
fhStr / fhNumOrNull / fhIntOrNull / fhSafeId / fhCoordOrNull → coercion yardımcıları
csvEscape(val) → CSV alanını formül-enjeksiyonuna karşı da nötralize eder

# Validation / Klinik muhakeme motoru (#2)
computeValidation()       → {errors, warnings, observations}; her bulgu {code, text, action?}
                            (code dile bağımsız/test için; text currentLang'a göre yerel)
updateValidationPanel()   → #validationPanel'i render eder (esc + sev-error/warn/info + info-only)
applyObligateCarriers(ids)→ verilen ebeveynleri carrier işaretler (tek undo, klinisyen tetikler)
```

### Programatik aile kurma (yeni özellikler bunu kullanır)
```
clearAllSilent()  → familyData sıfırla (mode korunur)
setMode('rare'|'cancer')
addMemberDirect(name,sex,gen,age,affected,carrier,deceased,proband,diseases[],twinType,twinPId,note,abort,customX,customY) → id döner
addMemberWithCancers(sex,gen,name,age,deceased,proband,cancers[],variant,note) → id döner
familyData.partnerships.push({partner1,partner2,consanguinity,divorced})
familyData.parentChildLinks.push({parent1,parent2,childId})
updateUI(); renderPedigree();
saveState(action,detail)  → DEĞİŞİKLİKTEN ÖNCE çağır (undo snapshot)
loadFamilyData(data)      → {members,partnerships,parentChildLinks,diseases(array),nextId,mode,title} alır; setMode+updateUI+render yapar; tek undo için saveState'i ÖNCE çağır
customX:null → auto-layout (computeLayout)
```

### LocalStorage / SessionStorage Anahtarları
```
pedigreeApp_data_v1, _theme, _lang, _tourSeen_v1, _labelMode,
_leftPanelCollapsed, _legendVisible, _textScale
pedigreeApp_anthropicKey        → AI BYOK anahtarı (localStorage)
pedigreeApp_aiPrivacyAck (sessionStorage) → AI gizlilik onayı (oturum başına)
```

### Repo
- **Owner:** Farmakeus · **Repo:** `Farmakeus/pedigree-app` · **Branch:** `main`
- **Live:** https://farmakeus.github.io/pedigree-app/
- **Son commit (lokal):** WIP `#6 penetrans/recurrence/overlay` — **push EDİLMEDİ**, 5 review bulgusu bekliyor (bkz. bölüm 4). Canlı: `38c0321`.

---

## 🎬 Yeni Session Başlangıcı İçin Hızlı Komut

```bash
cd "/Users/ozanvural/claude code/pedigree-app"
git log --oneline -10
/opt/homebrew/bin/node -e "
const fs=require('fs');const html=fs.readFileSync('index.html','utf8');
const m=[...html.matchAll(/<script>([\\s\\S]*?)<\\/script>/g)].pop();
try{new Function(m[1]);console.log('JS syntax OK');}catch(e){console.log('ERR:',e.message);}
"
```
Preview: `mcp__Claude_Preview__preview_start` "Pedigree App" → http://localhost:5175/
Console'da: `runAllTests().allPassed` → `true` olmalı (73 test).

**Önemli notlar:**
- Kullanıcı **Türkçe** konuşur. UI TR/EN tam çeviri.
- **Single source of truth: `familyData`** — yeni feature eklerken bu kuralı bozma.
- **safeRender()** kullan; **i18n tuzağı**: ikon+metin butonlarda metin AYRI `<span data-i18n>` içinde.
- **Kullanıcı tercihi**: emoji yerine **Lucide-style SVG ikon**; **basit/mevcut pattern'leri** tercih eder.
- **XSS**: SVG/innerHTML'e kullanıcı verisi basarken **`fhEscapeHtml()`** kullan.
- **git push sandbox'ta DNS'e takılıyor** → `dangerouslyDisableSandbox: true` ile çalıştır; bazen birkaç retry gerekir (15s arayla). Pages build'i `gh api repos/Farmakeus/pedigree-app/pages/builds/latest` ile izle, CDN propagation birkaç dk sürebilir.
- Her commit `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` ile imzalanır (mevcut model imzanı kullan).
- Değişiklik sonrası `runAllTests()` + JS syntax kontrolü; commit öncesi.
- Demo bağlamı: 8-15 üyeli pedigriler; 50+ test edilmedi.
