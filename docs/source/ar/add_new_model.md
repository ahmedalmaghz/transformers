# كيف تضيف نموذجًا إلى 🤗 Transformers؟

مكتبة 🤗 Transformers قادرة في كثير من الأحيان على تقديم نماذج جديدة بفضل المساهمين من المجتمع. ولكن يمكن أن يكون هذا مشروعًا صعبًا ويتطلب معرفة متعمقة بمكتبة 🤗 Transformers والنموذج الذي سيتم تنفيذه. في Hugging Face، نحاول تمكين المزيد من المجتمع للمساهمة بنشاط في إضافة النماذج، وقد قمنا بتجميع هذا الدليل لاصطحابك خلال عملية إضافة نموذج PyTorch (تأكد من تثبيت [PyTorch](https://pytorch.org/get-started/locally/)).

على طول الطريق، سوف:

- الحصول على رؤى حول أفضل الممارسات مفتوحة المصدر
- فهم مبادئ التصميم وراء واحدة من أكثر المكتبات شعبية في التعلم العميق
- تعلم كيفية اختبار النماذج الكبيرة بكفاءة
- تعلم كيفية دمج أدوات Python مثل `black`، و`ruff`، و`make fix-copies` لضمان نظافة الكود وقابليته للقراءة

سيكون أحد أعضاء فريق Hugging Face متاحًا لمساعدتك على طول الطريق، لذلك لن تكون بمفردك أبدًا. 🤗 ❤️

لتبدأ، قم بفتح [إضافة نموذج جديد](https://github.com/huggingface/transformers/issues/new?assignees=&labels=New+model&template=new-model-addition.yml) لإصدار النموذج الذي تريد رؤيته في 🤗 Transformers. إذا لم تكن مهتمًا بشكل خاص بالمساهمة بنموذج محدد، فيمكنك تصفية [علامة النموذج الجديد](https://github.com/huggingface/transformers/labels/New%20model) لمعرفة ما إذا كانت هناك أي طلبات نموذج غير مستلمة والعمل عليها.

بمجرد فتح طلب نموذج جديد، تتمثل الخطوة الأولى في التعرف على 🤗 Transformers إذا لم تكن كذلك بالفعل!

## نظرة عامة على 🤗 Transformers

أولاً، يجب أن تحصل على نظرة عامة حول 🤗 Transformers. 🤗 Transformers هي مكتبة ذات آراء قوية، لذلك هناك
احتمال ألا تتفق مع بعض فلسفات المكتبة أو خيارات التصميم الخاصة بها. ومع ذلك، من تجربتنا، وجدنا أن خيارات التصميم والفلسفات الأساسية للمكتبة أمر بالغ الأهمية لزيادة حجم 🤗
Transformers مع الحفاظ على تكاليف الصيانة عند مستوى معقول.

نقطة بداية جيدة لفهم المكتبة بشكل أفضل هي قراءة [وثائق فلسفتنا](philosophy). وكنتيجة لطريقتنا في العمل، هناك بعض الخيارات التي نحاول تطبيقها على جميع النماذج:

- التكوين مفضل بشكل عام على التجريد
- لا يعد تكرار الكود أمرًا سيئًا دائمًا إذا كان يحسن بشكل كبير قابلية قراءة النموذج أو إمكانية الوصول إليه
- ملفات النماذج ذاتية الاحتواء قدر الإمكان بحيث عند قراءة كود نموذج محدد، فأنت بحاجة فقط
  للنظر في ملف `modeling_....py` المقابل.

في رأينا، فإن كود المكتبة ليس مجرد وسيلة لتوفير منتج، على سبيل المثال، القدرة على استخدام BERT للاستدلال، ولكن أيضًا كمنتج نريد تحسينه. وبالتالي، عند إضافة نموذج، فإن المستخدم ليس فقط
الشخص الذي سيستخدم نموذجك، ولكن أيضًا كل من سيقرأ كودك ويحاول فهمه وربما تعديله.

مع وضع ذلك في الاعتبار، دعنا نتعمق قليلاً في التصميم العام للمكتبة.

### نظرة عامة على النماذج

لإضافة نموذج بنجاح، من المهم فهم التفاعل بين نموذجك وتكوينه،
[`PreTrainedModel`]، و [`PretrainedConfig`]. ولأغراض التوضيح، سنطلق على النموذج الذي سيتم إضافته إلى 🤗 Transformers اسم `BrandNewBert`.

دعنا نلقي نظرة:

<img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers_overview.png"/>

كما ترى، فإننا نستخدم الوراثة في 🤗 Transformers، ولكننا نبقي مستوى التجريد عند الحد الأدنى المطلق. لا يوجد أبدًا أكثر من مستويين من التجريد لأي نموذج في المكتبة. `BrandNewBertModel`
يرث من `BrandNewBertPreTrainedModel` الذي بدوره يرث من [`PreTrainedModel`] وهذا كل شيء. كقاعدة عامة، نريد التأكد من أن النموذج الجديد يعتمد فقط على
[`PreTrainedModel`]. الوظائف المهمة التي يتم توفيرها تلقائيًا لأي نموذج جديد هي [`~PreTrainedModel.from_pretrained`] و
[`~PreTrainedModel.save_pretrained`]، والتي تستخدم للتسلسل واللإلغاء التسلسل. جميع
الوظائف المهمة الأخرى، مثل `BrandNewBertModel.forward` يجب أن تكون محددة بالكامل في النص الجديد
`modeling_brand_new_bert.py`. بعد ذلك، نريد التأكد من أن النموذج بطبقة رأس محددة، مثل
`BrandNewBertForMaskedLM` لا يرث من `BrandNewBertModel`، ولكنه يستخدم `BrandNewBertModel`
كمكون يمكن استدعاؤه في تمريره للأمام للحفاظ على مستوى التجريد منخفضًا. يتطلب كل نموذج جديد
فئة تكوين، تسمى `BrandNewBertConfig`. يتم دائمًا تخزين هذا التكوين كسمة في
[`PreTrainedModel`]، وبالتالي يمكن الوصول إليه عبر السمة `config` لجميع الفئات
التي ترث من `BrandNewBertPreTrainedModel`:

```python
model = BrandNewBertModel.from_pretrained("brandy/brand_new_bert")
model.config  # model has access to its config
```

مشابه للنموذج، يرث التكوين وظائف التسلسل وإلغاء التسلسل الأساسية من
[`PretrainedConfig`]. لاحظ أن التكوين والنموذج يتم تسلسلهما دائمًا إلى تنسيقين مختلفين - النموذج إلى ملف *pytorch_model.bin* والتكوين إلى ملف *config.json*. يؤدي استدعاء
[`~PreTrainedModel.save_pretrained`] للنموذج إلى استدعاء
[`~PretrainedConfig.save_pretrained`] للتكوين، بحيث يتم حفظ كل من النموذج والتكوين.


### أسلوب الترميز

عند كتابة كود نموذجك الجديد، ضع في اعتبارك أن Transformers هي مكتبة ذات آراء قوية ولدينا بعض الغرابة الخاصة بنا فيما يتعلق بكيفية كتابة الكود :-)

1. يجب كتابة تمرير النموذج للأمام بالكامل في ملف النمذجة مع الاستقلال التام عن النماذج الأخرى في المكتبة. إذا كنت تريد إعادة استخدام كتلة من نموذج آخر، فقم بنسخ الكود ولصقه مع
تعليق `# Copied from` في الأعلى (انظر [هنا](https://github.com/huggingface/transformers/blob/v4.17.0/src/transformers/models/roberta/modeling_roberta.py#L160)
لمثال جيد و [هنا](pr_checks#check-copies) لمزيد من الوثائق حول "Copied from"). 
2. يجب أن يكون الكود مفهومًا تمامًا، حتى لغير المتحدثين الأصليين باللغة الإنجليزية. وهذا يعني أنه يجب عليك اختيار
أسماء متغيرات وصفية وتجنب الاختصارات. على سبيل المثال، يفضل `activation` على `act`.
يتم التثبيط بشدة من استخدام أسماء متغيرات مكونة من حرف واحد ما لم يكن مؤشرًا في حلقة for.
3. بشكل أكثر عمومية، نفضل الكود الطويل الصريح على الكود السحري القصير.
4. تجنب الوراثة الفرعية لـ `nn.Sequential` في PyTorch ولكن ورث من `nn.Module` واكتب تمرير النموذج للأمام، بحيث يمكن لأي شخص
يستخدم كودك أن يقوم بالتصحيح السريع عن طريق إضافة عبارات الطباعة أو نقاط التوقف.
5. يجب أن يكون توقيع الدالة الخاص بك معنونًا بالنوع. بالنسبة للباقي، أسماء المتغيرات الجيدة أكثر قابلية للقراءة والفهم من علامات النوع.

### نظرة عامة على المحللات

لسنا مستعدين لذلك بعد :-( سيتم إضافة هذا القسم قريبًا!

## وصفة خطوة بخطوة لإضافة نموذج إلى 🤗 Transformers

لدى الجميع تفضيلات مختلفة لكيفية نقل نموذج، لذا قد يكون من المفيد لك أن تلقي نظرة على ملخصات حول كيفية قيام المساهمين الآخرين بنقل النماذج إلى Hugging Face. فيما يلي قائمة منشورات المجتمع حول كيفية نقل نموذج:
1. [نقل نموذج GPT2](https://medium.com/huggingface/from-tensorflow-to-pytorch-265f40ef2a28) بقلم [Thomas](https://huggingface.co/thomwolf)
2. [نقل نموذج WMT19 MT](https://huggingface.co/blog/porting-fsmt) بقلم [Stas](https://huggingface.co/stas)

من تجربتنا، يمكننا أن نخبرك بأن أهم الأشياء التي يجب مراعاتها عند إضافة نموذج هي:

- لا تعيد اختراع العجلة! معظم أجزاء الكود الذي ستضيفه لنموذج 🤗 Transformers الجديد موجودة بالفعل
  في مكان ما في 🤗 Transformers. خذ بعض الوقت للعثور على نماذج ومحللات مماثلة موجودة بالفعل يمكنك النسخ منها. [grep](https://www.gnu.org/software/grep/) و [rg](https://github.com/BurntSushi/ripgrep) هما صديقانك. لاحظ أنه قد يحدث جيدًا أن تكون محول نموذجك النصي مستندًا إلى تنفيذ نموذج واحد،
  وكود نمذجة نموذجك مستندًا إلى نموذج آخر. على سبيل المثال، يعتمد كود نمذجة FSMT على BART، بينما يعتمد كود محول رموز FSMT النصي على XLM.
- إنه تحدٍ هندسي أكثر من كونه تحديًا علميًا. يجب أن تقضي المزيد من الوقت في إنشاء
  بيئة تصحيح فعالة بدلاً من محاولة فهم جميع الجوانب النظرية للنموذج في الورقة.
- اطلب المساعدة عندما تواجه مشكلة! النماذج هي المكون الأساسي لـ 🤗 Transformers، لذلك نحن في Hugging Face سعداء جدًا لمساعدتك في كل خطوة لإضافة نموذجك. لا تتردد في السؤال إذا لاحظت أنك لا تحقق أي تقدم.

فيما يلي، نحاول تقديم وصفة عامة وجدناها الأكثر فائدة عند نقل نموذج إلى 🤗 Transformers.

القائمة التالية هي ملخص لكل ما يجب القيام به لإضافة نموذج ويمكنك استخدامه كقائمة مرجعية:

☐ (اختياري) فهم الجوانب النظرية للنموذج<br>
☐ إعداد بيئة تطوير 🤗 Transformers<br>
☐ إعداد بيئة تصحيح المستودع الأصلي<br>
☐ إنشاء نص برمجي يقوم بتشغيل تمرير `forward()` باستخدام المستودع الأصلي ونقطة المراقبة<br>
☐ إضافة هيكل النموذج بنجاح إلى 🤗 Transformers<br>
☐ تحويل نقطة المراقبة الأصلية بنجاح إلى نقطة مراقبة 🤗 Transformers<br>
☐ تشغيل تمرير `forward()` في 🤗 Transformers الذي يعطي نفس الإخراج لنقطة المراقبة الأصلية<br>
☐ الانتهاء من اختبارات النموذج في 🤗 Transformers<br>
☐ إضافة المحلل بنجاح في 🤗 Transformers<br>
☐ تشغيل اختبارات التكامل من النهاية إلى النهاية<br>
☐ الانتهاء من الوثائق<br>
☐ تحميل أوزان النموذج إلى Hub<br>
☐ إرسال طلب السحب<br>
☐ (اختياري) إضافة دفتر ملاحظات توضيحي

في البداية، نوصي عادةً بالبدء بالحصول على فهم نظري جيد لـ `BrandNewBert`. ومع ذلك،
إذا كنت تفضل فهم الجوانب النظرية للنموذج أثناء العمل، فيمكنك الانتقال مباشرةً إلى قاعدة كود `BrandNewBert`. قد يناسبك هذا الخيار بشكل أفضل إذا كانت مهاراتك الهندسية أفضل من مهاراتك النظرية، أو إذا كنت تواجه صعوبة في فهم ورقة `BrandNewBert`، أو إذا كنت تستمتع بالبرمجة أكثر من قراءة الأوراق العلمية.

### 1. (اختياري) الجوانب النظرية لـ BrandNewBert

يجب أن تأخذ بعض الوقت لقراءة ورقة *BrandNewBert*، إذا كان هناك عمل وصفي. قد تكون هناك أجزاء كبيرة من الورقة يصعب فهمها. إذا كان الأمر كذلك، فلا بأس - لا تقلق! الهدف ليس هو الحصول على فهم نظري عميق للورقة، ولكن لاستخراج المعلومات اللازمة ل
إعادة تنفيذ النموذج في 🤗 Transformers. هذا لا يعني أنه لا يتعين عليك قضاء الكثير من الوقت في الجوانب النظرية، ولكن التركيز على الجوانب العملية، وهي:
-  ما نوع نموذج *brand_new_bert*؟ نموذج تشفير فقط مثل BERT؟ نموذج فك تشفير فقط مثل GPT2؟ نموذج ترميز وفك تشفير مثل BART؟ راجع [model_summary](model_summary) إذا لم تكن على دراية بالاختلافات بين تلك النماذج.
-  ما هي تطبيقات *brand_new_bert*؟ تصنيف النصوص؟ توليد النصوص؟ مهام Seq2Seq، على سبيل المثال،
  تلخيص؟
-  ما هي الميزة الجديدة للنموذج التي تجعله مختلفًا عن BERT/GPT-2/BART؟
-  أي من نماذج [🤗 Transformers](https://huggingface.co/transformers/#contents) الموجودة تشبه *brand_new_bert*؟
-  ما نوع المحلل المستخدم؟ محلل sentencepiece؟ محلل word piece؟ هل هو نفس المحلل المستخدم لـ BERT أو BART؟

بعد أن تشعر أنك حصلت على نظرة عامة جيدة عن بنية النموذج، قد ترغب في الكتابة إلى
فريق Hugging Face بأي أسئلة قد تكون لديك. قد يتضمن ذلك أسئلة حول بنية النموذج،
طبقة الاهتمام، إلخ. سنكون سعداء لمساعدتك.

### 2. قم بإعداد بيئتك

1. قم بتشعب المستودع [repository](https://github.com/huggingface/transformers) بالنقر فوق زر "Fork" في صفحة المستودع. يقوم هذا بإنشاء نسخة من الكود في حساب GitHub الخاص بك.

2. قم باستنساخ تشعب `transformers` الخاص بك إلى القرص المحلي الخاص بك، وأضف المستودع الأساسي كجهاز بعيد:

   ```bash
   git clone https://github.com/[your Github handle]/transformers.git
   cd transformers
   git remote add upstream https://github.com/huggingface/transformers.git
   ```

3. قم بإعداد بيئة تطوير، على سبيل المثال عن طريق تشغيل الأمر التالي:

   ```bash
   python -m venv .env
   source .env/bin/activate
   pip install -e ".[dev]"
   ```

   اعتمادًا على نظام التشغيل الخاص بك، ونظرًا لأن عدد التبعيات الاختيارية لـ Transformers يتزايد، فقد تحصل على فشل بهذا الأمر. إذا كان الأمر كذلك، فتأكد من تثبيت إطار عمل Deep Learning الذي تعمل معه
(PyTorch، وTensorFlow، و/أو Flax)، ثم قم بما يلي:

   ```bash
   pip install -e ".[quality]"
   ```

   يجب أن يكون هذا كافيًا لمعظم حالات الاستخدام. يمكنك بعد ذلك العودة إلى الدليل الرئيسي

   ```bash
   cd ..
   ```

4. نوصي بإضافة إصدار PyTorch من *brand_new_bert* إلى Transformers. لتثبيت PyTorch، يرجى اتباع التعليمات على https://pytorch.org/get-started/locally/.

   **ملاحظة:** لا تحتاج إلى تثبيت CUDA. يكفي أن يعمل النموذج الجديد على وحدة المعالجة المركزية.

5. لنقل *brand_new_bert*، ستحتاج أيضًا إلى الوصول إلى مستودعه الأصلي:

   ```bash
   git clone https://github.com/org_that_created_brand_new_bert_org/brand_new_bert.git
   cd brand_new_bert
   pip install -e .
   ```

الآن قمت بإعداد بيئة تطوير لنقل *brand_new_bert* إلى 🤗 Transformers.

### 3.-4. تشغيل نقطة مراقبة مُدربة مسبقًا باستخدام المستودع الأصلي

في البداية، ستعمل على المستودع الأصلي لـ *brand_new_bert*. غالبًا ما يكون التنفيذ الأصلي "بحثًا". وهذا يعني أن الوثائق قد تكون مفقودة وقد يكون الكود صعب الفهم. ولكن يجب أن يكون هذا هو دافعك لإعادة تنفيذ *brand_new_bert*. في Hugging Face، يتمثل أحد أهدافنا الرئيسية في *مساعدة الناس
على الوقوف على أكتاف العمالقة* والتي تترجم هنا جيدًا إلى أخذ نموذج عامل وإعادة كتابته لجعله **سهل الوصول إليه وسهل الاستخدام وجميل** قدر الإمكان. هذه هي الدافع الرئيسي لإعادة تنفيذ النماذج في 🤗 Transformers - محاولة جعل تقنية NLP الجديدة المعقدة في متناول **الجميع**.

يجب أن تبدأ بالغوص في المستود
لم يتم العثور على ملفات الترجمة الخاصة بالنموذج المطلوب، يرجى التأكد من وجود الملفات وتوفيرها لي حتى أتمكن من ترجمتها.
لن يتم ترجمة النصوص البرمجية وروابط و رموز HTML و CSS الخاصة بالنص الأصلي. سيتم ترجمة النص الموجود في الفقرات والعناوين فقط.

**6. كتابة برنامج تحويل**

بعد ذلك، يجب عليك كتابة برنامج تحويل يسمح لك بتحويل نقطة التفتيش التي استخدمتها لتصحيح أخطاء *brand_new_bert* في المستودع الأصلي إلى نقطة تفتيش متوافقة مع تنفيذك لـ *brand_new_bert* في مكتبة 🤗 Transformers. لا يُنصح بكتابة برنامج التحويل من الصفر، ولكن بدلاً من ذلك، قم بالبحث في برامج التحويل الموجودة مسبقًا في 🤗 Transformers عن برنامج تم استخدامه لتحويل نموذج مشابه تم كتابته في نفس الإطار مثل *brand_new_bert*. عادةً، يكون من الكافي نسخ برنامج تحويل موجود مسبقًا وتعديله قليلاً ليناسب حالتك. لا تتردد في طلب المساعدة من فريق Hugging Face للإشارة إلى برنامج تحويل مشابه موجود مسبقًا لنموذجك.

- إذا كنت تقوم بنقل نموذج من TensorFlow إلى PyTorch، فقد تكون نقطة بداية جيدة هي برنامج تحويل BERT الموجود [هنا](https://github.com/huggingface/transformers/blob/7acfa95afb8194f8f9c1f4d2c6028224dbed35a2/src/transformers/models/bert/modeling_bert.py#L91).
- إذا كنت تقوم بنقل نموذج من PyTorch إلى PyTorch، فقد تكون نقطة بداية جيدة هي برنامج تحويل BART الموجود [هنا](https://github.com/huggingface/transformers/blob/main/src/transformers/models/bart/convert_bart_original_pytorch_checkpoint_to_pytorch.py).

فيما يلي، سنشرح بسرعة كيف تقوم نماذج PyTorch بتخزين أوزان الطبقات وتعريف أسماء الطبقات. في PyTorch، يتم تعريف اسم الطبقة بواسطة اسم وسيلة الفئة التي تعطيها للطبقة. دعنا نحدد نموذجًا وهميًا في PyTorch، يسمى `SimpleModel` كما يلي:

```python
from torch import nn


class SimpleModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.dense = nn.Linear(10, 10)
        self.intermediate = nn.Linear(10, 10)
        self.layer_norm = nn.LayerNorm(10)
```

الآن يمكننا إنشاء مثيل من هذا التعريف النموذجي والذي سيملأ جميع الأوزان: `dense`، `intermediate`، `layer_norm` بأوزان عشوائية. يمكننا طباعة النموذج لمعرفة بنائه:

```python
model = SimpleModel()

print(model)
```

سيقوم هذا بطباعة ما يلي:

```
SimpleModel(
  (dense): Linear(in_features=10, out_features=10, bias=True)
  (intermediate): Linear(in_features=10, out_features=10, bias=True)
  (layer_norm): LayerNorm((10,), eps=1e-05, elementwise_affine=True)
)
```

يمكننا أن نرى أن أسماء الطبقات يتم تعريفها بواسطة اسم وسيلة الفئة في PyTorch. يمكنك طباعة قيم الوزن لطبقة محددة:

```python
print(model.dense.weight.data)
```

لترى أن الأوزان تم تهيئتها بشكل عشوائي:

```
tensor([[-0.0818,  0.2207, -0.0749, -0.0030,  0.0045, -0.1569, -0.1598,  0.0212,
         -0.2077,  0.2157],
        [ 0.1044,  0.0201,  0.0990,  0.2482,  0.3116,  0.2509,  0.2866, -0.2190,
          0.2166, -0.0212],
        [-0.2000,  0.1107, -0.1999, -0.3119,  0.1559,  0.0993,  0.1776, -0.1950,
         -0.1023, -0.0447],
        [-0.0888, -0.1092,  0.2281,  0.0336,  0.1817, -0.0115,  0.2096,  0.1415,
         -0.1876, -0.2467],
        [ 0.2208, -0.2352, -0.1426, -0.2636, -0.2889, -0.2061, -0.2849, -0.0465,
          0.2577,  0.0402],
        [ 0.1502,  0.2465,  0.2566,  0.0693,  0.2352, -0.0530,  0.1859, -0.0604,
          0.2132,  0.1680],
        [ 0.1733, -0.2407, -0.1721,  0.1484,  0.0358, -0.0633, -0.0721, -0.0090,
          0.2707, -0.2509],
        [-0.1173,  0.1561,  0.2945,  0.0595, -0.1996,  0.2988, -0.0802,  0.0407,
          0.1829, -0.1568],
        [-0.1164, -0.2228, -0.0403,  0.0428,  0.1339,  0.0047,  0.1967,  0.2923,
          0.0333, -0.0536],
        [-0.1492, -0.1616,  0.1057,  0.1950, -0.2807, -0.2710, -0.1586,  0.0739,
          0.2220,  0.2358]]).
```

في برنامج التحويل، يجب عليك ملء الأوزان التي تم تهيئتها بشكل عشوائي بأوزان الطبقة المطابقة في نقطة التفتيش. على سبيل المثال:

```python
# retrieve matching layer weights, e.g. by
# recursive algorithm
layer_name = "dense"
pretrained_weight = array_of_dense_layer

model_pointer = getattr(model, "dense")

model_pointer.weight.data = torch.from_numpy(pretrained_weight)
```

بينما تفعل ذلك، يجب عليك التحقق من أن كل وزن تم تهيئته بشكل عشوائي في نموذج PyTorch الخاص بك ووزن نقطة التفتيش المطابق له يتطابق تمامًا في كل من **الشكل والاسم**. للقيام بذلك، من **الضروري** إضافة عبارات التأكيد للشكل وطباعة أسماء أوزان نقطة التفتيش. على سبيل المثال، يجب عليك إضافة عبارات مثل:

```python
assert (
    model_pointer.weight.shape == pretrained_weight.shape
), f"Pointer shape of random weight {model_pointer.shape} and array shape of checkpoint weight {pretrained_weight.shape} mismatched"
```

بالإضافة إلى ذلك، يجب عليك أيضًا طباعة أسماء كلا الوزنين للتأكد من تطابقهما، على سبيل المثال:

```python
logger.info(f"Initialize PyTorch weight {layer_name} from {pretrained_weight.name}")
```

إذا لم يتطابق الشكل أو الاسم، فمن المحتمل أنك عينت وزن نقطة تفتيش خاطئة لطبقة تم تهيئتها بشكل عشوائي في تنفيذ 🤗 Transformers.

يرجع عدم تطابق الشكل على الأرجح إلى إعداد خاطئ لمعلمات التهيئة في `BrandNewBertConfig()` والتي لا تتطابق تمامًا مع تلك المستخدمة لنقطة التفتيش التي تريد تحويلها. ومع ذلك، فقد يكون أيضًا أن تنفيذ PyTorch للطبقة يتطلب نقل الوزن مسبقًا.

أخيرًا، يجب عليك أيضًا التحقق من أن **جميع** الأوزان المطلوبة قد تم تهيئتها وطباعة جميع أوزان نقطة التفتيش التي لم يتم استخدامها للتهيئة للتأكد من تحويل النموذج بشكل صحيح. من الطبيعي تمامًا أن تفشل محاولات التحويل بعبارة شكل خاطئة أو تعيين اسم خاطئ. يرجع هذا على الأرجح إلى أنك استخدمت معلمات غير صحيحة في `BrandNewBertConfig()`، أو لديك بنية خاطئة في تنفيذ 🤗 Transformers، أو لديك خطأ في وظائف `init()` لأحد مكونات تنفيذ 🤗 Transformers، أو تحتاج إلى نقل أحد أوزان نقطة التفتيش.

يجب تكرار هذه الخطوة مع الخطوة السابقة حتى يتم تحميل جميع أوزان نقطة التفتيش بشكل صحيح في نموذج Transformers. بعد تحميل نقطة التفتيش بشكل صحيح في تنفيذ 🤗 Transformers، يمكنك بعد ذلك حفظ النموذج في مجلد من اختيارك `/path/to/converted/checkpoint/folder` والذي يجب أن يحتوي بعد ذلك على كل من ملف `pytorch_model.bin` وملف `config.json`:

```python
model.save_pretrained("/path/to/converted/checkpoint/folder")
```

**7. تنفيذ تمرير للأمام**

بعد أن تمكنت من تحميل الأوزان المُدربة مسبقًا في تنفيذ 🤗 Transformers، يجب عليك الآن التأكد من تنفيذ تمرير للأمام بشكل صحيح. في [تعرف على المستودع الأصلي](#3-4-run-a-pretrained-checkpoint-using-the-original-repository)، قمت بالفعل بإنشاء برنامج يقوم بتشغيل تمرير للأمام للنموذج باستخدام المستودع الأصلي. الآن يجب عليك كتابة برنامج مماثل باستخدام تنفيذ 🤗 Transformers بدلاً من الأصلي. يجب أن يبدو كما يلي:

```python
model = BrandNewBertModel.from_pretrained("/path/to/converted/checkpoint/folder")
input_ids = [0, 4, 4, 3, 2, 4, 1, 7, 19]
output = model(input_ids).last_hidden_states
```

من المحتمل جدًا ألا يعطي تنفيذ 🤗 Transformers والتنفيذ الأصلي نفس الإخراج في المرة الأولى أو أن تمرير للأمام يرمي خطأ. لا تشعر بخيبة أمل - هذا متوقع! أولاً، يجب عليك التأكد من أن تمرير للأمام لا يرمي أي أخطاء. غالبًا ما يحدث أن الأبعاد الخاطئة تؤدي إلى خطأ "عدم تطابق الأبعاد" أو أنه يتم استخدام نوع بيانات خاطئ، على سبيل المثال `torch.long` بدلاً من `torch.float32`. لا تتردد في طلب المساعدة من فريق Hugging Face إذا لم تتمكن من حل أخطاء معينة.

الجزء النهائي للتأكد من أن تنفيذ 🤗 Transformers يعمل بشكل صحيح هو التأكد من أن الإخراج مكافئ بدقة `1e-3`. أولاً، يجب التأكد من أن أشكال الإخراج متطابقة، أي يجب أن يعطي `outputs.shape` نفس القيمة لكل من برنامج تنفيذ 🤗 Transformers والتنفيذ الأصلي. بعد ذلك، يجب التأكد من أن قيم الإخراج متطابقة أيضًا. هذه واحدة من أصعب الأجزاء في إضافة نموذج جديد. الأخطاء الشائعة التي تجعل الإخراج غير متطابق هي:

- لم يتم إضافة بعض الطبقات، أي لم يتم إضافة طبقة "تنشيط" أو تم نسيان الاتصال المتبقي
- لم يتم ربط مصفوفة تضمين الكلمات
- يتم استخدام تضمينات موضعية خاطئة لأن التنفيذ الأصلي يستخدم إزاحة
- يتم تطبيق الإسقاط أثناء تمرير للأمام. لإصلاح هذا، تأكد من أن *model.training* هي False وأنه لم يتم تنشيط أي إسقاط أثناء تمرير للأمام، أي قم بتمرير *self.training* إلى [الإسقاط الوظيفي لـ PyTorch](https://pytorch.org/docs/stable/nn.functional.html?highlight=dropout#torch.nn.functional.dropout)
أفضل طريقة لإصلاح المشكلة هي عادةً النظر في تمرير للأمام للتنفيذ الأصلي وتنفيذ 🤗 Transformers جنبًا إلى جنب والتحقق مما إذا كان هناك أي اختلافات. تتمثل إحدى الطرق البسيطة والفعالة في إضافة العديد من عبارات الطباعة في كل من التنفيذ الأصلي وتنفيذ 🤗 Transformers، في نفس المواضع في الشبكة على التوالي، وإزالة عبارات الطباعة التي تظهر نفس القيم للعروض التقديمية الوسيطة بشكل تدريجي.

عندما تكون واثقًا من أن كلا التنفيذين يعطيان نفس الإخراج، تحقق من الإخراج باستخدام `torch.allclose(original_output، output، atol=1e-3)`، لقد انتهيت من الجزء الأكثر صعوبة! تهانينا - يجب أن يكون العمل المتبقي سهلاً 😊.

**8. إضافة جميع اختبارات النموذج الضرورية**

في هذه المرحلة، قمت بنجاح بإضافة نموذج جديد. ومع ذلك، من الممكن جدًا ألا يتوافق النموذج بعد مع التصميم المطلوب. للتأكد من أن التنفيذ متوافق تمامًا مع 🤗 Transformers، يجب أن تمر جميع الاختبارات الشائعة. يجب أن يكون Cookiecutter قد أضاف تلقائيًا ملف اختبار لنموذجك، ربما تحت `tests/models/brand_new_bert/test_modeling_brand_new_bert.py`. قم بتشغيل ملف الاختبار هذا للتحقق من أن جميع الاختبارات الشائعة قد مرت:

```bash
pytest tests/models/brand_new_bert/test_modeling_brand_new_bert.py
```

بعد إصلاح جميع الاختبارات الشائعة، من الضروري الآن التأكد من أن جميع الأعمال الجيدة التي قمت بها تم اختبارها جيدًا، بحيث:

- يمكن للمجتمع فهم عملك بسهولة من خلال النظر في اختبارات محددة لـ *brand_new_bert*
- لن تؤدي التغييرات المستقبلية على نموذجك إلى كسر أي ميزة مهمة للنموذج.

أولاً، يجب إضافة اختبارات التكامل. تقوم اختبارات التكامل هذه أساسًا بنفس ما فعلته برامج التصحيح التي استخدمتها سابقًا لتنفيذ النموذج في 🤗 Transformers. أضاف قالب Cookiecutter بالفعل قالبًا لهذه الاختبارات التكاملية، يسمى `BrandNewBertModelIntegrationTests`، ويجب عليك فقط ملؤه. للتأكد من أن هذه الاختبارات تمر، قم بتشغيل ما يلي:

```bash
RUN_SLOW=1 pytest -sv tests/models/brand_new_bert/test_modeling_brand_new_bert.py::BrandNewBertModelIntegrationTests
```

<Tip>

في حالة استخدام Windows، يجب استبدال `RUN_SLOW=1` بـ `SET RUN_SLOW=1`

</Tip>

ثانيًا، يجب اختبار جميع الميزات الخاصة بـ *brand_new_bert* بشكل إضافي في اختبار منفصل تحت `BrandNewBertModelTester`/`BrandNewBertModelTest`. غالبًا ما يتم نسيان هذا الجزء ولكنه مفيد جدًا بطريقتين:

- يساعد على نقل المعرفة التي اكتسبتها أثناء إضافة النموذج إلى المجتمع من خلال إظهار كيفية عمل الميزات الخاصة لـ *brand_new_bert*.
- يمكن للمساهمين المستقبليين اختبار التغييرات على النموذج بسرعة عن طريق تشغيل هذه الاختبارات الخاصة.


**9. تنفيذ برنامج التحليل اللغوي**

بعد ذلك، يجب علينا إضافة برنامج التحليل اللغوي لـ *brand_new_bert*. عادةً ما يكون برنامج التحليل اللغوي مكافئًا أو مشابهًا جدًا لبرنامج تحليل لغوي موجود مسبقًا في 🤗 Transformers.

من المهم جدًا العثور على/استخراج ملف برنامج التحليل اللغوي الأصلي وإدارة تحميل هذا الملف في تنفيذ برنامج التحليل اللغوي لـ 🤗 Transformers.

للتأكد من عمل برنامج التحليل اللغوي بشكل صحيح، يُنصح بإنشاء برنامج نص
لم يتبق سوى القليل! أنت الآن على وشك الانتهاء من إضافة نموذج *brand_new_bert* إلى مكتبة 🤗 Transformers. في هذا الدليل، ستجد جميع الخطوات التي تحتاج إلى اتباعها لإضافة نموذجك الجديد إلى المكتبة.

**9. إضافة رمز التعرف على الرموز**

قد تضطر إلى إلقاء نظرة فاحصة مرة أخرى على المستودع الأصلي للعثور على دالة التعرف على الرموز الصحيحة، أو قد تضطر حتى إلى إجراء تغييرات على نسخة المستودع الأصلي لديك لإخراج `input_ids` فقط. بعد كتابة برنامج نصي للتعرف على الرموز يعمل باستخدام المستودع الأصلي، يجب إنشاء برنامج نصي مماثل لـ 🤗 Transformers. يجب أن يبدو مشابهًا لهذا:

```python
from transformers import BrandNewBertTokenizer

input_str = "This is a long example input string containing special characters .$?-, numbers 2872 234 12 and words."

tokenizer = BrandNewBertTokenizer.from_pretrained("/path/to/tokenizer/folder/")

input_ids = tokenizer(input_str).input_ids
```

عندما تعطي كلا من `input_ids` نفس القيم، يجب أيضًا إضافة ملف اختبار للتعرف على الرموز كخطوة أخيرة.

بالتشابه مع ملفات اختبار النمذجة في *brand_new_bert*، يجب أن تحتوي ملفات اختبار التعرف على رموز *brand_new_bert* على بعض اختبارات التكامل المرمزة ثابتة.

**10. تشغيل اختبارات التكامل الشاملة**

بعد إضافة رمز التعرف على الرموز، يجب أيضًا إضافة بعض اختبارات التكامل الشاملة التي تستخدم كل من النموذج والرمز إلى `tests/models/brand_new_bert/test_modeling_brand_new_bert.py` في 🤗 Transformers.

يجب أن يظهر هذا الاختبار على عينة نصية ذات معنى أن تنفيذ 🤗 Transformers يعمل كما هو متوقع. يمكن أن تتضمن العينة ذات المعنى نصًا مصدريًا إلى زوج ترجمة الهدف، أو مقالًا إلى ملخص، أو سؤالًا إلى إجابة، وما إلى ذلك... إذا لم يتم ضبط دقيق أي من نقاط التفتيش المنقولة على مهمة أسفل النهر، فمن الكافي الاعتماد ببساطة على اختبارات النموذج. كخطوة نهائية لضمان أن النموذج يعمل بشكل كامل، يُنصح بتشغيل جميع الاختبارات على GPU. يمكن أن يحدث أنك نسيت إضافة بعض عبارات `.to(self.device)` إلى المنسوجات الداخلية للنموذج، والتي ستظهر في خطأ في مثل هذا الاختبار. في حالة عدم وجود وصول إلى وحدة معالجة الرسومات (GPU)، يمكن لفريق Hugging Face تشغيل هذه الاختبارات نيابة عنك.

**11. إضافة Docstring**

الآن، تمت إضافة جميع الوظائف اللازمة لـ *brand_new_bert* - أنت على وشك الانتهاء! كل ما تبقى هو إضافة Docstring وصفحة Doc لطيفة. يجب أن يكون Cookiecutter قد أضاف ملف قالب يسمى `docs/source/model_doc/brand_new_bert.md` الذي يجب عليك ملؤه. عادةً ما يلقي مستخدمو نموذجك نظرة على هذه الصفحة قبل استخدام نموذجك. وبالتالي، يجب أن تكون الوثائق مفهومة وموجزة. من المفيد جدًا للمجتمع إضافة بعض "النصائح" لإظهار كيفية استخدام النموذج. لا تتردد في الاتصال بفريق Hugging Face بخصوص Docstrings.

بعد ذلك، تأكد من أن Docstring المضافة إلى `src/transformers/models/brand_new_bert/modeling_brand_new_bert.py` صحيحة وتضمنت جميع المدخلات والمخرجات الضرورية. لدينا دليل مفصل حول كتابة الوثائق وتنسيق Docstring [هنا](writing-documentation). من الجيد دائمًا تذكير المرء بأن الوثائق يجب أن تعامل بعناية مثل الرمز في 🤗 Transformers لأن الوثائق هي عادةً أول نقطة اتصال للمجتمع مع النموذج.

**إعادة هيكلة التعليمات البرمجية**

رائع، الآن أضفت جميع التعليمات البرمجية اللازمة لـ *brand_new_bert*. في هذه المرحلة، يجب عليك تصحيح بعض أساليب التعليمات البرمجية غير الصحيحة المحتملة عن طريق تشغيل:

```bash
make style
```

وتأكد من أن أسلوب الترميز الخاص بك يمر بفحص الجودة:

```bash
make quality
```

هناك عدد قليل من الاختبارات التصميمية الصارمة للغاية الأخرى في 🤗 Transformers التي قد لا تزال تفشل، والتي تظهر في اختبارات طلب السحب الخاص بك. غالبًا ما يكون ذلك بسبب بعض المعلومات المفقودة في Docstring أو بعض التسميات غير الصحيحة. سيساعدك فريق Hugging Face بالتأكيد إذا كنت عالقًا هنا.
أخيرًا، من الجيد دائمًا إعادة هيكلة التعليمات البرمجية الخاصة بك بعد التأكد من أن التعليمات البرمجية تعمل بشكل صحيح. مع اجتياز جميع الاختبارات، الآن هو الوقت المناسب للمرور على التعليمات البرمجية المضافة مرة أخرى وإجراء بعض إعادة الهيكلة.

لقد انتهيت الآن من الجزء البرمجي، تهانينا! 🎉 أنت رائع! 😎

**12. تحميل النماذج إلى مركز النماذج**

في هذا الجزء النهائي، يجب عليك تحويل جميع نقاط التفتيش وتحميلها إلى مركز النماذج وإضافة بطاقة نموذج لكل نموذج تم تحميله نقطة تفتيش. يمكنك التعرف على وظائف المركز من خلال قراءة صفحتنا [مشاركة النموذج والتحميل](model_sharing). يجب أن تعمل جنبًا إلى جنب مع فريق Hugging Face هنا لتحديد اسم مناسب لكل نقطة تفتيش والحصول على حقوق الوصول المطلوبة للتمكن من تحميل النموذج ضمن منظمة مؤلف *brand_new_bert*. تعد طريقة `push_to_hub`، الموجودة في جميع النماذج في `transformers`، طريقة سريعة وفعالة لدفع نقطة تفتيش إلى المركز. يتم لصق جزء صغير أدناه:

```python
brand_new_bert.push_to_hub("brand_new_bert")
# قم بإلغاء التعليق عن السطر التالي لدفعه إلى منظمة.
# brand_new_bert.push_to_hub("<organization>/brand_new_bert")
```

من الجدير بالوقت قضاء بعض الوقت في إنشاء بطاقات نموذج مناسبة لكل نقطة تفتيش. يجب أن تبرز بطاقات النموذج الخصائص المحددة لهذه نقطة تفتيش معينة، على سبيل المثال، ما هي مجموعة البيانات التي تم ضبط نقطة التفتيش مسبقًا/ضبطها بدقة؟ ما هي المهمة التي يجب استخدام النموذج لأسفل النهر؟ ويشمل أيضًا بعض التعليمات البرمجية حول كيفية استخدام النموذج بشكل صحيح.

**13. (اختياري) إضافة دفتر ملاحظات**

من المفيد جدًا إضافة دفتر ملاحظات يوضح بالتفصيل كيفية استخدام *brand_new_bert* للاستدلال و/أو الضبط الدقيق لمهمة أسفل النهر. على الرغم من أن هذا ليس إلزاميًا لدمج طلب السحب الخاص بك، إلا أنه مفيد جدًا للمجتمع.

**14. قدم طلب السحب النهائي**

لقد انتهيت الآن من البرمجة ويمكنك الانتقال إلى الخطوة الأخيرة، وهي دمج طلب السحب الخاص بك في الرئيسي. عادةً ما يكون فريق Hugging Face قد ساعدك بالفعل في هذه المرحلة، ولكن من الجدير بالوقت إعطاء طلب السحب النهائي الخاص بك وصفًا لطيفًا وإضافة تعليقات إلى التعليمات البرمجية الخاصة بك، إذا كنت تريد الإشارة إلى خيارات التصميم معينة لمراجعك.

### شارك عملك!!

الآن، حان الوقت للحصول على بعض الإشادة من المجتمع لعملك! إن إكمال إضافة نموذج يعد مساهمة كبيرة في Transformers ومجتمع NLP بالكامل. سيتم بالتأكيد استخدام التعليمات البرمجية الخاصة بك والنماذج الأولية التي تم تدريبها مسبقًا من قبل المطورين والباحثين. يجب أن تفخر بعملك ومشاركة إنجازاتك مع المجتمع.

**لقد قمت بإنشاء نموذج آخر يسهل الوصول إليه للجميع في المجتمع! 🤯**