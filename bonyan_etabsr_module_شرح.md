# BonyanETABSR — شرح المود الكامل

هذا المستند يشرح بنية، سلوك، وطرق تشغيل مود **BonyanETABSR** المرفق في سؤالك. الهدف: توضيح كل مكون عملياً بحيث تستطيع التشغيل، التصحيح، والتعديل بسرعة.

## نظرة عامة

BonyanETABSR هو تطبيق GUI (Tkinter) مع محرك استخراج منفصل (`engine.py`) يتصل/يفتح نموذج ETABS عبر API الخاص به، يصدّر جداول CSV مؤقتة ثم يؤرشفها إلى قاعدة SQLite. يدعم واجهة ثنائية اللغة (عربي/إنجليزي) وملف إعدادات JSON مركزي.

---

## هيكل الملفات (المهم)

- `run.py` / نقطة الدخول البسيطة التي تستدعي `BonyanApp`.
- `bonyan_app/app.py` — الكلاس الرئيسي `BonyanApp` المبني فوق `MainWindow`، يدير الأحداث، حفظ الإعدادات، تشغيل المحرك كعملية فرعية، وقراءة مخرجاته.
- `bonyan_app/core/engine.py` — المحرك الفعلي المسؤول عن:
  1. قراءة إعدادات JSON
  2. الاتصال بـ ETABS API
  3. طلب جداول CSV
  4. أرشفة الجداول داخل SQLite
- `bonyan_app/ui/main_window.py` — واجهة المستخدم (Tkinter + ttk) وتعامل ثنائي اللغة وRTL.
- `bonyan_app/ui/about_window.py` — نافذة "حول" التطبيق.
- `bonyan_app/ui/localization.py` (أو `i18n`) — تحميل ملفات JSON للترجمة (`i18n/en.json`, `i18n/ar.json`).
- `configs/*.json` — ملفات تعريف لكل كائن (columns.json, beams.json, walls.json, slabs.json) تحتوي `related_tables` المطلوبة.
- `configs/settings.json` — ملف الإعدادات المستخدم (مثال أدناه).

---

## ملف الإعدادات (مثال)

```json
{
    "output_directory": "C:/Users/User/Desktop/BounyanD",
    "etabs_install_dir": "C:\\Program Files\\Computers and Structures\\ETABS 22",
    "extract_objects": {
        "columns": true,
        "beams": true,
        "walls": true,
        "slabs": false
    }
}
```

> الحقول الأساسية: `output_directory`, `etabs_install_dir`, `extract_objects` (قائمة كائنات للاستخراج true/false).

---

## سريان العمل (Engine phases)

1. **قراءة الإعدادات وبناء قائمة المهام**
   - يقرأ `settings.json` وملفات `configs/<object>.json` ليكوّن مجموعة `task_list` من أسماء الجداول المطلوب استخراجها.
2. **الاتصال بـ ETABS**
   - يحاول تحميل DLLs من مسار التثبيت (`ETABSv1.dll`, `CSiAPIv1.dll`).
   - يدعم إما فتح ملف ETABS محدد (`--etabs_file`) أو الإلحاق بالنموذج النشط.
   - يطبع إشارات تقدم `BONYAN_PROGRESS:CONNECTING_START` و `CONNECTING_END`.
3. **طلب CSV من ETABS**
   - لكل جدول في `task_list` يستدعي `GetTableForDisplayCSVFile` ويمرر مسار ملف CSV مؤقت.
   - يتحقق من رمز الإرجاع (`ret_code`) ومن وجود ملف CSV وحجمه (>0) قبل إضافته لقائمة الأرشفة.
   - يطبع تحديثات `BONYAN_PROGRESS:EXPORTING_UPDATE:<done>/<total>`.
4. **أرشفة إلى SQLite**
   - إذا لا توجد CSVs ناجحة، يطبع تحذير حاسم وينهي.
   - ينشئ قاعدة SQLite باسم `{model_name}_BonyanETABSR_Archive.veda` في `output_directory`، ثم يستورد كل CSV إلى جدول باسم مناسب.
   - يطبع `BONYAN_PROGRESS:ARCHIVING_UPDATE:<i>/<n>`.
5. **تنظيف الموارد وإنهاء**
   - يفصل مراجع ETABS، يحذف المجلد المؤقت، ويطبع `BONYAN_PROGRESS:DONE`.

---

## رسائل التقدم (Progress tokens)

الواجهة تعتمد على سطور خاصة يطبعها المحرك لتحديث شريط التقدّم:
- `BONYAN_PROGRESS:CONNECTING_START` / `CONNECTING_END`
- `BONYAN_PROGRESS:EXPORTING_START:<total>`
- `BONYAN_PROGRESS:EXPORTING_UPDATE:<done>/<total>`
- `BONYAN_PROGRESS:ARCHIVING_START:<total>`
- `BONYAN_PROGRESS:ARCHIVING_UPDATE:<done>/<total>`
- `BONYAN_PROGRESS:DONE`

واجهة المستخدم (`handle_progress_update`) تحسب نسب ثابتة لكل مرحلة: اتصال 10%، تصدير 45%، أرشفة 45%.

---

## نقاط مهمة في الكود (ملاحظات تقنية)

- `resource_path(relative_path)` يدعم التشغيل كـ PyInstaller (يستخدم `sys._MEIPASS`). يجب التأكد من أن الملفات `configs` و `bonyan_app/core/engine.py` مضمنة عند التجميع.
- عند فتح عملية فرعية للمحرك، يتم تشغيل Python مع `-u` لضمان إخراج فورياً (`text=True`, `encoding='utf-8'`). تهيئة `startupinfo` تخفي نافذة الكونسول.
- `engine.py` يعتمد `pythonnet` (`clr`) للوصول إلى ETABS .NET API. يجب تثبيت واختبار `pythonnet` في بيئة التشغيل.
- ملفات CSV تُكتب في مجلد مؤقت ثم تقرأ `pandas.read_csv(..., low_memory=False, encoding='utf-8-sig')` قبل حفظها داخل SQLite.
- قاعدة البيانات تستخدم امتداد `.veda` لكنه في الواقع ملف SQLite. تأكد من طرق النسخ الاحتياطي والمزامنة.

---

## واجهة المستخدم (ملاحظات)

- `MainWindow` منشأة لفصل إنشاء الودجيت عن تحديث النصوص (`_create_and_layout_widgets` + `update_ui_texts`).
- دعم RTL للعربية باستخدام `arabic_reshaper` و `bidi` عند العرض.
- حقل اختيار المصدر (`attach` أو `open`) يتحكم في تمكين/تعطيل حقل طريق الملف.
- إعدادات الاستخراج تتم مزامنتها من الإعدادات المحفوظة عبر `SettingsManager` (كائن مساعد غير مرفق هنا — تأكد من وجوده).

---

## كيفية التشغيل

### من الواجهة (GUI)
1. شغّل `run.py` أو `python -m bonyan_app.run` (أو `python run.py`)—سيعمل `BonyanApp()`.
2. اختر اللغة، مسار تثبيت ETABS إذا لزم، مجلد الإخراج، وطريقة المصدر (Attach أو Open).
3. اضغط "Start Extraction".

### من سطر الأوامر (لتجريب المحرك مباشرة)
```bash
python bonyan_app/core/engine.py --settings path/to/settings.json --config path/to/configs --etabs_file path/to/file.edb
```

---

## متطلبات النظام والاعتمادات

- Python 3.8+ (توصية 3.9/3.10)
- مكتبات: `pythonnet` (clr)، `pandas`, `sqlite3` (مضمن)، `tkinter`, `arabic_reshaper`, `python-bidi`, `sv_ttk`، `bincopy` أو ما شابه إذا استُخدمت في التجميع.
- ETABS مثبت (نسخة مطابقة للمسار في `etabs_install_dir`) ويجب أن تكون مكتبات `ETABSv1.dll` و `CSiAPIv1.dll` متوفرة.

---

## تحري الأخطاء السريع (Checklist)

- تأكد أن `etabs_install_dir` صحيح ويحتوي على `ETABSv1.dll` و `CSiAPIv1.dll`.
- تشغيل Python كـ 64-bit يتطابق مع إصدار ETABS (عادة ETABS 64-bit).
- امتيازات الكتابة للمجلد المؤقت و `output_directory`.
- تأكد من أن أسماء الجداول في ملفات `configs/*.json` صحيحة ومطابقة لأسماء جداول ETABS.
- في حال خطأ "failed to load ETABS API" — تحقق من `pythonnet` وإصدار CLR المتوافق.
- إذا أبلغ المحرك عن ملف CSV فارغ: تفقد أذونات الكتابة إلى `TEMP_CSV_DIR` أو مشاكل في استدعاء `GetTableForDisplayCSVFile`.

---

## إخراج التطبيق

- ملفات CSV مؤقتة في مجلد `temp` (تحذف عند الانتهاء).
- ملف قاعدة البيانات النهائي: `{model_name}_BonyanETABSR_Archive.veda` داخل `output_directory`. يحتوي جداول SQL مسماة وفق `clean_table_name`.
- سجل التشغيل يظهر رسائل تشخيصية مفصّلة في واجهة التطبيق.

---

## تحسينات مقترَحة

- إضافة تحقق/اختبار وجود DLLs مبكراً قبل بدء العملية.
- إضافة خيار لإبقاء ملفات CSV المؤقتة (لأغراض التصحيح).
- تسجيل (logging) إلى ملف مستقل بجانب طباعة stdout لتسهيل الدعم.
- تحسين إدارة الأخطاء عند استدعاء ETABS API مع رموز خطأ مفصّلة.

---

## خاتمة

المود منظم جيداً: فصل الواجهة عن المحرك، استعمال ملفات إعدادات وملفات تعريف الجداول، وآليات تقدم واضحة. للمضي قدماً: اختبر الاتصال بالـ ETABS محلياً، تأكد من توافق الإصدارات (Python/ETABS/CLR)، وفعّل سجلات مفصّلة عند الحاجة.


