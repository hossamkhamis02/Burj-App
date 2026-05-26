# خطة إعادة بناء `index.html` — نظام إدارة برج الأمل

> **هدف الخطة:** Claude Code يقرأ هذا الملف ويُنفّذ كل مرحلة بالترتيب دون أي خروج عن التفاصيل المذكورة.

---

## 0. المدخلات والمخرجات

| البند | التفصيل |
|---|---|
| **ملف المصدر** | `Index.html` (الملف الحالي — لا يُمس) |
| **ملف الإخراج** | `index.html` جديد (ملف واحد: HTML + CSS + JS inline) |
| **بيئة التشغيل** | Google Apps Script — `google.script.run` متاح |
| **اللغة** | عربي RTL — جميع الأسماء والنصوص تُنقل حرفياً |

---

## 1. نقل البيانات حرفياً (لا تعديل)

انقل هذه الكتل كما هي من `Index.html`:

### 1-أ. ثوابت العرض
```js
const FN = ['الأول','الثاني', ... ,'الثالث عشر'];   // أسماء الطوابق
const PN = ['يسار','وسط','يمين'];
const COL = [2, 1, 0];
```

### 1-ب. بيانات السكان الأولية (SEED_T)
كل 34 مدخلاً من `SEED_T` في الملف الأصلي (السطور 259–294) تُنقل بالضبط — الأسماء العربية، `closed`, `notes`, `phone`.

### 1-ج. سجل المعاملات الأولي (SEED_TXS)
كل المعاملات من السطر 296 تُنقل كاملةً.

### 1-د. قيم الحالة الابتدائية
```js
let CM = '2026-05';
let SECTIONS = [
  {id:'floors', title:'الأدوار', type:'floors', visible:true, icon:'🏠'},
  {id:'txs',    title:'العمليات', type:'txs',   visible:true, icon:'🧾'},
  {id:'settings',title:'الإعدادات',type:'settings',visible:true,icon:'⚙'}
];
let CSID = 'floors';
```

---

## 2. تغيير نموذج البيانات

### 2-أ. إضافة كائن `SC` — "Section Closed" (الجديد)

```js
// SC[sectionId][residentName] = true  →  مغلق في هذا القسم فقط
// SC[sectionId][residentName] = false →  مفتوح في هذا القسم (يُلغي الإغلاق الكوني)
let SC = {};   // يُحفظ ويُحمَّل مع باقي الحالة
```

**القاعدة:** لكل قسم (section) خريطة مستقلة تتحكم في حالة "مغلق/مفتوح" لكل ساكن داخل هذا القسم تحديداً.  
- إذا لم يوجد مدخل في `SC[sectionId][name]` → يُستخدم `T[name].closed` كقيمة افتراضية.
- إذا وُجد مدخل → يُستخدم هذا المدخل بدلاً منه.

أضف دالة مساعدة:
```js
function isClosedIn(name, sectionId) {
  if (SC[sectionId] && SC[sectionId][name] !== undefined)
    return SC[sectionId][name];
  return !!(T[name] && T[name].closed);
}
```

### 2-ب. تعديل دالة الحفظ والتحميل

في `doSave()` أضف `SC` للحزمة المُرسَلة:
```js
{ tenants: T, payments: P, transactions: TX,
  currentMonth: CM, sections: SECTIONS, mapping: M, sectionClosed: SC }
```

في `load()` استقبل:
```js
SC = d.sectionClosed || {};
```

### 2-ج. Migration عند التحميل (للبيانات القديمة)

إذا كانت `d.sectionClosed` غير موجودة (بيانات قديمة):
- لكل قسم من `SECTIONS` ذو `type === 'floors'`
- لكل ساكن في `T` حيث `T[name].closed === true`
- اضبط `SC[sectionId][name] = true`

---

## 3. التغيير الأول — Checkbox مغلق مرتبط بالقسم

### 3-أ. في `renderFloors()`

استبدل:
```js
if (t.closed) { d.className = 'uc cl'; ... }
```
بـ:
```js
const closed = isClosedIn(t.name, CSID);
if (closed) { d.className = 'uc cl'; dot.className = 'dot x'; }
```

وفي onclick:
```js
if (!isClosedIn(t.name, CSID)) d.onclick = () => openMdl(f, p, CSID);
```

### 3-ب. في `openMdl()`

عند عرض بيانات الساكن أضف مؤشر "مغلق في هذا القسم" إن كان مغلقاً.

### 3-ج. في popup تعديل الساكن (edit-popup-bg)

**استبدل checkbox الوحيد:**
```html
<!-- قديم: -->
<input type="checkbox" id="at-cl"> مغلق / غير نشط (لا يظهر في إحصائيات الدفع)

<!-- جديد: -->
<input type="checkbox" id="at-cl-global"> مغلق كونياً (في كل الأقسام)
<!-- + -->
<div id="at-cl-per-section">
  <!-- يُعبَّأ ديناميكياً بـ renderSectionClosedCheckboxes(name) -->
</div>
```

### 3-د. دالة `renderSectionClosedCheckboxes(name)`

```js
function renderSectionClosedCheckboxes(name) {
  const container = document.getElementById('at-cl-per-section');
  const floorSecs = SECTIONS.filter(s => s.type === 'floors');
  container.innerHTML = floorSecs.map(sec => `
    <div style="display:flex;align-items:center;gap:8px;padding:4px 0">
      <input type="checkbox"
             id="cl-sec-${sec.id}"
             style="width:16px;height:16px;cursor:pointer"
             ${isClosedIn(name, sec.id) ? 'checked' : ''}
             onchange="toggleClosedInSection('${name}','${sec.id}',this.checked)">
      <label for="cl-sec-${sec.id}" style="font-size:12px;cursor:pointer">
        مغلق في: ${sec.icon || ''} ${sec.title}
      </label>
    </div>
  `).join('');
}
```

استدعِها في `editTenant(name)` بعد تعبئة الحقول.  
استدعِها أيضاً في `openAddTenant()` (ستكون فارغة/غير محددة للساكن الجديد).

### 3-هـ. دالة `toggleClosedInSection(name, sectionId, value)`

```js
function toggleClosedInSection(name, sectionId, value) {
  if (!SC[sectionId]) SC[sectionId] = {};
  SC[sectionId][name] = value;
  autoSave();
  refreshAll();
}
```

### 3-و. تعديل `saveTenant()`

```js
// احفظ الحالة الكونية
T[name].closed = document.getElementById('at-cl-global').checked;
// SC يُحدَّث مباشرةً عبر toggleClosedInSection ولا يحتاج حفظاً هنا
```

---

## 4. التغيير الثاني — عداد "دفعوا هذا الشهر" مرتبط بالقسم

### 4-أ. في `renderStats()`

عدّل حساب `paid/total` ليعتمد على القسم النشط `CSID`:

```js
function renderStats() {
  // ... (income/expense كما هو)

  let paid = 0, total = 0;
  const sec = SECTIONS.find(s => s.id === CSID);

  if (sec && sec.type === 'floors') {
    // حساب خاص بالقسم الحالي فقط
    const namesInSection = [...new Set(
      Object.entries(M)
        .filter(([loc]) => parseUK(loc).sid === CSID)
        .map(([, name]) => name)
    )];

    namesInSection.forEach(name => {
      if (!T[name]) return;
      if (isClosedIn(name, CSID)) return;   // مغلق في هذا القسم → لا يُحتسب
      total++;
      // هل دفع في هذا القسم تحديداً؟
      const locs = Object.entries(M)
        .filter(([loc, n]) => n === name && parseUK(loc).sid === CSID);
      const hasPaid = locs.some(([loc]) => {
        const { f, p } = parseUK(loc);
        return gTP(f, p, CSID) > 0;
      });
      if (hasPaid) paid++;
    });

  } else {
    // في تبويبات غير الأدوار: أظهر المجموع الكوني (السلوك القديم)
    const uniqueNames = [...new Set(Object.values(M))];
    uniqueNames.forEach(name => {
      const t = T[name];
      if (!t || t.closed) return;
      total++;
      let hasPaid = false;
      Object.entries(M).forEach(([loc, tName]) => {
        if (tName === name) {
          const { f, p, sid } = parseUK(loc);
          if (gTP(f, p, sid) > 0) hasPaid = true;
        }
      });
      if (hasPaid) paid++;
    });
  }

  document.getElementById('s-paid').textContent = paid + '/' + total;
}
```

### 4-ب. تحديث label البطاقة الإحصائية

في HTML الـ stats card:
```html
<div class="sc-l" id="s-paid-label">دفعوا هذا الشهر</div>
```

في `renderStats()` بعد تحديد القسم:
```js
const label = (sec && sec.type === 'floors')
  ? 'دفعوا — ' + (sec.title || 'الأدوار')
  : 'دفعوا هذا الشهر';
document.getElementById('s-paid-label').textContent = label;
```

### 4-ج. استدعِ `renderStats()` عند تبديل التبويبات

في `sw(id, el)` — تأكد أن `renderStats()` يُستدعى في نهايتها (كان يستدعيها `refreshAll()` بالفعل، لكن تأكد أنها تُستدعى بعد `CSID = id`).

---

## 5. لا تغيير على باقي الدوال

| الدالة | الحالة |
|---|---|
| `renderFloors()` | تعديل بسيط فقط (التغيير 3-أ) |
| `renderTenants()` | كما هي |
| `renderTxs()` | كما هي |
| `addTx()` / `delTx()` | كما هي |
| `recPay()` / `delPay()` | كما هي |
| `addSection()` / `delSection()` | كما هي |
| `renderSectionsList()` | كما هي |
| `renderTabs()` | كما هي |
| `renderEditList()` | كما هي |
| `doDrive()` — أضف `sectionClosed: SC` للتصدير | تعديل بسيط |

---

## 6. هيكل الملف النهائي

```
index.html (ملف واحد)
├── <!DOCTYPE html>
├── <head>
│   ├── meta charset, viewport
│   ├── title: نظام إدارة برج الأمل
│   └── <style> ... كل CSS inline ... </style>
├── <body>
│   ├── <div class="w" id="root">
│   │   ├── .hdr  (header)
│   │   ├── .stats (4 بطاقات — s-paid-label قابل للتحديث)
│   │   ├── .tabs #tabs-bar
│   │   ├── #sec-floors
│   │   ├── #sec-tenants
│   │   ├── #sec-txs
│   │   ├── #sec-settings
│   │   ├── #edit-popup-bg  (popup تعديل الساكن — مع checkboxes الجديدة)
│   │   └── #popup-bg       (popup تسجيل الدفع — كما هو)
│   └── </div>
└── <script> ... كل JS inline ... </script>
```

---

## 7. نقاط تحقق (Checklist)

- [ ] كل 34 اسم ساكن عربي موجود في `SEED_T` وينعكس في الشاشة
- [ ] كل معاملة في `SEED_TXS` محفوظة
- [ ] الاتصال بـ `google.script.run` يعمل (`loadData` / `saveData`)
- [ ] `SC` يُحفظ ويُحمَّل مع باقي البيانات
- [ ] `isClosedIn(name, sectionId)` تُرجع القيمة الصحيحة
- [ ] عند فتح popup تعديل ساكن: تظهر checkbox لكل قسم floors موجود
- [ ] تغيير checkbox قسم معين → يُغلق/يفتح الساكن في ذلك القسم فقط دون تأثير على الأقسام الأخرى
- [ ] `دفعوا — الأدوار` يعرض الرقم الصحيح للقسم النشط فقط
- [ ] عند إضافة قسم جديد (clone) → يحصل على checkbox منفصل في popup التعديل
- [ ] تصدير JSON (Drive) يحتوي `sectionClosed`
- [ ] RTL/Arabic rendering صحيح في المتصفح
- [ ] لا يوجد `localStorage` — كل التخزين عبر `google.script.run`

---

## 8. ملاحظات للتنفيذ

1. **ملف واحد فقط** — لا `import`، لا bundler، لا TypeScript compile step. كل شيء inline.
2. **لا تحذف `google.script.run`** — التطبيق يعمل داخل Google Apps Script فقط.
3. **`SC` migration**: إذا كانت البيانات المحملة لا تحتوي `sectionClosed`، شغّل migration تلقائياً كما في القسم 2-ج.
4. **لا تُعدّل `SEED_T` أو `SEED_TXS`** — هي فقط بيانات أولية للمرة الأولى، لا تُغيّر قيمها.
5. **`autoSave()`** يُستدعى بعد أي تغيير في `SC`.