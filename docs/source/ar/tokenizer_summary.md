# ملخص عن المجزئات اللغوية

[[open-in-colab]]

في هذه الصفحة، سنتناول بالتفصيل عملية التجزئة.

<Youtube id="VFp38yj8h3A"/>

كما رأينا في [برنامج تعليمي حول المعالجة المسبقة](preprocessing)، فإن تجزئة النص يقسمه إلى كلمات أو
الرموز الفرعية (كلمات جزئية)، والتي يتم بعد ذلك تحويلها إلى معرفات من خلال قائمة بحث. يعد تحويل الكلمات أو الرموز الفرعية إلى معرفات مباشرًا، لذا في هذا الملخص، سنركز على تقسيم النص إلى كلمات أو رموز فرعية (أي تجزئة النص).
وبشكل أكثر تحديدًا، سنلقي نظرة على الأنواع الثلاثة الرئيسية من المُجزئات اللغوية المستخدمة في 🤗 المحولات: [ترميز الأزواج البايتية (BPE)](#byte-pair-encoding)، [WordPiece](#wordpiece)، و [SentencePiece](#sentencepiece)، ونعرض أمثلة
على نوع المُجزئة الذي يستخدمه كل نموذج.

لاحظ أنه في كل صفحة نموذج، يمكنك الاطلاع على وثائق المُجزئة المرتبط لمعرفة نوع المُجزئ
الذي استخدمه النموذج المُدرب مسبقًا. على سبيل المثال، إذا نظرنا إلى [`BertTokenizer`]، يمكننا أن نرى أن النموذج يستخدم [WordPiece](#wordpiece).

## مقدمة

إن تقسيم النص إلى أجزاء أصغر هو مهمة أصعب مما تبدو، وهناك طرق متعددة للقيام بذلك.
على سبيل المثال، دعنا نلقي نظرة على الجملة `"Don't you love 🤗 Transformers? We sure do."`

<Youtube id="nhJxYji1aho"/>

يمكن تقسيم  هذه الجملة ببساطة عن طريق  المسافات، مما  سينتج عنه ما يلي:```

```
["Don't", "you", "love", "🤗", "Transformers?", "We", "sure", "do."]
```

هذه خطوة أولى منطقية، ولكن إذا نظرنا إلى الرموز `"Transformers?"` و `"do."`، فإننا نلاحظ أن علامات الترقيم مُرفقة بالكلمات `"Transformer"` و `"do"`، وهو أمر ليس مثالي. يجب أن نأخذ علامات الترقيم في الاعتبار حتى لا يضطر النموذج إلى تعلم تمثيل مختلف للكلمة وكل رمز ترقيم مُحتمل قد يليها، الأمر الذي من شأنه أن يزيد بشكل هائل عدد التمثيلات التي يجب على النموذج تعلمها.
مع مراعاة علامات الترقيم، سيُصبح تقسيم  نصنا  على النحو التالي:

```
["Don", "'", "t", "you", "love", "🤗", "Transformers", "?", "We", "sure", "do", "."]
```

أفضل. ومع ذلك، من غير الملائم كيفية تقسيم الكلمة `"Don't"`. `"Don't"` تعني `"do not"`، لذا سيكون من الأفضل تحليلها على أنها كلمتين  مُدمجتين `["Do"، "n't"]`. هنا تبدأ الأمور في التعقيد، وهو جزء من سبب امتلاك كل نموذج لنوّعه  الخاص من مُجزّئ  النصوص (tokenizer). اعتمادًا على القواعد التي نطبقها لتقسيم النص، يسيتم إنشاء مخرجات مُجزّأة  مُختلفة لنفس النص. ولن يؤدي النموذج المُدرب مسبقًا إلى الأداء بشكل صحيح إلا إذا  قُدّم  له مُدخل تم تقسيمه بنفس  القواعد التي تم استخدامها لتقسيم بيانات التدريب الخاصة به.

يُعد كل من [spaCy](https://spacy.io/) و [Moses](http://www.statmt.org/moses/?n=Development.GetStarted) هما مجزّئي النصوص التي تعتمد على القواعد
الشائعة. عند تطبيقها على مثالنا، فإن *spaCy* و *Moses* ستخرج نّصًا مثل:

```
["Do", "n't", "you", "love", "🤗", "Transformers", "?", "We", "sure", "do", "."]
```

كما يمكنك أن ترى، يتم هنا استخدام التقسيم المكاني والترقيم، وكذلك تقسيم الكلمات القائم على القواعد. يعد التقسيم المكاني والترقيم والتحليل القائم على القواعد كلاهما مثالين على تقسيم الكلمات، والذي يُعرّف بشكل غير مُحدد على أنه تقسيم  الجُمل إلى كلمات. في حين أنها الطريقة الأكثر بديهية لتقسيم النصوص إلى أجزاء أصغر،
يمكن أنها تؤدى إلى مشكلات لمجموعات النصوص الضخمة. في هذه الحالة، عادةً ما يؤدي التقسيم المكاني والترقيم
إلى إنشاء مفردات كبيرة جدًا (مجموعة من جميع الكلمات والرموز الفريدة المستخدمة). على سبيل المثال، يستخدم [Transformer XL](model_doc/transfo-xl) التقسيم المكاني والترقيم، مما يؤدي إلى حجم مُفردات يبلغ 267735!

يفرض حجم المُفردات الكبير هذا على النموذج أن يكون لديه مصفوفة تضمين (embedding matrix) ضخمة كطبقة إدخال وإخراج، مما يؤدي إلى زيادة كل من التعقيد الزمني والذاكرة. بشكل عام، نادرًا ما يكون لدى نماذج المحولات حجم مفردات
أكبر من 50000، خاصة إذا تم تدريبها مسبقًا على لغة واحدة فقط.

لذا إذا كان التقسيم المكاني و الترقيم البسيط غير مرضٍ، فلماذا لا نقسّم الحروف ببساطة؟

<Youtube id="ssLq_EK2jLE"/>

في حين أن تقسيم الأحرف بسيط للغاية ومن شأنه أن يقلل بشكل كبير من التعقيد الزمني والذاكرة، إلا أنه يجعل من الصعب
على النموذج تعلم تمثيلات المدخلات ذات معنى. على سبيل المثال، يعد تعلم تمثيل مستقل عن السياق للحرف "t" أكثر صعوبة من تعلم تمثيل مستقل عن السياق لكلمة "اليوم". لذلك، غالبًا ما يكون تحليل الأحرف مصحوبًا بفقدان الأداء. لذا للحصول على أفضل ما في العالمين، تستخدم نماذج المحولات نظامًا  هجينًا  بين تقسيم على مستوى الكلمة وتقسيم علي مستوى الأحرف يسمى **تقسيم   الوحدات  الفرعية  للّغة**   (subword   tokenization).

## تقسيم الوحدات الفرعية للّغة (Subword Tokenization)

<Youtube id="zHvTiHr506c"/>

تعتمد خوارزميات تقسيم الوحدات الفرعية subword على المبدأ القائل بأن الكلمات الشائعة الاستخدام لا ينبغي تقسيمها إلى وحدات فرعية أصغر، ولكن يجب تفكيك الكلمات النادرة إلى رموز فرعية ذات معنى. على سبيل المثال، قد يتم اعتبار "annoyingly"
كلمة نادرة ويمكن تحليلها إلى "annoying" و "ly". كل من "annoying" و "ly" كـ subwords مستقلة ستظهر بشكل متكرر أكثر في حين أن معنى "annoyingly" يتم الاحتفاظ به من خلال المعنى المركب لـ "annoying" و "ly". هذا مفيد بشكل خاص في اللغات التلصيقية مثل التركية، حيث يمكنك تشكيل كلمات مُركبة طويلة (تقريبًا) بشكل تعسفي عن طريق ضم الرموز الفرعية معًا.

يسمح تقسيم subword للنموذج بأن يكون له حجم مفردات معقول مع القدرة على تعلم تمثيلات مستقلة عن السياق ذات معنى. بالإضافة إلى ذلك، يمكّن تقسيم subword النموذج من معالجة الكلمات التي لم يسبق له رؤيتها من قبل، عن طريق تحليلها إلى رموز فرعية معروفة. على سبيل المثال، يقوم المحلل [`~transformers.BertTokenizer`] بتحليل"I have a new GPU!" كما يلي:

```py
>>> from transformers import BertTokenizer

>>> tokenizer = BertTokenizer.from_pretrained("google-bert/bert-base-uncased")
>>> tokenizer.tokenize("I have a new GPU!")
["i", "have", "a", "new", "gp", "##u", "!"]
```

نظرًا  لأننا نستخدم  نموذجًا غير حساس لحالة الأحرف (uncased model)، فقد تم تحويل الجملة إلى أحرف صغيرة أولاً. يمكننا أن نرى أن الكلمات `["i"، "have"، "a"، "new"]` موجودة في مفردات  مُجزّئ النصوص، ولكن الكلمة "gpu" غير موجودة. وبالتالي، يقوم مُجزّئ النصوص   بتقسيم "gpu" إلى رموز فرعية معروفة: `["gp" و "##u"]`. يعني "##" أنه يجب ربط بقية الرمز بالرمز السابق، دون مسافة (للترميز أو عكس عملية  تقسيم  الرموز).

كمثال آخر، يقوم المحلل [`~transformers.XLNetTokenizer`] بتقسيم نّص مثالنا السابق كما يلي:

```py
>>> from transformers import XLNetTokenizer

>>> tokenizer = XLNetTokenizer.from_pretrained("xlnet/xlnet-base-cased")
>>> tokenizer.tokenize("Don't you love 🤗 Transformers? We sure do.")
["▁Don", "'", "t", "▁you", "▁love", "▁"، "🤗"، "▁"، "Transform"، "ers"، "؟"، "▁We"، "▁sure"، "▁do"، "."]
```
سنعود إلى معنى تلك `"▁"` عندما نلقي نظرة على [SentencePiece](#sentencepiece). كما يمكنك أن ترى،
تم تقسيم الكلمة النادرة "Transformers" إلى الرموز الفرعية الأكثر تكرارًا `"Transform"` و `"ers"`.

دعنا الآن نلقي نظرة على كيفية عمل خوارزميات تقسيم subword المختلفة. لاحظ أن جميع خوارزميات التقسيم هذه تعتمد على بعض أشكال التدريب الذي يتم عادةً على مجموعة البيانات التي سيتم تدريبها النموذج عليها.

<a id='byte-pair-encoding'></a>

### ترميز الأزواج البايتية (BPE)

تم تقديم رميز أزواج البايت (BPE) في ورقة بحثية بعنوان [الترجمة الآلية العصبية للكلمات النادرة باستخدام وحدات subword (Sennrich et al.، 2015)](https://huggingface.co/papers/1508.07909). يعتمد BPE على مُجزّئ أولي يقسم بيانات التدريب إلى
كلمات. يمكن أن يكون التحليل المسبق بسيطًا مثل التقسيم المكاني، على سبيل المثال [GPT-2](model_doc/gpt2)، [RoBERTa](model_doc/roberta). تشمل التقسيم الأكثر تقدمًا معتمد على التحليل القائم على القواعد، على سبيل المثال [XLM](model_doc/xlm)، [FlauBERT](model_doc/flaubert) الذي يستخدم Moses لمعظم اللغات، أو [GPT](model_doc/openai-gpt) الذي يستخدم spaCy و ftfy، لحساب تكرار كل كلمة في مجموعة بيانات التدريب.

بعد التحليل المسبق، يتم إنشاء مجموعة من الكلمات الفريدة وقد تم تحديد تكرار كل كلمة في تم تحديد بيانات التدريب. بعد ذلك، يقوم BPE بإنشاء مفردات أساسية تتكون من جميع الرموز التي تحدث في مجموعة الكلمات الفريدة ويتعلم قواعد الدمج لتشكيل رمز جديد من رمزين من المفردات الأساسية. إنه يفعل ذلك حتى تصل المفردات إلى حجم المفردات المطلوب. لاحظ أن حجم المفردات هو فرط معلمة لتحديد قبل تدريب مُجزّئ  النصوص.

كمثال، دعنا نفترض أنه بعد  التقسيم    الأولي، تم تحديد مجموعة الكلمات التالية بما في ذلك تكرارها:

```
("hug", 10), ("pug", 5), ("pun", 12), ("bun", 4), ("hugs", 5)
```

وبالتالي، فإن المفردات الأساسية هي `["b"، "g"، "h"، "n"، "p"، "s"، "u"]`. من خلال تقسيم جميع الكلمات إلى رموز من
المفردات الأساسية، نحصل على:

```
("h" "u" "g"، 10)، ("p" "u" "g"، 5)، ("p" "u" "n"، 12)، ("b" "u" "n"، 4)، ("h" "u" "g" "s"، 5)
```

بعد ذلك، يقوم BPE بعدد مرات حدوث كل زوج من الرموز المحتملة ويختار زوج الرموز الذي يحدث بشكل متكرر. في
في المثال أعلاه، يحدث "h" متبوعًا بـ "u" _10 + 5 = 15_ مرة (10 مرات في 10 مرات
حدوث "hug"، 5 مرات في 5 مرات حدوث "hugs"). ومع ذلك، فإن أكثر أزواج الرموز شيوعًا هو "u" متبوعًا
بواسطة "g"، والتي تحدث _10 + 5 + 5 = 20_ مرة في المجموع. وبالتالي، فإن أول قاعدة دمج يتعلمها المحلل هي تجميع جميع
رموز "u" التي تتبعها "g" معًا. بعد ذلك، يتم إضافة "ug" إلى المفردات. تصبح مجموعة الكلمات

```
("h" "ug"، 10)، ("p" "ug"، 5)، ("p" "u" "n"، 12)، ("b" "u" "n"، 4)، ("h" "ug" "s"، 5)
```

بعد ذلك، يحدد BPE ثاني أكثر أزواج الرموز شيوعًا. إنه "u" متبوعًا بـ "n"، والذي يحدث 16 مرة. "u"،
يتم دمج "n" في "un" ويضاف إلى المفردات. ثالث أكثر أزواج الرموز شيوعًا هو "h" متبوعًا
بواسطة "ug"، والتي تحدث 15 مرة. مرة أخرى يتم دمج الزوج ويتم إضافة "hug" إلى المفردات.

في هذه المرحلة، تكون المفردات هي `["b"، "g"، "h"، "n"، "p"، "s"، "u"، "ug"، "un"، "hug"]` ومجموعة الكلمات الفريدة لدينا
تمثيله كما يلي:

```
("hug", 10), ("p" "ug", 5), ("p" "un", 12), ("b" "un", 4), ("hug" "s", 5)
```

بافتراض أن تدريب ترميز الأزواج البايت سيتوقف عند هذه النقطة، فسيتم تطبيق قواعد الدمج التي تم تعلمها بعد ذلك على الكلمات الجديدة (طالما أن هذه الكلمات الجديدة لا تشمل رموزًا لم تكن في المفردات الأساسية). على سبيل المثال، سيتم تقسيم كلمة "bug" إلى `["b"، "ug"]` ولكن سيتم تقسيم "mug" على أنها `["<unk>"، "ug"]` نظرًا لأن الرمز "m" غير موجود في المفردات الأساسية. بشكل عام، لا يتم استبدال الأحرف الفردية مثل "m" بالرمز "<unk>" لأن بيانات التدريب تتضمن عادةً ظهورًا واحدًا على الأقل لكل حرف، ولكن من المحتمل أن يحدث ذلك لرموز خاصة جدًا مثل الرموز التعبيرية.

كما ذكرنا سابقًا، فإن حجم المفردات، أي حجم المفردات الأساسية + عدد عمليات الدمج، هو معامل يجب اختياره. على سبيل المثال، لدى [GPT](model_doc/openai-gpt) حجم مفردات يبلغ 40478 منذ أن كان لديهم 478 حرفًا أساسيًا واختاروا التوقف عن التدريب بعد 40,000 عملية دمج.

#### ترميز الأزواج البايتية على مستوى البايت

قد تكون المفردات الأساسية التي تتضمن جميع الأحرف الأساسية كبيرة جدًا إذا *على سبيل المثال* تم اعتبار جميع أحرف اليونيكود
كأحرف أساسية. لذا، ليكون لديك مفردات أساسية أفضل، يستخدم [GPT-2](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf) البايتات كمفردات أساسية، وهي حيلة ذكية لإجبار المفردات الأساسية على أن تكون بحجم 256 مع ضمان أن يتم تضمين كل حرف أساسي في المفردات. مع بعض القواعد الإضافية للتعامل مع علامات الترقيم، يمكن لمُجزّئ  النصوص GPT2 تجزئة أي نص دون الحاجة إلى رمز <unk>. لدى [GPT-2](model_doc/gpt) حجم مفردات يبلغ 50257، والذي يتوافق مع رموز 256 base byte، ورمز خاص لنهاية النص والرموز التي تم تعلمها باستخدام 50000 عملية دمج.

<a id='wordpiece'></a>

### WordPiece

تعتبر WordPiece  خوارزمية تجزئة الكلمات الفرعية subword المستخدمة لـ [BERT](model_doc/bert)، [DistilBERT](model_doc/distilbert)، و [Electra](model_doc/electra). تم توضيح الخوارزمية في [البحث الصوتي الياباني والكوري
(Schuster et al.، 2012)](https://static.googleusercontent.com/media/research.google.com/ja//pubs/archive/37842.pdf) وهو مشابه جدًا
BPE. أولاً، يقوم WordPiece بتكوين المفردات لتضمين كل حرف موجود في بيانات التدريب
وتعلم تدريجياً عددًا معينًا من قواعد الدمج. على عكس BPE، لا يختار WordPiece أكثر زوج الرموز المتكررة، ولكن تلك التي تزيد من احتمال بيانات التدريب بمجرد إضافتها إلى المفردات.

لذا، ماذا يعني هذا بالضبط؟ بالإشارة إلى المثال السابق، فإن زيادة احتمال بيانات التدريب تعادل إيجاد زوج الرموز، الذي يكون احتمال تقسيمه على احتمالات رمزه الأول تليها رمزه الثاني هو الأكبر بين جميع أزواج الرموز. *مثال* `"u"`، تليها `"g"` كانت قد اندمجت فقط إذا كان احتمال `"ug"` مقسومًا على `"u"`، `"g"` كان سيكون أكبر من أي زوج آخر من الرموز. بديهيًا، WordPiece مختلف قليلاً عن BPE في أنه يقيم ما يفقده عن طريق دمج رمزين للتأكد من أنه يستحق ذلك.

<a id='unigram'></a>

### Unigram

Unigram هو خوارزمية توكنيز subword التي تم تقديمها في [تنظيم subword: تحسين نماذج الترجمة الشبكة العصبية
نماذج مع مرشحين subword متعددة (Kudo، 2018)](https://huggingface.co/papers/1804.10959). على عكس BPE أو
WordPiece، يقوم Unigram بتكوين مفرداته الأساسية إلى عدد كبير من الرموز ويقللها تدريجياً للحصول على مفردات أصغر. يمكن أن تتوافق المفردات الأساسية على سبيل المثال مع جميع الكلمات المسبقة التوكنز والسلاسل الفرعية الأكثر شيوعًا. لا يتم استخدام Unigram مباشرة لأي من النماذج في المحولات، ولكنه يستخدم بالاقتران مع [SentencePiece](#sentencepiece).

في كل خطوة تدريب، يحدد خوارزمية Unigram خسارة (غالبًا ما يتم تعريفها على أنها اللوغاريتم) عبر بيانات التدريب بالنظر إلى المفردات الحالية ونموذج اللغة unigram. بعد ذلك، بالنسبة لكل رمز في المفردات، يحسب الخوارزمية مقدار زيادة الخسارة الإجمالية إذا تم إزالة الرمز من المفردات. ثم يقوم Unigram بإزالة p (مع p عادة ما تكون 10% أو 20%) في المائة من الرموز التي تكون زيادة الخسارة فيها هي الأدنى، *أي* تلك
الرموز التي تؤثر أقل على الخسارة الإجمالية عبر بيانات التدريب. تتكرر هذه العملية حتى تصل المفردات إلى الحجم المطلوب. يحتفظ خوارزمية Unigram دائمًا بالشخصيات الأساسية بحيث يمكن توكنز أي كلمة.

نظرًا لأن Unigram لا يعتمد على قواعد الدمج (على عكس BPE وWordPiece)، فإن للخوارزمية عدة طرق
توكنز نص جديد بعد التدريب. على سبيل المثال، إذا كان محول Unigram المدرب يعرض المفردات:

```
["b"، "g"، "h"، "n"، "p"، "s"، "u"، "ug"، "un"، "hug"]،
```

يمكن توكنز `"hugs"` على أنه `["hug"، "s"]`، أو `["h"، "ug"، "s"]` أو `["h"، "u"، "g"، "s"]`. إذن ماذا
لاختيار؟ يحفظ Unigram احتمال كل رمز في فيلق التدريب بالإضافة إلى حفظ المفردات بحيث
يمكن حساب احتمال كل توكنز ممكن بعد التدريب. ببساطة، يختار الخوارزمية الأكثر
توكنز المحتملة في الممارسة، ولكنه يوفر أيضًا إمكانية أخذ عينات من توكنز ممكن وفقًا لاحتمالاتها.

تتم تعريف هذه الاحتمالات بواسطة الخسارة التي يتم تدريب المحول عليها. بافتراض أن بيانات التدريب تتكون
من الكلمات \\(x_{1}، \dots، x_{N}\\) وأن مجموعة جميع التوكنزات الممكنة لكلمة \\(x_{i}\\) هي
يتم تعريفها على أنها \\(S(x_{i})\\)، ثم يتم تعريف الخسارة الإجمالية على النحو التالي

$$\mathcal{L} = -\sum_{i=1}^{N} \log \left ( \sum_{x \in S(x_{i})} p(x) \right )$$

<a id='sentencepiece'></a>

### SentencePiece

تحتوي جميع خوارزميات توكنز الموصوفة حتى الآن على نفس المشكلة: من المفترض أن النص المدخل يستخدم المسافات لفصل الكلمات. ومع ذلك، لا تستخدم جميع اللغات المسافات لفصل الكلمات. أحد الحلول الممكنة هو استخداممعالج مسبق للغة محدد، *مثال* [XLM](model_doc/xlm) يلذي يستخدم معالجات مسبقة محددة للصينية واليابانية والتايلاندية.
لحل هذه المشكلة بشكل أعم، [SentencePiece: A simple and language independent subword tokenizer and
detokenizer for Neural Text Processing (Kudo et al.، 2018)](https://huggingface.co/papers/1808.06226) يتعامل مع المدخلات
كتدفق بيانات خام، وبالتالي يشمل المسافة في مجموعة الأحرف التي سيتم استخدامها. ثم يستخدم خوارزمية BPE أو unigram
لبناء المفردات المناسبة.

يستخدم [`XLNetTokenizer`] SentencePiece على سبيل المثال، وهو أيضًا سبب تضمين تم تضمين حرف `"▁"` في المفردات. عملية فك التشفير باستخدام SentencePiece سهلة للغاية نظرًا لأنه يمكن دائمًا دمج الرموز معًا واستبدال `"▁"` بمسافة.

تستخدم جميع نماذج المحولات في المكتبة التي تستخدم SentencePiece بالاقتران مع unigram. أمثلة على النماذج
باستخدام SentencePiece هي [ALBERT](model_doc/albert)، [XLNet](model_doc/xlnet)، [Marian](model_doc/marian)، و [T5](model_doc/t5).
