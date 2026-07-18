# 📊 Enterprise Leave Management Hub (V7.3)

![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi)
![Pandas](https://img.shields.io/badge/pandas-%23150458.svg?style=for-the-badge&logo=pandas&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-07405E?style=for-the-badge&logo=sqlite&logoColor=white)
![Mint](https://img.shields.io/badge/Linux_Mint-9CE03F?style=for-the-badge&logo=linuxmint&logoColor=black)

نظام متكامل وعالي الكفاءة لإدارة وحساب أرصدة الإجازات والعطلات الرسمية لموظفي الشركات (سواء موظفي الكول سنتر والـ Avaya أو موظفي الإدارة الـ Non-Avaya). يعتمد النظام على معالجة مصفوفية فائقة السرعة (**Vectorized Matrix Processing**) لضمان دقة الحسابات الفورية بدون أي ثقل في الأداء.

---

## 🚀 الميزات الرئيسية (Core Features)

- **⚡ معالجة مصفوفية فورية (High-Performance Vectorization):** استخدام القوة الكاملة لـ `Pandas` و `NumPy` لحساب رصيد مئات الموظفين في أجزاء من الثانية بدلاً من تكرار الحسابات عبر الـ Loops التقليدية.
- **📅 مرونة قواعد الاحتساب المتقدمة:**
  - **العطلات الرسمية (Public Holidays Earned):** يتم احتسابها بدقة وتصفيتها من تاريخ الـ **Go-Live Date** الفعلي لكل موظف.
  - **الإجازات السنوية (Annual Leaves):** تُحسب وتُستقطع بناءً على تاريخ التعيين (**Hiring Date**) ووفقاً لتعديلات قوانين العمل المتغيرة (مثل قانون سبتمبر 2025).
- **👔 دعم الموظفين الإداريين (Non-Avaya Ledger):** نظام مرن يقبل رفع حضور موظفي الإدارة بناءً على الـ `UAE ID` مباشرة وتوليد الـ `ACD ID` تلقائياً لمنع أي تعارض في قاعدة البيانات.
- **🛡️ أمن واستقرار البيانات (Database Integration):** الاعتماد على محرك `SQLite3` مع توفير حماية كاملة ضد أخطاء الـ `KeyError` ومطابقة ذكية وديناميكية لأعمدة الملفات المرفوعة حتى لو احتوت على مسافات زائدة (`.strip()`).
- **🌌 واجهة مستخدم متطورة (Futuristic Dark UI):** واجهة مستخدم عصرية مدمج بها تأثيرات حركية، تدعم التبديل بين الوضع المظلم والمنير، ومزودة بشريط إخباري (Ticker) لمتابعة حالة النظام والتحكم في إغلاق السيرفر.

---

## 🏗️ البنية التكنولوجية (Tech Stack)

- **Backend:** FastAPI (Python 3.12)
- **Data Science:** Pandas & NumPy
- **Database:** SQLite3
- **Frontend:** HTML5, CSS3 (Glow & Glassmorphism Effects), Native JavaScript

---

## 🛠️ تشغيل المشروع (Installation & Running)

1. قم بإنشاء البيئة الافتراضية وتفعيلها:
```bash
python3 -m venv venv
source venv/bin/bin/activate
