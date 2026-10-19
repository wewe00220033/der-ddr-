دراسة هندسية واقتصادية متكاملة لاقتصاد Old School RuneScape (OSRS)
Economic Modeling & Resource Yield Optimization — بمنظومة Human-in-the-Loop نظيفة
تاريخ اللقطة: 18 أكتوبر 2026 (Africa/Cairo)
النطاق: رادار سوق لحظي (قراءة فقط) + تحسين مسارات إنتاج يدوية + خارطة رأس مال من الصفر إلى الـ Bond وما بعدها.
ملفات مرافقة في نفس المجلد:


ge_flip_radar.py
 — رادار بايثون مصحّح الضريبة، يعمل على API v2، مع وضع --demo ووضع حي.

demo_flips.csv
 — ناتج اختبار المنطق دون شبكة.

live_flips_f2p.csv
 / 
live_flips_p2p.csv
 — لقطتان حيّتان سُحبتا فعليًا أثناء إعداد الدراسة (تتقادمان خلال دقائق؛ أعد التشغيل للحصول على لقطة جديدة).
0) تصحيحان حاسمان قبل أي شيء — سكريبتكم الحالي يحتاج تحديثًا
0.1 الضريبة لم تعد 1% — أصبحت 2% منذ 29 مايو 2025
ضريبة الـ Grand Exchange (Convenience Fee) ارتفعت من 1% إلى 2% على كل عملية بيع خاضعة، تُقرَّب للأسفل floor، وبسقف 5,000,000 gp لكل عنصر واحد. عناصر تُباع بأقل من 50 gp ضريبتها صفر حسابيًا (floor(49×0.02)=0)، وتوجد قائمة إعفاءات (من أشهرها الـ Old School Bond). أي هامش محسوب بـ 1% في 2026 مبالغ فيه بنسبة 1% كاملة من سعر البيع — وهذا وحده يقلب كثيرًا من "الصفقات الرابحة" الرقيقة إلى خاسرة.

text

tax(sell)      = 0                                      إذا العنصر معفى
               = min( floor(0.02 × sell) , 5_000_000 )  غير ذلك
net_sell(sell) = sell − tax(sell)
margin         = net_sell(sell) − buy
ROI            = margin / buy
نقاط كسر يجب أن تكون Unit Tests عندكم: tax(49)=0، tax(50)=1، tax(250,000,000)=5,000,000، tax(500,000,000)=5,000,000 (معدل فعلي 1% فقط فوق السقف، و0.5% عند مليار). السكريبت المرافق يحتوي هذه الـ asserts فعليًا.

0.2 الـ API انتقل إلى v2 ومصدر بياناته تغيّر
نقطة النهاية الحالية هي prices.runescape.wiki/api/v2/osrs بمسارات /mapping و/latest و/5m و/1h و/timeseries (الأخير أصبح بـ lookback بدل timestep، والأسعار المتوسطة قد تعود بكسرين عشريين). وبيانات ما قبل 30 سبتمبر 2026 كانت من RuneLite فقط، أما الجديدة فمن Jagex مباشرة وتشمل كل العملاء — انتبهوا لأي مقارنة تاريخية عبر هذا التاريخ. والشرط الوحيد الصريح من فريق الويكي: User-Agent وصفي فيه وسيلة تواصل؛ الـ agents العامة مثل python-requests محظورة استباقيًا.

1) المحور الأول: أقوى مسارات جمع وإنتاج الموارد — Optimal High-Yield Pathways
1.1 كيف تقرأ الأرقام (حتى لا تخدعكم الجداول)
كل معدلات الـ GP/Hour أدناه لقطات من OSRS Wiki أثناء إعداد الدراسة (أكتوبر 2026)، محسوبة بأسعار السوق لحظتها وبعد ضريبة 2% حيثما ورد "after tax". وهي تتذبذب يوميًا، لذلك عاملوها كـ نطاقات للتخطيط وأعيدوا حسابها من الـ API قبل أي التزام. فرّقوا دائمًا بين نوعين:

Sustainable GP/h: طريقة يمكن تكرارها ساعة كاملة متواصلة (Blast Furnace، Vorkath…).
Effective GP/h (للطرق المتكررة Recurring): ربح دفعة قصيرة مقسومًا على دقائقها النشطة فقط. مثال: جولة أعشاب تربح ~193–207k في 6 دقائق كل ~90 دقيقة = "نحو 1.9–2.1M/h فعّال"، لكن سقفها اليومي محدود بعدد الجولات. خلط النوعين أشهر خطأ في نمذجة العائد.
عمود "انتظام النمط اليدوي" أدناه يقيس قابلية قياس وتنبؤ العائد عند لعب بشري طبيعي (ثبات زمن الدورة، تباين الناتج، حساسية الأخطاء) — وهو مدخل لنموذج العائد، وليس تقييمًا لصلاحية أي أتمتة لعب (انظر القسم 2.4).

1.2 جداول F2P (مجاني)
الصدارة النظرية في الويكي لحظة اللقطة:

الطريقة	GP/h (لقطة ويكي)	المتطلبات	كثافة الانتباه	ملاحظة هندسية
Smelting runite bars بـ Superheat Item	~744,000	Smithing 85 + Magic 43	متوسطة	معالجة برأس مال دوّار كبير؛ عائدها = فرق (runite ore + coal + nature rune) مقابل bar
Selling archery supplies (شراء من متاجر وإعادة بيع)	~702,000	لا شيء	متوسطة	Shop-hopping؛ ينهار مع المنافسة وتناقص مخزون المتاجر
High-alching بنود F2P مربحة	~626,000	Magic 55	منخفضة	أرباحه تحكمها معادلة: high_alch_value − item_buy − nature_rune > 0؛ حدود شراء 4h هي عنق الزجاجة
Mining runite ore (F2P)	~393,000	Mining 85+	متوسطة + خطر Wilderness	الخام ~10k للقطعة لكن المواقع قليلة/خطرة؛ تباين عالٍ
Making pie shells	~272,000	لا شيء	منخفضة	Processing بلا متطلبات — مناسب جدًا لحساب جديد
Collecting anti-dragon shields (F2P)	~267,000	لا شيء	متوسطة	Collecting متكرر منتظم
Buying team capes	~257,000	لا شيء	منخفضة	إعادة بيع متاجر؛ راقبوا تشبّع السوق
أما واقع الحساب الجديد (قبل Smithing 85 ومثيلاتها) فنطاقاته العملية من أدلة 2026 أقرب إلى: دبغ جلود الأبقار Cowhide tanning نحو 182k–319k/h بحسب استعادة الطاقة وسرعة الدورة (~2,000–3,500 جلدة/h)، وTelegrab Wine of Zamorak نحو 200k–350k/h عند Magic 33، وHill/Moss Giants نحو 100k–200k/h، وصيد Swordfish/Lobster وقطع Yew في نطاق 60k–120k/h لكن بانتباه منخفض. الخلاصة التخطيطية: F2P النظيف الواقعي للمبتدئ 150k–350k/h، والقفزة الحقيقية تأتي من Processing/Flipping برأس مال متنامٍ لا من الجمع الخام.

1.3 جداول P2P — مهارات ومعالجة وتجميع خام
الطريقة	GP/h بعد الضريبة (لقطة ويكي)	المتطلبات	رأس مال دوّار/ساعة	انتظام النمط وحساسية الخطأ
Blast Furnace — Runite bars (2,160 bar/h)	~2.56–2.76M	Smithing 85	مدخلات ~23.2M/h + رسم فرن 72k/h	دورات دقيقة منتظمة جدًا؛ خطأ ترتيب الفحم/الخام يفسد دفعة كاملة
Blast Furnace — Steel bars (5,400 bar/h)	~1.48–1.54M	Smithing 30 (60+ موصى به لتفادي رسم دخول) + Coal bag وStamina	مدخلات ~1.26M/h	أفضل نسبة عائد/رأس مال مبكرة؛ بحقيبة فحم ~5,400، دونها ~نصف الناتج
Blast Furnace — Mithril bars	~1.46–1.55M	Smithing 50 (60+ موصى به)	متوسط	بين الستيل والرونايت عائدًا ومخاطرة
Farming herbs — Ranarr (Recurring)	~193–207k لكل جولة 6 دقائق كل ~90 دقيقة (≈1.9–2.1M/h فعّال)	Farming 32 + رقع وتيليبورتات	بذور + Ultracompost	أعلى GP لكل دقيقة نشطة في اللعبة تقريبًا؛ EV تحكمه الغلة المتوقعة والأمراض
Bird house trapping (Recurring)	~61k لكل جولة 1.4 دقيقة كل 50 دقيقة (≈2.6M/h فعّال عند أعلى فئة)	Bone Voyage + Hunter (يتدرج 5→90)	بذور رخيصة + بيوت	تراكمي شبه سلبي؛ مثالي فوق أي نشاط آخر
Mining amethyst	~259k (80–100 قطعة/h)	Mining 92+	لا شيء تقريبًا	بطيء XP، انتباه منخفض جدًا، تباين ضئيل — "جمع خام" نظيف لكن سقفه منخفض
Catching anglerfish	~89k–179k (قاعدة ويكي ~173k عند 150/h)	Fishing 82+ (94+ موصى به) + Sandworms	طُعم ~16k/h	AFK-friendly؛ العائد وحده لا يبرر إلا مع XP أو استهلاك ذاتي
Catching raw karambwan	~146k عند ~80 Fishing (450→875/h نظريًا من 65 إلى 99 بكل التعزيزات)	Fishing 65+ + Fairytale II جزئيًا مفضّل	Karambwanji	مناسب للتدريب المربح ببطء، لا كمحرك رأس مال
Cooking raw karambwan (Processing)	~173–181k	Cooking 30 (90+ عمليًا لتقليل الحرق/1-tick اختياري)	~315k مدخلات/h	مثال كلاسيكي: المعالجة ترفع القيمة أكثر من الصيد الخام نفسه
قراءة هندسية: التعدين والصيد الخالصان (Amethyst/Anglerfish/Karambwan) سقفهما 150k–260k/h — قيمتهما في الـ XP والـ AFK والإمداد الذاتي، لا في مضاعفة الرصيد. قفزة العائد تأتي من سلاسل القيمة: خام → قضبان (Blast Furnace، خمسة أضعاف فأكثر)، وبذرة → عشبة (Herb runs)، ومواد رخصية → منتج استهلاكي سائل. أي نموذج Yield عندكم يجب أن يُسعّر السلسلة كاملة لا حلقة الجمع فقط.

1.4 جداول P2P — قتال/PvM (لاكتمال الصورة، لأنها السقف الفعلي)
الطريقة	GP/h بعد الضريبة (لقطة ويكي)	المتطلبات الجوهرية	طبيعة التباين
Vorkath بـ Dragon hunter lance (~30 قتلة/h)	~4.0–4.3M (مدخلات ~431k/h)	Dragon Slayer II + قتال عالٍ	منخفض التباين: قطرات ثابتة كل قتلة (Superior dragon bones…) — أنسب PvM للنمذجة الحتمية
Zulrah (20 قتلة/h متعلم ← حتى ~35–40 كفاءة قصوى)	~1.9M ← ~3.3M+	بدء Regicide + Ranged/Magic عالٍ	أعلى تباينًا: قيمة متوقعة ~130k/قتلة لكن الـ Uniques نادرة
غارات/نهايات عليا (ToB، ToA، Nex، زعماء DT2…)	~4M–25M بحسب المهارة والفريق والدروبات	إحصاءات وعتاد شبه قصوى	تباين شديد ومهارة بشرية؛ خارج أي نموذج "نمط منتظم"
1.5 معادلات العائد التي يجب أن يعيش بها نموذجكم
لأي طريقة Processing بمعدل إنتاج N وحدة/ساعة:

text

Profit/h = N × net_sell(P_out)                       # الإيراد بعد ضريبة 2%
         − Σ_i ( Qty_i/h × P_buy_i )                 # كل مدخل بسعر شرائه الفعلي
         − Fees/h                                    # رسوم فرن 72k، جرعات طاقة، تيليبورتات...
         − ExpectedLoss/h                            # موت Wilderness، حرق طبخ، مرض رقع...
تحليل حساسية (مثال Steel بـ N=5,400): ∂Profit/∂P_steel ≈ N×0.98 = 5,292 لكل 1 gp. أي انخفاض 10 gp في سعر القضيب يمحو ~52.9k/h، وانخفاض ~280 gp يصفّر الطريقة. راقبوا هامش الأمان هذا في الرادار لا السعر المطلق.
سعران لا سعر واحد: احسبوا نسختين دائمًا — متحفظة (شراء مدخلات Instant-Buy عند high وبيع ناتج Instant-Sell عند low) وصابرة (عروض واقفة قرب الطرفين). الحقيقة بينهما بحسب سيولة البند ووقتكم.
القيمة المتوقعة للرقعة (Herb): EV_patch = E[Yield] × net_sell(P_herb) − P_seed − P_compost − TeleportShare، مع E[Yield] من معادلة/حاسبة الويكي (مستوى + كمبوست + Magic secateurs + مذكرات)، ولا تستخدموا رقمًا ثابتًا مثل "6 أعشاب".
تكلفة الفرصة للوقت: أي طريقة تحت عتبتكم الحالية (مثلًا ما يدرّه الـ Flipping السلبي على رأس مالكم) تُرفض حتى لو "مربحة" مطلقًا.
2) المحور الثاني: قابلية الأتمتة البرمجية — ما هو مشروع، وما هو خط أحمر
2.1 المعمارية المقترحة: Data Pipeline + Human-in-the-Loop Copilot
الأتمتة المشروعة الوحيدة هي أتمتة المعلومة والقرار المساعد، بينما يبقى تنفيذ كل نقرة داخل اللعبة بشريًا بالكامل:

text

[Wiki API v2]                [RuneLite (لعب يدوي)]
 /mapping (يوميًا)             Flipping Utilities ──► ~/.runelite/flipping/<name>.json
 /latest  (كل 60–300ث)                    │ (قراءة ملف محلي تخصّكم)
 /1h , /5m (مجدولة)                       ▼
        │                       ┌─────────────────────┐
        ▼                       │ Trade Journal محلي   │
┌───────────────┐   features    │ أزواج FIFO + ضريبة  │
│ Ingestion     │──────────────►└─────────┬───────────┘
│ + User-Agent  │                         │
└──────┬────────┘                         ▼
       ▼                        ┌─────────────────────┐
┌───────────────┐               │ Backtest / Paper     │
│ Storage       │               │ Trading (محاكاة فقط) │
│ SQLite/DuckDB │               └─────────┬───────────┘
│ + Parquet     │                         │
└──────┬────────┘                         ▼
       ▼                        ┌─────────────────────┐
┌───────────────┐   قائمة مرتبة │ Dashboard + تنبيهات  │
│ Scoring Engine│──────────────►│ CSV / Discord / ويب  │
│ (قسم 2.2)     │               │ قرار بشري ◆ تنفيذ يدوي│
└───────────────┘               └─────────────────────┘
الاستيعاب: لا تضربوا /latest?id= في حلقة عن كل عنصر — الويكي يصرّح أن الطلب الجماعي الواحد أكفأ بمئة مرة للطرفين. /mapping يُخزّن ويُحدّث يوميًا، /1h و/5m بتردداتها الطبيعية، مع Backoff وكاش. لا حدود_RATE رسمية معلنة، لكن الاستخدام المهدِّد للاستقرار قد يُحجب.
التخزين: محليًا SQLite/DuckDB + لقطات Parquet مقسمة بالتاريخ يكفي لسنوات من بيانات 5-min. متعدد المستخدمين؟ PostgreSQL (+ Timescale اختياريًا). جداول جوهرية: items, latest_snapshots, bars_5m, bars_1h, my_offers, my_fills, alerts.
سجلّ صفقاتكم أنتم: إضافة Flipping Utilities (Plugin Hub) تحفظ عروضكم محليًا في JSON — قراءته ملفًا يخصّكم مشروعة تمامًا وتبني لكم FIFO دقيقًا للربح بعد الضريبة بدل تخمين الـ API العام.
2.2 محرك الترتيب — لماذا "أعلى هامش" قائمة خادعة، وما البديل
/latest ليس دفتر أوامر؛ هو آخر طبعة تداول لكل جهة. قائمة خام ستصدّر لكم فخاخًا: أثناء إعداد هذه الدراسة ظهر في لقطة حية Raw cod: low=31 مقابل high=1,699 (ROI شبحي بآلاف النِّسب من طبعة شاذة). لذلك صمّموا الترتيب طبقات:

text

# مرشحات صارمة أولًا
age(high), age(low) ≤ T_max                    # حداثة الطبعتين (افتراضي 30د)
volume_1h ≥ V_min                              # سيولة دنيا
high/low ≤ 1.35                                # رفض فروق شاذة
|latest − avg_1h| / avg_1h ≤ 25%               # انحراف عن متوسط الساعة
margin = net_sell(high) − low > 0  و  ROI ≥ ROI_min

# كمية قابلة للتنفيذ فعليًا خلال نافذة حد الشراء (4 ساعات)
Q_4h = min( BuyLimit,
            floor(Capital × MaxPositionFrac / low),     # سقف مخاطرة 35% افتراضيًا
            floor(k × Volume_1h × 4) )                  # k≈0.05–0.20 حصة سوق متحفظة

Profit_4h = margin × Q_4h
Profit/Slot-Hour = Profit_4h / 4
Score = Profit/Slot-Hour × 1/(1 + age_min/15) × log10(Volume_1h + 10)
وهذه بالضبط معادلات 
ge_flip_radar.py
 المرفق. ملاحظات تشغيل:

حدّ الشراء يُصفّر كل 4 ساعات لكل عنصر؛ والفتحات Slots: 3 لـ F2P و8 للأعضاء — لذا وحدة التحسين الحقيقية هي الربح لكل Slot-Hour لا هامش القطعة.
أضيفوا في النسخة التالية: تقلبًا من /timeseries (انحراف معياري/متوسط)، وعمق تنفيذ من اتفاق /5m مع /1h، و"حرارة منافسة" (تواتر تغير الأسعار)، وقائمة مراقبة Watchlist بسقوف شراء محسوبة MaxBuy = net_sell(target_sell) − target_profit.
Backtesting أمين: المحاكاة الورقية يجب أن تخصم الضريبة، وتحترم Buy Limit والفتحات وزمن تعبئة مستنتجًا من الحجم، وتفترض تعبئة جزئية — وإلا فهي تسويق ذاتي لا اختبار.
2.3 مرجعيات مفتوحة المصدر مشروعة (قراءة/عرض/تحليل فقط)
المشروع	الرابط	ماذا تدرسون منه
RuneLite (العميل)	https://github.com/runelite/runelite	الـ API الرسمية: أحداث مثل GrandExchangeOfferChanged وItemContainerChanged وOverlays وSide Panels
RuneLite Plugin Hub	https://github.com/runelite/plugin-hub	كيفية تقديم إضافة ومراجعتها؛ فهرس الإضافات المعتمدة
Example Plugin	https://github.com/runelite/example-plugin	قالب البداية الصحيح لأي إضافة عرض/تنبيه منكم
دليل المطور + الميزات المرفوضة	https://github.com/runelite/runelite/wiki/Developer-Guide و https://github.com/runelite/runelite/wiki/Rejected-or-Rolled-Back-Features	الحدود التفصيلية: ممنوع حقن الإدخال، وClient.menuAction مقيد بشدة، وممنوع Reflection/JNI/تنفيذ عمليات خارجية في إضافات الـ Hub
Flipping Utilities	https://github.com/Flipping-Utilities/rl-plugin	نموذج تتبع هامش وسجل صفقات محلي — مصدر بياناتكم الشخصية الأنظف
Wise Old Man	https://github.com/wise-old-man/wise-old-man (+ https://docs.wiseoldman.net)	معمارية تتبع تقدم لاعبين مفتوحة (Node/Postgres/Redis) — مرجع لهندسة API لا للسوق تحديدًا
وثائق أسعار الويكي	https://oldschool.runescape.wiki/w/RuneScape:Real-time_Prices	العقد الرسمي للبيانات وسياسة الاستخدام
حاسبات الويكي (Farming/Smithing…)	https://oldschool.runescape.wiki/w/Calculator:Farming/Herb_patch_expected_value (وأخواتها)	معادلات غلة وربح موثقة بدل إعادة اختراعها
إضافة RuneLite "نظيفة" منكم يمكنها قانونيًا أن: ترسم Overlay بهامش بعد الضريبة، تنبّه عند امتلاء عرض أو انتهاء زرع، تصدّر سجلّكم، وتعرض لوحة رادار جانبًا. هذه هي "قراءة البيانات" المشروعة كاملة.

2.4 خط أحمر صريح — لن أقدّم معماريات تحكّم ذاتي داخل اللعبة
طلبكم ذكر "تحليل شاشة" و"تحكمًا ذاتيًا نظيفًا". هنا يجب أن أكون مباشرًا كمعماري مسؤول: لا يوجد تحكم ذاتي نظيف داخل OSRS. أي منظومة — بوتات لون/شاشة (Color bots)، Auto-clickers، حقن أحداث فأرة/لوحة، عملاء Injection/Reflection، إضافات Sideloaded تنفّذ أفعالًا، أو "تأنيس إدخال" لخداع الكشف — تنتهك قواعد Jagex ضد الـ Macroing/البوتات وإرشادات عملاء الطرف الثالث، بينما RuneLite نفسه عميل معتمد وإضافات الـ Hub مراجعة تحديدًا لتتوافق مع تلك القواعد. الكشف سلوكي أساسًا، والعقوبات تصاعدية حتى الحظر الدائم وحظر حسابات مزرعة الذهب دون إنذار. لذلك لا أزوّد روابط أو تصاميم أو أكوادًا أو أساليب إخفاء لأي من ذلك، وأنصحكم ألا تبنوا فوق أصول (حسابات) قابلة للمصادرة بضغطة.

البديل الهندسي المكافئ وظيفيًا والمشروع: ما صُمم أعلاه — رادار يقرر ماذا يستحق انتباه الإنسان، وتنبيهات فورية، وحاسبات عائد لحظية، ومحاكاة ورقية، وقياس أدائكم اليدوي (زمن دورة، أخطاء/ساعة، عائد محقق مقابل نموذج) لتحسين المسار بشريًا. هذا يمنحكم 90% من القيمة التحليلية التي تبحثون عنها بصفر مخاطرة حساب.

3) المحور الثالث: من الصفر إلى مضاعفة الرصيد وتغطية الـ Bond
3.1 هدف متحرك — لا تثبّتوا سعر الـ Bond في أي خطة
الـ Bond (ID ‏13190، حد شراء 100، يُستبدل بـ 14 يوم عضوية) كانت لقطته على ويكي أكتوبر 2026 نحو 10.86M–11.41M gp بحسب الصفحة واللحظة، وتقارير أغسطس 2026 حول 11.4M. عاملوا الرقم كمتغير حي يُسحب من /latest?id=13190، واشتقوا منه معيار الاستدامة اليومي:

text

BondDailyBurn = BondPrice / 14        # عند 11.4M ≈ 814k gp/يوم
HoursToBond   = BondPrice / GP_per_hour
معدل واقعي	ساعات لبلوغ ~11.4M	الدلالة
250k/h (F2P مبكر نشط)	~46 ساعة	ممكن كتحدٍّ، مكلف زمنيًا
500k/h (F2P معالجة/قلب برأس مال متوسط)	~23 ساعة	عتبة الجدوى العملية
1.5M/h (P2P Steel Blast Furnace)	~7.6 ساعة	بعد أول Bond تصبح الاستدامة سهلة
4M/h (Vorkath)	~2.9 ساعة	مرحلة تراكم لا بقاء
ملاحظة صدق اقتصادي: إن كانت قيمة وقتكم الحقيقي مهمة، فشراء أول عضوية نقدًا ثم استدامتها بالذهب أرخص زمنًا بكثير من طحن F2P — لكن إن كان مشروعكم بحثيًا/تحديًا من الصفر، فالخارطة التالية مصممة لذلك.

3.2 الخارطة على مراحل (رأس مال → أدوات → قاعدة مخاطرة)
المرحلة أ: 0 → 100k (ساعات)
أموال البداية المجانية (Stronghold of Security ونحوها) + Pie shells / دبغ Cowhide / بيع مستلزمات متاجر منتظمة. ممنوع الـ Flipping بعد: لا رأس مال يحمِل رسوم تعلّم. الهدف الثاني بالتوازي: رفع Magic نحو 33 ثم 55 وSmithing نحو 30 استعدادًا لما بعد الـ Bond — لا تهدروا أيام العضوية لاحقًا في رفع أساسيات F2P.

المرحلة ب: 100k → 1M
نشاط نشط 200k–350k/h (دبغ منضبط الطاقة، Wine of Zamorak عند توفر الخوادم، Giants مع Alching للقطرات القيمة) + أول قلب GE تعليمي: بنود F2P فائقة السيولة (رونيات، أسهم، أطعمة، جلود) بهامش بعد ضريبة 2% موجب وحجم ساعة مرتفع. قواعد صارمة: فتحاتكم 3 فقط؛ حد أقصى 25% من الرصيد في بند واحد؛ اختبار هامش Margin-check بوحدة واحدة شراءً وبيعًا فوريين قبل أي كمية؛ خروج فوري عند انعكاس الهامش بدل "الانتظار حتى يعود".

المرحلة ج: 1M → Bond (~11.4M)
محرك مزدوج: (1) جلسات نشطة بأعلى معدل متاح لكم (معالجات الويكي الأعلى مما تملكون متطلباته فعلًا)، و(2) قلب صابر Overnight بعروض واقفة يعمل أثناء انشغالكم بالنشط. مؤشر الأداء الوحيد: ΔBank/day لا GP/h نظري. أعيدوا استثمار كل شيء إلا احتياطي أدوات؛ ولا تشتروا "عتاد أحلام" قبل الـ Bond. إذا تباطأ التقدم تحت ~400k/يوم، فالمشكلة اختيار طريقة لا ساعات إضافية.

المرحلة د: أول 14 يوم عضوية — لا تُحرق
قبل تفعيل الـ Bond جهّزوا قائمة فتح فوري: رقع أعشاب وتيليبورتاتها، Bone Voyage لبيوت الطيور، وSmithing 30/60 + Coal bag route لـ Blast Furnace، ومهام نقل أساسية. روتين الاستدامة اليومي المستهدف (~>815k فائض/يوم عند سعر اللقطة):

text

جلتا أعشاب      ≈ 2 × ~200k   = ~400k   (12 دقيقة نشطة)
جلتا بيوت طيور  ≈ 2 × ~40-60k = ~80-120k (3 دقائق)
30د Steel BF    ≈ 0.5 × ~1.5M = ~750k
المجموع قبل أي Flipping ≈ 1.23M+/يوم  ← فائض فوق Burn الـ Bond
عندها يتحول الـ Flipping بـ 8 فتحات وبنود أعضاء عالية السيولة (مواد Herblore/Prayer، ذخائر، أطعمة PvM، قضبان وخامات Blast Furnace نفسها) إلى مسرّع، لا إلى وسيلة بقاء.

المرحلة هـ: مضاعفة وتوسيع (Bond+ → 50M → 250M+)

وزّعوا الفتحات الثماني محفظيًا: 4 قلب سريع عالي حجم، و2 قلب صابر أعمق هامشًا، و2 لمدخلات/مخرجات سلسلة إنتاجكم (شراء خام صابرًا وبيع قضبان صابرًا يرفع عائد Blast Furnace نفسه فوق أرقام الويكي الفورية).
حجم مركز منضبط بصيغة كسرية شبيهة Kelly متحفظة: لا تتجاوزوا 10–20% في بند لحساب كبير (35% في كود البداية للحسابات الصغيرة)، واحتفظوا ≥20% نقدًا سائلًا لاقتناص انهيارات Dumps بعد التحديثات.
فوق 250M للقطعة يبدأ سقف الضريبة 5M بخفض المعدل الفعلي عن 2% — هناك تصبح البنود الفاخرة أكفأ ضريبيًا، لكن بسيولة أرق ومخاطر فجوات أعلى؛ لا تدخلوها قبل سجل محاكاة ورقية موثق.
راجعوا أسبوعيًا: ربح محقق بعد الضريبة لكل Slot-Hour، ونسبة تعبئة، وانحراف النموذج عن الواقع. البند الذي يخسر مرتين متتاليتين يُحظر من الرادار أسبوعًا — انضباط آلي في التحليل لا في اللعب.
3.3 أخطاء تقتل المنظومة (من واقع البيانات نفسها)
بقاء ضريبة 1% في الكود. 2) ترتيب بالهامش الخام دون سيولة/حداثة (فخ Raw cod أعلاه). 3) افتراض تعبئة كامل حد الشراء دائمًا. 4) خلط Effective وSustainable GP/h. 5) تجاهل تكلفة مدخلات بسعر فوري مقابل صابر. 6) مقارنة سلاسل زمنية عبر 30 سبتمبر 2026 دون ملاحظة تغير مصدر البيانات. 7) أي اختصار أتمتة لعب — مخاطرة وجودية لا مخاطرة سوق.
4) خطة تنفيذ من أربع خطوات لفريقكم (أسبوعان)
إصلاح النموذج (يوم): ضريبة 2% + سقف 5M + إعفاءات كجدول إعدادات، وUnit Tests نقاط الكسر، والهجرة إلى API v2 مع User-Agent وصفي حقيقي.
الرادار v1 (أيام): شغّلوا 
ge_flip_radar.py
 حيًا مجدولًا، خزّنوا كل لقطة، وأضيفوا /5m و/timeseries لخصائص التقلب. ابدأوا: OSRS_USER_AGENT="اسم-مشروعكم/1.0 (contact: بريدكم)" python3 ge_flip_radar.py --capital 10000000 --slots 3 --out flips.csv.
دفتر يوميات ومحاكاة (أسبوع): استيراد JSON من Flipping Utilities، مطابقة FIFO، وPaper-trading يحترم الفتحات وحدود الشراء — قارنوا النموذج بالمحقق أسبوعيًا.
لوحة وقياس مسارات (أسبوع ثانٍ): Dashboard بسيط (Streamlit/FastAPI) يعرض أعلى Profit/Slot-Hour وسلاسل إنتاج (Steel/Runite/Herb EV) محدثة حية، وسجّل زمن دوراتكم اليدوية لمعايرة N الفعلية في معادلة العائد.
مراجع مباشرة
ضريبة الـ GE الحالية وتاريخ رفعها: https://oldschool.runescape.wiki/w/Grand_Exchange
دليل كسب المال (عام): https://oldschool.runescape.wiki/w/Money_making_guide — F2P: https://oldschool.runescape.wiki/w/Money_making_guide/Free-to-play — متكرر (أعشاب/طيور): نفس الدليل بقسم Recurring
Blast Furnace: https://oldschool.runescape.wiki/w/Money_making_guide/Smelting_steel_bars_at_Blast_Furnace و https://oldschool.runescape.wiki/w/Money_making_guide/Smelting_runite_bars_at_Blast_Furnace
تعدين/صيد: https://oldschool.runescape.wiki/w/Money_making_guide/Mining_amethyst و https://oldschool.runescape.wiki/w/Money_making_guide/Catching_anglerfish و https://oldschool.runescape.wiki/w/Money_making_guide/Catching_raw_karambwan
PvM: https://oldschool.runescape.wiki/w/Money_making_guide/Killing_Vorkath_(Dragon_hunter_lance) و https://oldschool.runescape.wiki/w/Money_making_guide/Killing_Zulrah
API الأسعار v2: https://oldschool.runescape.wiki/w/RuneScape:Real-time_Prices
الـ Bond: https://oldschool.runescape.wiki/w/Old_school_bond وصفحة تبادله الحية https://oldschool.runescape.wiki/w/Exchange:Old_school_bond
تطوير RuneLite وحدوده: https://github.com/runelite/runelite/wiki/Developer-Guide و https://github.com/runelite/runelite/wiki/Rejected-or-Rolled-Back-Features و https://runelite.net/plugin-hub/
تنبيه ختامي: كل الأرقام لقطات سوق متحركة بتاريخ الدراسة وليست وعود عائد؛ أعادوا الحساب من بيانات حية، واختبرا هامش كل بند بوحدة واحدة داخل اللعبة قبل توظيف رأس مال معتبر.







rank	item_id	name	members	buy_at_or_below(low)	sell_at_or_above(high)	tax_per_item	margin_after_tax	roi	volume_1h	age_min	buy_limit	qty_model_4h	profit_model_4h	profit_per_slot_hour	score	max_buy_for_1gp_profit
1	4151	Abyssal whip	True	1450000	1560000	31200	78800	0.0543	1700	5.0	70	12	945600	236400.0	573210.21	1528799
2	560	Death rune	False	195	215	4	16	0.0821	750000	3.0	25000	25000	400000	100000.0	489588.92	210
3	13190	Old School Bond	False	11000000	11400000	0	400000	0.0364	480	15.0	100	1	400000	100000.0	134509.8	11399999








#!/usr/bin/env python3
"""
OSRS GE Flip Radar — read-only market analytics (human-in-the-loop).

What this script does
---------------------
- Pulls public data only from OSRS Wiki Real-Time Prices API v2:
  /mapping, /latest, /1h
- Applies the CURRENT Grand Exchange tax: 2%, floor, capped 5,000,000 gp/item.
  (It was raised from 1% to 2% on 29 May 2025. Do NOT use 1% in 2026.)
- Ranks flips by a liquidity-adjusted, capital-aware score, not raw margin.
- Exports a CSV for manual review. It never logs in, never sends input to the
  game, never interacts with RuneLite at runtime.

Usage
-----
1) Set a descriptive User-Agent (REQUIRED by the Wiki API policy):
   export OSRS_USER_AGENT="my-ge-radar/1.0 (contact: you@example.com)"

2) Live run:
   python ge_flip_radar.py --capital 5000000 --slots 8 --members \
       --min-roi 0.02 --min-volume-1h 500 --max-age-min 30 --out flips.csv

3) Offline logic test (no network):
   python ge_flip_radar.py --demo --out demo_flips.csv

Notes
-----
- /latest high = last instant-buy price (what a seller received / buyer paid
  to buy instantly), low = last instant-sell price. A patient flip buys near
  `low`-side and sells near `high`-side. These are last-trade prints, not a
  live order book, so always margin-check in game before committing capital.
- Some items are GE-tax exempt (e.g. Old School Bond, ID 13190). Keep
  EXEMPT_IDS maintained from the Wiki; everything else uses 2%.
"""

from __future__ import annotations

import argparse
import csv
import json
import math
import os
import sys
import time
import urllib.request
from dataclasses import dataclass
from typing import Any

API = "https://prices.runescape.wiki/api/v2/osrs"
TAX_RATE = 0.02
TAX_CAP = 5_000_000
EXEMPT_IDS = {13190}  # Old School Bond. Verify/extend against the Wiki list.


def ge_tax(sell_price: int, item_id: int | None = None) -> int:
    """2% seller tax, rounded down, capped at 5m per item. <50 gp => 0."""
    if item_id in EXEMPT_IDS or sell_price <= 0:
        return 0
    return min(int(sell_price * TAX_RATE), TAX_CAP)  # int() floors for gp >= 0


def net_sell(sell_price: int, item_id: int | None = None) -> int:
    return sell_price - ge_tax(sell_price, item_id)


def min_sell_for_target(buy_price: int, target_profit: int, item_id: int | None = None) -> int:
    """Smallest integer sell price whose after-tax proceeds >= buy + target."""
    need = buy_price + target_profit
    if item_id in EXEMPT_IDS:
        return need
    # Start from the linear approximation, then walk to the exact integer
    # (handles the floor() and the 250m+ cap region correctly).
    s = max(1, math.ceil(need / (1 - TAX_RATE)) - 2)
    if need + TAX_CAP < s:  # cap region
        s = need + TAX_CAP
    while net_sell(s, item_id) < need:
        s += 1
    while s > 1 and net_sell(s - 1, item_id) >= need:
        s -= 1
    return s


def fetch_json(path: str, user_agent: str, timeout: int = 30) -> Any:
    req = urllib.request.Request(
        f"{API}{path}", headers={"User-Agent": user_agent, "Accept": "application/json"}
    )
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return json.loads(r.read().decode("utf-8"))


@dataclass
class Opportunity:
    item_id: int
    name: str
    members: bool
    buy_limit: int
    high: int          # last instant-buy print  -> patient sell target zone
    low: int           # last instant-sell print -> patient buy target zone
    age_min: float
    volume_1h: int
    tax: int
    margin: int        # net_sell(high) - low
    roi: float
    qty_4h: int
    profit_4h: int
    profit_per_slot_hour: float
    score: float


def rank(mapping: list[dict], latest: dict, hourly: dict, *, capital: int,
         slots: int, members: bool, min_roi: float, min_volume_1h: int,
         max_age_min: float, market_share: float, max_position_frac: float,
         max_spread_ratio: float, max_avg_deviation: float,
         now: int) -> list[Opportunity]:
    meta = {int(m["id"]): m for m in mapping}
    latest_data: dict = latest.get("data", latest)
    hourly_data: dict = hourly.get("data", hourly)
    # Position-size cap (risk control), NOT capital/slots: one GE offer may
    # legitimately use more than an equal slot share, but never the whole stack.
    position_capital = capital * max_position_frac
    out: list[Opportunity] = []

    for sid, p in latest_data.items():
        item_id = int(sid)
        m = meta.get(item_id)
        if not m:
            continue
        if (not members) and m.get("members"):
            continue
        high, low = p.get("high"), p.get("low")
        if not high or not low or high <= 0 or low <= 0:
            continue
        # Freshness: both prints must be recent enough to trust.
        ht, lt = p.get("highTime") or 0, p.get("lowTime") or 0
        age_min = max(now - ht, now - lt) / 60.0
        if age_min > max_age_min:
            continue
        h = hourly_data.get(sid) or hourly_data.get(item_id) or {}
        vol_1h = int(h.get("highPriceVolume") or 0) + int(h.get("lowPriceVolume") or 0)
        if vol_1h < min_volume_1h:
            continue
        # --- Outlier / manipulation guard ---------------------------------
        # /latest is a LAST-PRINT feed: a single junk trade can print e.g.
        # high=1699 vs low=31. Never trust a spread wildly wider than the
        # 1h averages. This filter is what separates a radar from a trap list.
        if int(high) / int(low) > max_spread_ratio:
            continue
        avg_high, avg_low = h.get("avgHighPrice"), h.get("avgLowPrice")
        if avg_high and abs(int(high) - float(avg_high)) / float(avg_high) > max_avg_deviation:
            continue
        if avg_low and abs(int(low) - float(avg_low)) / float(avg_low) > max_avg_deviation:
            continue

        tax = ge_tax(int(high), item_id)
        margin = int(high) - tax - int(low)
        if margin <= 0:
            continue
        roi = margin / int(low)
        if roi < min_roi:
            continue

        buy_limit = int(m.get("limit") or 0)
        affordable = int(position_capital // int(low))
        # Conservative fill model: you can only capture a slice of flow.
        flow_4h = int(vol_1h * 4 * market_share)
        qty = min(x for x in (buy_limit if buy_limit else 10**9, affordable, flow_4h) if x is not None)
        if qty <= 0:
            continue
        profit_4h = margin * qty
        ppsh = profit_4h / 4.0
        # Score favours slot-hour profit, penalises stale prints & thin flow.
        freshness = 1.0 / (1.0 + age_min / 15.0)
        score = ppsh * freshness * math.log10(vol_1h + 10)
        out.append(Opportunity(
            item_id=item_id, name=m.get("name", "?"), members=bool(m.get("members")),
            buy_limit=buy_limit, high=int(high), low=int(low), age_min=round(age_min, 1),
            volume_1h=vol_1h, tax=tax, margin=margin, roi=round(roi, 4), qty_4h=qty,
            profit_4h=profit_4h, profit_per_slot_hour=round(ppsh, 1), score=round(score, 2),
        ))
    return sorted(out, key=lambda x: x.score, reverse=True)


def demo_data() -> tuple[list[dict], dict, dict, int]:
    now = int(time.time())
    mapping = [
        {"id": 560, "name": "Death rune", "members": False, "limit": 25000},
        {"id": 4151, "name": "Abyssal whip", "members": True, "limit": 70},
        {"id": 13190, "name": "Old School Bond", "members": False, "limit": 100},
    ]
    latest = {"data": {
        "560": {"high": 215, "highTime": now - 120, "low": 195, "lowTime": now - 180},
        "4151": {"high": 1_560_000, "highTime": now - 300, "low": 1_450_000, "lowTime": now - 240},
        "13190": {"high": 11_400_000, "highTime": now - 600, "low": 11_000_000, "lowTime": now - 900},
    }}
    hourly = {"data": {
        "560": {"highPriceVolume": 400_000, "lowPriceVolume": 350_000},
        "4151": {"highPriceVolume": 900, "lowPriceVolume": 800},
        "13190": {"highPriceVolume": 300, "lowPriceVolume": 180},
    }}
    return mapping, latest, hourly, now


def main() -> int:
    ap = argparse.ArgumentParser(description="OSRS GE flip radar (read-only, 2% tax).")
    ap.add_argument("--capital", type=int, default=5_000_000, help="Liquid gp available for flipping")
    ap.add_argument("--slots", type=int, default=8, help="GE slots: 3 F2P, 8 members")
    ap.add_argument("--members", action="store_true", help="Include members items")
    ap.add_argument("--min-roi", type=float, default=0.02)
    ap.add_argument("--min-volume-1h", type=int, default=500)
    ap.add_argument("--max-age-min", type=float, default=30.0)
    ap.add_argument("--market-share", type=float, default=0.10,
                    help="Conservative share of hourly flow you assume you can fill (0-1)")
    ap.add_argument("--max-position-frac", type=float, default=0.35,
                    help="Max fraction of capital in a single item (risk cap)")
    ap.add_argument("--max-spread-ratio", type=float, default=1.35,
                    help="Discard if latest high/low exceeds this (outlier guard)")
    ap.add_argument("--max-avg-deviation", type=float, default=0.25,
                    help="Discard if latest print deviates from 1h average by more than this")
    ap.add_argument("--top", type=int, default=50)
    ap.add_argument("--out", default="flips.csv")
    ap.add_argument("--demo", action="store_true", help="Run on bundled sample data (no network)")
    a = ap.parse_args()

    if a.demo:
        mapping, latest, hourly, now = demo_data()
    else:
        ua = os.environ.get("OSRS_USER_AGENT", "").strip()
        if not ua or ua.lower() in {"python-requests", "python-urllib"} or "contact" not in ua.lower():
            print('ERROR: set OSRS_USER_AGENT to a descriptive value, e.g.\n'
                  '  export OSRS_USER_AGENT="my-ge-radar/1.0 (contact: you@example.com)"',
                  file=sys.stderr)
            return 2
        mapping = fetch_json("/mapping", ua)
        latest = fetch_json("/latest", ua)
        hourly = fetch_json("/1h", ua)
        now = int(time.time())

    rows = rank(mapping, latest, hourly, capital=a.capital, slots=a.slots,
                members=a.members or a.demo, min_roi=a.min_roi,
                min_volume_1h=a.min_volume_1h if not a.demo else 0,
                max_age_min=a.max_age_min, market_share=a.market_share,
                max_position_frac=a.max_position_frac,
                max_spread_ratio=a.max_spread_ratio,
                max_avg_deviation=a.max_avg_deviation, now=now)[: a.top]

    with open(a.out, "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["rank", "item_id", "name", "members", "buy_at_or_below(low)",
                    "sell_at_or_above(high)", "tax_per_item", "margin_after_tax",
                    "roi", "volume_1h", "age_min", "buy_limit", "qty_model_4h",
                    "profit_model_4h", "profit_per_slot_hour", "score",
                    "max_buy_for_1gp_profit"])
        for i, r in enumerate(rows, 1):
            w.writerow([i, r.item_id, r.name, r.members, r.low, r.high, r.tax, r.margin,
                        r.roi, r.volume_1h, r.age_min, r.buy_limit, r.qty_4h,
                        r.profit_4h, r.profit_per_slot_hour, r.score,
                        net_sell(r.high, r.item_id) - 1])
            print(f"{i:>3} {r.name:<22} buy<={r.low:>11,} sell>={r.high:>11,} "
                  f"tax={r.tax:>8,} margin={r.margin:>8,} roi={r.roi:6.2%} "
                  f"vol1h={r.volume_1h:>8,} model4h={r.profit_4h:>10,}")
    print(f"\nWrote {len(rows)} rows -> {a.out}  (tax model: 2% capped 5m; verify in-game margin check)")
    # Self-check examples pinned to documented tax breakpoints.
    assert ge_tax(49) == 0 and ge_tax(50) == 1 and ge_tax(250_000_000) == 5_000_000
    assert ge_tax(500_000_000) == 5_000_000 and ge_tax(11_400_000, 13190) == 0
    return 0


if __name__ == "__main__":
    raise SystemExit(main())







rank	item_id	name	members	buy_at_or_below(low)	sell_at_or_above(high)	tax_per_item	margin_after_tax	roi	volume_1h	age_min	buy_limit	qty_model_4h	profit_model_4h	profit_per_slot_hour	score	max_buy_for_1gp_profit
1	1654	Gold necklace	False	139	146	2	5	0.036	85126	1.4	18000	18000	90000	22500.0	101149.74	143
2	1639	Emerald ring	False	538	570	11	21	0.039	12232	1.0	10000	4892	102732	25683.0	98324.12	558
3	225	Limpwurt root	False	210	220	4	6	0.0286	62885	0.9	13000	13000	78000	19500.0	88461.88	215
4	554	Fire rune	False	5	6	0	1	0.2	7634306	1.2	50000	50000	50000	12500.0	79416.58	5
5	555	Water rune	False	4	5	0	1	0.25	3905848	0.6	50000	50000	50000	12500.0	79142.81	4
6	556	Air rune	False	5	6	0	1	0.2	6459509	1.2	50000	50000	50000	12500.0	78821.76	5
7	563	Law rune	False	120	125	2	3	0.025	1027362	0.8	18000	18000	54000	13500.0	77293.64	122
8	557	Earth rune	False	5	6	0	1	0.2	3826007	1.1	50000	50000	50000	12500.0	76425.08	5
9	1741	Leather	False	193	210	4	13	0.0674	18916	5.8	13000	7566	98358	24589.5	75905.1	205
10	1635	Gold ring	False	146	156	3	7	0.0479	25716	1.6	18000	10286	72002	18000.5	71953.69	152








rank	item_id	name	members	buy_at_or_below(low)	sell_at_or_above(high)	tax_per_item	margin_after_tax	roi	volume_1h	age_min	buy_limit	qty_model_4h	profit_model_4h	profit_per_slot_hour	score	max_buy_for_1gp_profit
1	12905	Anti-venom(4)	True	8516	10427	208	1703	0.2	3521	9.3	2000	1027	1748981	437245.2	955627.16	10218
2	33589	Seeking amethyst arrow	True	507	560	11	42	0.0828	42119	3.4	0	16847	707574	176893.5	667500.01	548
3	27616	Ancient essence	True	18	19	0	1	0.0556	3306153	2.2	300000	300000	300000	75000.0	426409.29	18
4	33625	Letvek	True	1725	1849	36	88	0.051	14797	3.9	0	5072	446336	111584.0	368681.04	1812
5	32336	Halibut	True	2476	2625	52	97	0.0392	49181	5.1	10000	3533	342701	85675.2	299983.94	2572
6	235	Unicorn horn dust	True	815	977	19	143	0.1755	11570	21.7	13000	4628	661804	165451.0	274675.52	957
7	534	Babydragon bones	True	1049	1201	24	128	0.122	6109	1.5	3000	2443	312704	78176.0	269388.05	1176
8	1777	Bow string	True	211	234	4	19	0.09	237835	3.7	13000	13000	247000	61750.0	266774.59	229
9	72	Magic shortbow (u)	True	505	554	11	38	0.0752	18274	1.7	10000	7309	277742	69435.5	266079.06	542
10	29143	Cooked moonlight antelope	True	2615	2813	56	142	0.0543	12906	13.5	6000	3346	475132	118783.0	257317.86	2756
11	11229	Dragon arrow(p++)	True	2054	2196	43	99	0.0482	8353	5.5	11000	3341	330759	82689.8	237514.42	2152
12	1753	Green dragonhide	True	1342	1415	28	45	0.0335	22898	5.2	13000	6520	293400	73350.0	237283.23	1386
13	21948	Dragonstone dragon bolts (e)	True	2045	2227	44	138	0.0675	6357	6.8	11000	2542	350796	87699.0	230069.85	2182
14	217	Grimy dwarf weed	True	1230	1290	25	35	0.0285	40804	4.2	11000	7113	248955	62238.8	224586.0	1264
15	6016	Cactus spine	True	1483	1553	31	39	0.0263	14117	1.9	11000	5646	220194	55048.5	203170.58	1521















ممتاز — سأبني الدراسة على بيانات حية من الويكي والسوق، مع فصل واضح بين الأتمتة التحليلية المشروعة وأي تحكم آلي داخل اللعبة يُعرّض الحساب للحظر.

وجدت تصحيحًا مهمًا: ضريبة الـ GE أصبحت 2% منذ مايو 2025 وليست 1% — سأتحقق من معدلات الربح الحالية من الويكي مباشرة.

الآن سأستكمل بيانات التعدين والصيد تحديدًا، ثم أجهّز لك الدراسة وكود رادار السوق المصحح.

الكود اشتغل لكن بيانات التجربة بهوامش أضيق من حد الـ ROI — سأصحّحها وأعيد الاختبار.

ممتاز — الرادار يعمل على البيانات الحية فعلًا. سأسحب أيضًا قائمة للأعضاء P2P للمقارنة قبل كتابة الدراسة النهائية.

الرادار الآن يفلتر الصفقات الشاذة. سأكتب الدراسة الكاملة بالعربية مع الجداول والمعادلات والمراجع.

# دراسة OSRS الاقتصادية والهندسية — التسليم التنفيذي

جهّزت لكم الدراسة الكاملة في ملف **`OSRS_Economic_Modeling_Study_AR.md`** داخل مجلد `osrs_study`، وأرفقت معها أداة عملية جاهزة للتشغيل:

- **`ge_flip_radar.py`** — رادار سوق بايثون (قراءة فقط) على API v2، بضريبة مصحّحة، وفلاتر سيولة وحداثة وشذوذ، وترتيب حسب ربح Slot-Hour لا الهامش الخام.
- **`demo_flips.csv`** — ناتج اختبار منطق دون شبكة (تم التحقق منه).
- **`live_flips_f2p.csv` / `live_flips_p2p.csv`** — لقطتان حيّتان سحبتهما فعلًا من الـ API بتاريخ 18 أكتوبر 2026 أثناء الإعداد (تتقادمان خلال دقائق — أعد التشغيل دائمًا).

وفيما يلي خلاصة الدراسة كاملة بأهم الجداول والمعادلات.

---

## 0) تصحيحان حاسمان لسكريبتكم الحالي

### 0.1 الضريبة 2% وليست 1%
ضريبة الـ Grand Exchange ارتفعت من 1% إلى **2% في 29 مايو 2025**، تُقرّب للأسفل وبسقف 5,000,000 gp لكل عنصر، وما يُباع بأقل من 50 gp صفر حسابيًا، وتوجد إعفاءات منها الـ Bond [2](https://oldschool.runescape.wiki/w/Grand_Exchange). أي هامش محسوب بـ 1% في 2026 مبالغ فيه بنسبة 1% من سعر البيع كاملة، وهذا يقلب أغلب الصفقات الرقيقة إلى خاسرة.

```
tax(sell)      = 0                                        إذا معفى
               = min( floor(0.02 × sell) , 5_000_000 )    غير ذلك
net_sell(sell) = sell − tax(sell)
margin         = net_sell(sell) − buy
ROI            = margin / buy
MaxBuy (لهدف ربح P عند سعر بيع مستهدف S) = net_sell(S) − P
```

نقاط اختبار ملزمة لكم: `tax(49)=0` ، `tax(50)=1` ، `tax(250M)=5M` ، `tax(500M)=5M` (المعدل الفعلي فوق السقف يهبط إلى 1% عند 500M و0.5% عند مليار [4](https://www.grandexchanger.com/ge-calculator)). هذه الـ asserts موجودة فعلًا داخل السكريبت المرفق.

### 0.2 الـ API على v2 ومصدر بياناته تغيّر
المسار الحالي `prices.runescape.wiki/api/v2/osrs` (`/mapping`, `/latest`, `/5m`, `/1h`, `/timeseries` بـ `lookback`)، وبيانات ما قبل **30 سبتمبر 2026** كانت من RuneLite فقط بينما الجديدة من Jagex لكل العملاء — احذفوا أي مقارنة زمنية تعبر هذا التاريخ دون تنبيه [1](https://oldschool.runescape.wiki/w/RuneScape:Real-time_Prices). والشرط الوحيد الصريح: User-Agent وصفي فيه وسيلة تواصل؛ الـ agents العامة مثل `python-requests` محظورة استباقيًا، وطلب جماعي واحد بـ `/latest` أكفأ بمئة مرة من حلقة طلبات لكل عنصر [1](https://oldschool.runescape.wiki/w/RuneScape:Real-time_Prices).

---

## 1) المحور الأول: مسارات العائد الأعلى (Optimal High-Yield Pathways)

تنبيه منهجي قبل الجداول: فرّقوا بين **Sustainable GP/h** (قابل للتكرار ساعة كاملة) و**Effective GP/h** للطرق المتكررة Recurring (ربح دفعة ÷ دقائقها النشطة فقط، وسقفها اليومي محدود). عمود "انتظام النمط" يقيس ثبات العائد والقابلية للتنبؤ عند لعب يدوي — مدخل لنموذج العائد، ولا علاقة له بأي أتمتة لعب (انظر 2.4).

### F2P — صدارة الويكي النظرية (لقطة أكتوبر 2026) [6](https://oldschool.runescape.wiki/w/Money_making_guide/Free-to-play)

| الطريقة | GP/h | المتطلبات | ملاحظة |
|---|---:|---|---|
| Smelting runite bars (Superheat) | ~744k | Smithing 85 + Magic 43 | Processing برأس مال دوار كبير |
| Selling archery supplies | ~702k | لا شيء | Shop-hopping، ينهار مع المنافسة |
| High-alching بنود مربحة | ~626k | Magic 55 | القيد الحقيقي: Buy Limit لكل 4 ساعات |
| Mining runite ore | ~393k | Mining 85+ | مواقع F2P قليلة وWilderness خطرة |
| Making pie shells / Anti-dragon shields / Team capes | 257k–272k | لا شيء | نقاط بداية حساب جديد النظيفة |

واقع الحساب الجديد قبل متطلبات 85+: دبغ Cowhide نحو 182k–319k/h حسب إدارة الطاقة [8](https://probemas.com/blog/easy-osrs-money-making-guide)، وWine of Zamorak بـ Telegrab نحو 200k–350k/h عند Magic 33، والعمالقة 100k–200k/h. **الخلاصة: F2P المبكر الواقعي 150k–350k/h.**

### P2P — معالجة ومهارات (حيث القفزة الحقيقية)

| الطريقة | GP/h بعد ضريبة 2% | المتطلبات | انتظام/حساسية |
|---|---:|---|---|
| Blast Furnace — Runite (2,160/h) | ~2.56–2.76M [2](https://oldschool.runescape.wiki/w/Money_making_guide/Smelting_runite_bars_at_Blast_Furnace) | Smithing 85، مدخلات ~23.2M/h، رسم فرن 72k/h | دورات دقيقة جدًا؛ خطأ فحم يفسد دفعة |
| Blast Furnace — Steel (5,400/h) | ~1.48–1.54M [1](https://oldschool.runescape.wiki/w/Money_making_guide/Smelting_steel_bars_at_Blast_Furnace) | Smithing 30 (60+ مفضل)، مدخلات ~1.26M/h فقط | أفضل عائد/رأس مال مبكر في اللعبة |
| Farming Ranarr (Recurring) | ~193–207k لكل 6 دقائق كل ~90د (≈2M/h فعّال) [2](https://oldschool.runescape.wiki/w/Money_making_guide/Farming_herbs) | Farming 32 + تيليبورتات | أعلى GP لكل دقيقة نشطة تقريبًا |
| Bird houses (Recurring) | ~61k لكل 1.4 دقيقة كل 50د (≈2.6M/h فعّال أقصى) [9](https://oldschool.runescape.wiki/w/Money_making_guide/Bird_house_trapping) | Bone Voyage + Hunter متدرج | تراكمي فوق أي نشاط آخر |
| Mining Amethyst | ~259k (80–100/h) [2](https://oldschool.runescape.wiki/w/Money_making_guide/Mining_amethyst) | Mining 92+ | انتباه أدنى، سقف منخفض |
| Anglerfish / Karambwan | ~146k–179k [3](https://oldschool.runescape.wiki/w/Money_making_guide/Catching_anglerfish) [1](https://oldschool.runescape.wiki/w/Money_making_guide/Catching_raw_karambwan) | Fishing 65+/82+ | قيمة XP وAFK أكثر منها رأس مال |
| Vorkath (Lance, ~30/h) | ~4.0–4.3M | Dragon Slayer II | أدنى تباين PvM — الأصلح نمذجيًا [4](https://timesaver.gg/blog/osrs-vorkath-vs-zulrah) |
| Zulrah (20→35+/h) | ~1.9M ← +3.3M [1](https://oldschool.runescape.wiki/w/Money_making_guide/Killing_Zulrah) | بدء Regicide | قيمة متوقعة ثابتة، تباين ساعة مرتفع |

**القراءة الهندسية:** التعدين والصيد الخالصان سقفهما 150k–260k/h. المضاعفة تأتي من **سلسلة القيمة الكاملة** (خام ← قضيب، بذرة ← عشبة)، ونموذجكم يجب أن يسعّرها كاملة:

```
Profit/h = N × net_sell(P_out) − Σ(Q_i × P_buy_i) − Fees/h − ExpectedLoss/h
∂Profit/∂P_out ≈ N × 0.98   # لستيل N=5,400: كل −10gp في القضيب = −52.9k/h
EV(patch) = E[Yield] × net_sell(P_herb) − P_seed − P_compost − حصة التنقل
```

---

## 2) المحور الثاني: الأتمتة والتحليل التلقائي

### 2.1 المعمارية المشروعة: Pipeline + Human-in-the-Loop
```
Wiki API v2 (/mapping يوميًا، /latest كل 60–300ث، /1h و/5m مجدولة)
  → Storage (SQLite/DuckDB + Parquet؛ Postgres عند التوسع)
  → Scoring Engine (أدناه) → CSV/Dashboard/تنبيهات → قرار بشري → تنفيذ يدوي
بالتوازي: Flipping Utilities ← RuneLite (يدوي) → ملف JSON محلي → دفتر FIFO + محاكاة ورقية
```

### 2.2 لا ترتّبوا بالهامش الخام — `/latest` آخر طبعة لا دفتر أوامر
أثناء إعداد الدراسة ظهرت في لقطة حية صفقة شبحية `Raw cod: low=31 مقابل high=1,699` — درس مجاني في ضرورة الفلاتر. السكريبت المرفق يطبّق:

```
فلترة: عمر الطبعتين ≤ 30د، volume_1h ≥ حد أدنى، high/low ≤ 1.35،
      انحراف عن متوسط 1h ≤ 25%، margin>0 وROI ≥ 2%
Q_4h = min( BuyLimit , floor(Capital×35% / low) , floor(0.10 × Volume_1h × 4) )
Profit_4h = margin × Q_4h  ←  الوحدة الحقيقية: Profit لكل Slot-Hour = Profit_4h / 4
Score = Profit/Slot-Hour × 1/(1+age/15) × log10(Volume_1h + 10)
```

### 2.3 مراجع مفتوحة المصدر نظيفة (قراءة/عرض فقط)
RuneLite وPlugin Hub وقالب Example Plugin ووثائق الميزات المرفوضة هي مرجعيتكم الرسمية لما تستطيع إضافة معتمدة فعله (Overlays، تنبيهات، اشتراك في أحداث مثل تغير عروض الـ GE) وما لا تستطيعه — حقن إدخال، و`Client.menuAction`، وReflection/JNI محظور في إضافات الـ Hub [3](https://github.com/runelite/runelite/wiki/Rejected-or-Rolled-Back-Features) [9](https://github.com/runelite/runelite/wiki/Developer-Guide) [14](https://github.com/runelite/plugin-hub). إضافة Flipping Utilities مفتوحة المصدر هي نموذج تتبع سجل صفقاتكم محليًا الأفضل للدراسة [1](https://github.com/Flipping-Utilities)، وWise Old Man مرجع معمارية API مفتوحة ناجح في هذا النظام البيئي [1](https://github.com/wise-old-man/wise-old-man).

### 2.4 خط أحمر يجب أن أقوله صراحة
بشأن "تحليل الشاشة والتحكم الذاتي النظيف": **لا يوجد تحكم ذاتي داخل اللعبة نظيف**. RuneLite نفسه عميل معتمد من Jagex وإضافات الـ Hub مراجعة للتوافق مع قواعد الطرف الثالث [15](https://github.com/runelite/runelite/wiki/FAQ) [6](https://github.com/runelite/runelite/wiki/Information-about-the-Plugin-Hub)، أما أي بوتات شاشة/لون أو Auto-clicker أو حقن أحداث أو إضافات خارجية تنفّذ أفعالًا أو "تأنيس إدخال" لخداع الكشف فهي Macroing محظور، والكشف سلوكي، والعقوبات تصل للحظر الدائم. لذلك لا أقدم روابط أو تصاميم أو تقنيات إخفاء لها. البديل المكافئ في 90% من القيمة وبمخاطرة صفر: الرادار والتنبيهات والمحاكاة الورقية وقياس أدائكم اليدوي (زمن دورة، عائد محقق/نموذجي) كما هو مصمم أعلى.

---

## 3) المحور الثالث: خارطة من الصفر إلى الـ Bond فالمضاعفة

سعر الـ Bond (ID ‏13190، 14 يومًا) متغير حي: لقطات أكتوبر 2026 بين ~10.86M و~11.41M بحسب الصفحة واللحظة [1](https://oldschool.runescape.wiki/w/Old_school_bond) [7](https://oldschool.runescape.wiki/w/Exchange:Old_school_bond). اسحبوه من الـ API ولا تثبّتوه في أي خطة:

```
BondDailyBurn = BondPrice / 14 ≈ 814k/يوم عند 11.4M
ساعات بلوغ البوند: 250k/h ← ~46س | 500k/h ← ~23س | 1.5M/h ← ~7.6س | 4M/h ← ~2.9س
```

1. **0 ← 100k:** منح مجانية + Pie shells/دبغ. بالتوازي ارفع Magic ← 33 ثم 55 وSmithing ← 30 استعدادًا لليوم الأول بعضوية.
2. **100k ← 1M:** نشاط نشط 200k–350k/h + أول قلب تعليمي بنود F2P سائلة. قواعد: 3 فتحات فقط، ≤25% من الرصيد في البند، Margin-check بوحدة واحدة قبل أي كمية، خروج عند انعكاس الهامش.
3. **1M ← Bond:** محرك مزدوج: جلسات نشطة بأعلى طريقة متاحة لمتطلباتكم + عروض صابرة ليلية تعمل أثناءها. الـ KPI الوحيد: نموّ البنك يوميًا لا GP/h النظري.
4. **أول 14 يومًا (لا تحرقوها):** افتحوا رقاع أعشاب وبيوت طيور وSteel Blast Furnace فورًا — جولتا أعشاب (~400k) + جولتا طيور (~100k) + 30 دقيقة ستيل (~750k) ≈ **1.25M/يوم**، أي فائض واضح فوق حرق الـ Bond اليومي، فيتحول القلب بـ 8 فتحات لمسرّع لا لوسيلة بقاء.
5. **مضاعفة:** وزع الفتحات 4 سريع/2 صابر/2 لشراء مدخلات وبيع مخرجات سلسلة إنتاجكم نفسها، ≤10–20% للبند عند كبر الرصيد، ≥20% نقدًا لاقتناص انهيارات ما بعد التحديثات، ومراجعة أسبوعية لربح Slot-Hour المحقق مقابل النموذج. وفوق 250M/قطعة تبدأ كفاءة سقف الضريبة [2](https://oldschool.runescape.wiki/w/Grand_Exchange) — مرحلة متقدمة فقط بعد سجل محاكاة موثق.

---

## تشغيل الأداة المرفقة

```bash
cd osrs_study
export OSRS_USER_AGENT="my-ge-radar/1.0 (contact: you@example.com)"
python3 ge_flip_radar.py --demo                                  # اختبار دون شبكة
python3 ge_flip_radar.py --capital 10000000 --slots 3            # F2P حي
python3 ge_flip_radar.py --capital 25000000 --slots 8 --members  # P2P حي
```

تذكيران أخيران: مخرجات أي رادار **مرشحون لفحص الهامش داخل اللعبة بوحدة واحدة** وليست أوامر شراء، وكل أرقام الدراسة لقطات 18 أكتوبر 2026 تتطلب إعادة حساب حية. إن أردت، الخطوة التالية الطبيعية أن أبني لكم في رسالة قادمة: (أ) طبقة تخزين مجدولة + رسوم `timeseries`، أو (ب) قالب إضافة RuneLite للعرض والتنبيه، أو (ج) حاسبة سلاسل إنتاج (Steel/Runite/Herb EV) حية فوق نفس هذا الهيكل.








