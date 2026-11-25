# Лаборатори №6 — Category-Partition ба PICT

**Оюутан:** Н.Мөнхбаяр  
**Код:** B242270058  

# Орчин

debain13@debian:~/Lab06-Gobi-Bagsh$ dpkg -l git cmake g++ make | grep '^ii' | awk '{print $2, $3}'

cmake 3.31.6-2

g++ 4:14.2.0-1

git 1:2.47.3-0+deb13u1

make 4.4.1-2


---

## Алхам 1: PICT-ийг эх кодоос бүтээх

PICT-ийг Microsoft-ийн GitHub репогоос эх кодоос нь бүтээж, системд суулгасан. PICT-ийн эх код лабын репоос гадна (`~/tools/pict`) байрлана.

```bash
# 1. Хамааралтай багцуудыг суулгах
sudo apt update && sudo apt install -y git cmake g++ make

# 2. PICT-ийг лабын репоосоо ГАДНА clone хийх
mkdir -p ~/tools && cd ~/tools
git clone https://github.com/microsoft/pict
cd pict

# 3. CMake-ээр бүтээх
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j4

# 4. Системд суулгах
sudo cp build/cli/pict /usr/local/bin/

# 5. Нотолгоог текст файлаар хадгалах
mkdir -p results
(git -C ~/tools/pict log -1 --format='pict commit %h %ad' --date=short; pict 2>&1 | head -3) | tee results/pict-build.txt
```

Нотолгоо: [`results/pict-build.txt`](results/pict-build.txt) (pict commit `ab76c25` ба хэрэглээний заавар). Зураг: `Screenshot/`.

---

## Хэсэг A: Category-Partition гараар — Шилжүүлгийн функц

### Давуу эрэмбэ (нэг хүсэлтэд олон алдаа зэрэг байвал)

Тодорхойлолтод аль алдаа буцахыг зориуд бичээгүй тул давуу эрэмбийг дараах байдлаар тогтоов: **from → to → amount**, amount дотор **BAD_AMOUNT → INSUFFICIENT → LIMIT**. Үндэслэл нь данс байхгүй, царцаагдсан бол үлдэгдэл, лимитийг шалгах боломжгүй тул эхлээд оролцогч талуудыг, дараа нь дүнг шалгах нь логик дараалал мөн.

### A1. Сонголт ба төлөөлөх утгууд

| Параметр / сонголт | Төлөөлөх утгууд |
|---|---|
| **from** | Valid (байгаа, царцаагдаагүй), Frozen (царцаагдсан), Missing (байхгүй) |
| **to** | Valid (байгаа), Missing (байхгүй), SameAsFrom (from-той ижил) |
| **amount** | Normal (>0, үлдэгдэл ба лимитэд багтана), ExactBalance (яг үлдэгдэлтэй тэнцүү), Zero (≤ 0), OverBalance (үлдэгдлээс их), OverLimit (өдрийн лимитээс их) |
| **currency** | Same (дансны валюттай ижил), Different (өөр валют, ханшийн хөрвүүлэлттэй) |
| **Далд орчин** | Дансны үлдэгдэл, өдрийн лимитийн ашиглалт, ханш |

Далд орчны сонголтыг тест бүрийн setup-д тохируулна. Жишээ нь OverLimit-ийг "хэт том дүн" эсвэл "лимит ихэнхдээ дүүрсэн байхад жижиг дүн" гэсэн хоёр аргаар үүсгэж болно. Өдрийн лимит хэр дүүрсэн, ямар ханш байсан нь `OverBalance`, `OverLimit`, `Different`-ийн үр дүнг шууд тодорхойлдог.

### A2. Хязгаарлалтууд

| Утга | Тэмдэглэгээ | Үндэслэл |
|---|---|---|
| from = Frozen | [ERROR] | Царцаагдсан данснаас шилжүүлэхийг дүрэм хориглодог (ERROR_FROZEN). |
| from = Missing | [ERROR] | Данс байхгүй тул шилжүүлэг боломжгүй (ERROR_NO_FROM). |
| to = Missing | [ERROR] | Хүлээн авагч данс байхгүй (ERROR_NO_TO). |
| to = SameAsFrom | [ERROR] | to нь from-оос өөр байх ёстой (ERROR_SAME_ACCOUNT). |
| amount = Zero | [ERROR] | amount > 0 байх ёстой (ERROR_BAD_AMOUNT). |
| amount = OverBalance | [ERROR] [IF from=Valid, to=Valid] | Үлдэгдэл нь зөвхөн хүчинтэй данс дээр байна, мөн давуу эрэмбээр данс шалгах нь түрүүлнэ (ERROR_INSUFFICIENT). |
| amount = OverLimit | [ERROR] [IF from=Valid, to=Valid] | Лимит нь зөвхөн хүчинтэй данс дээр хэрэглэгдэнэ (ERROR_LIMIT). |
| amount = ExactBalance | [SINGLE] [IF from=Valid, to=Valid] | Хүчинтэй хязгаарын утга (дүрэмд "хэтрэхгүй" гэсэн тул амжилттай). Нэг тест хангалттай, нөгөө параметрүүдтэй хосолж давтах шаардлагагүй. |
| from = Valid, to = Valid, amount = Normal, currency = Same / Different | — | Хэвийн утгууд, өөр хоорондоо хосолно. |

**Тооцоо (N → M):**

- Бүрэн олонлог: N = 3 × 3 × 5 × 2 = **90**
- Хязгаарлалтын дараа: ERROR 7 (2 + 2 + 3) нэг нэгээр, SINGLE 1, хэвийн утгууд 1 × 1 × 1 × 2 = 2. Нийт M = 7 + 1 + 2 = **10**

**90 → 10**

### A3. Спецификацийн хүснэгт

| # | from | to | amount | currency | Хүлээгдэх үр дүн | Тайлбар |
|---|---|---|---|---|---|---|
| 1 | Valid | Valid | Normal | Same | 201 OK | Happy path |
| 2 | Valid | Valid | Normal | Different | 201 OK | Happy path, ханшийн хөрвүүлэлттэй |
| 3 | Valid | Valid | ExactBalance | Same | 201 OK | [SINGLE] хязгаарын утга |
| 4 | Frozen | Valid | Normal | Same | 200 ERROR_FROZEN | |
| 5 | Missing | Valid | Normal | Same | 200 ERROR_NO_FROM | |
| 6 | Valid | Missing | Normal | Same | 200 ERROR_NO_TO | |
| 7 | Valid | SameAsFrom | Normal | Same | 200 ERROR_SAME_ACCOUNT | |
| 8 | Valid | Valid | Zero | Same | 200 ERROR_BAD_AMOUNT | |
| 9 | Valid | Valid | OverBalance | Same | 200 ERROR_INSUFFICIENT | |
| 10 | Valid | Valid | OverLimit | Same | 200 ERROR_LIMIT | |
| 11 | Valid | Valid | OverBalance (хөрвүүлсний дараа) | Different | 200 ERROR_INSUFFICIENT | Сонирхолтой нөхцөл: нэрлэсэн дүн бага ч ханшаар хөрвүүлсний дараа үлдэгдлээс хэтэрнэ |
| 12 | Missing | Missing | Normal | Same | 200 ERROR_NO_FROM | Давхар нөхцөл: from түрүүлнэ |
| 13 | Frozen | SameAsFrom | Normal | Same | 200 ERROR_FROZEN | Давхар нөхцөл: from түрүүлнэ |

---

## Хэсэг B: PICT — pairwise covering array

### B1. Браузерын жишээ

Модель: [`browser.model`](browser.model). Гаралт: [`results/browser-2way.txt`](results/browser-2way.txt), [`results/browser-3way.txt`](results/browser-3way.txt).

| Хувилбар | Мөрийн тоо | Бүрэн олонлог |
|---|---|---|
| Гараар (лекц) | 9 | 144 |
| PICT 2-way | **10** | 144 |
| PICT 3-way (`/o:3`) | **20** | 144 |

**Greedy-гийн тайлбар.** PICT нь хос бүрийг аль болох олныг шинээр хамарсан мөрийг дараалан сонгодог greedy алгоритм ашигладаг. Алхам бүрийн локал хамгийн сайн сонголт нь нийт хамгийн цөөн мөрийг баталдаггүй тул PICT 10 мөр өгсөн, лекцэд гараар олсон 9-өөс нэгээр илүү. Хамгийн цөөн covering array олох нь тооцооллын хувьд хэцүү бодлого тул PICT хурд болон хангалттай сайн үр дүнг сонгосон. 3-way нь 144 бүрэн олонлогийг 20 мөр (ойролцоогоор 14%) болгож багасгасан бөгөөд хамрах хэмжээ нэмэгдэх тусам мөрийн тоо хурдан өснө.

### B2. Шилжүүлгийн модель

Файлууд: [`transfer-nc.model`](transfer-nc.model) (хязгаарлалтгүй), [`transfer.model`](transfer.model) (A2-ын хязгаарлалттай), [`transfer-neg.model`](transfer-neg.model) (`~` сөрөг утгатай).

`transfer.model`-ийн хязгаарлалт:

```
IF [From] <> "Valid" OR [To] <> "Valid" THEN [Amount] = "Normal";
```

Энэ нь A2-ын [IF] дүгнэлтийг тусгана: үлдэгдэл, лимитийн утгууд (OverBalance, OverLimit, ExactBalance) нь зөвхөн from, to хоёулаа хүчинтэй үед л утга учиртай.

`transfer-neg.model`-ийн хязгаарлалт:

```
IF [From] <> "Valid" OR [To] <> "Valid" THEN [Amount] <> "ExactBalance";
```

**Харьцуулалт:**

| Хувилбар | Файл | Мөрийн тоо | Happy path мөр | Нэг мөр дэх алдаа |
|---|---|---|---|---|
| Хязгаарлалтгүй | `results/transfer-nc.txt` | 15 | 0 | 1–3 (1 алдаатай 5, 2 алдаатай 6, 3 алдаатай 4 мөр) |
| Хязгаарлалттай | `results/transfer.txt` | 16 | 2 | 0–2 (0: 2, 1: 10, 2: 4 мөр) |
| Сөрөг (`~`) | `results/transfer-neg.txt` | 18 | 4 | 0–1 (0: 4, 1: 14 мөр) |

"Happy path" гэдэгт from, to хоёулаа Valid, amount нь Normal эсвэл ExactBalance, бусад алдаа байхгүй мөрийг тоолсон.

**Гаралтаас харсан зүйл:**

- Хязгаарлалтгүй `transfer-nc`-д happy path мөр огт гараагүй, бүх мөрөнд 1–3 алдаа орсон. Ийм мөр нь алдаа бусдыгаа нууж (error masking) аль шалгалт ажиллаж байгааг ялгах боломжгүй.
- `transfer.txt`-д Valid / Valid / **Normal** гэсэн мөр байхгүй, зөвхөн ExactBalance-тай 2 мөр happy path болсон. Pairwise нь гурав ба түүнээс дээш параметрийн хослолыг шаарддаггүй тул ийм зүйл гарна.
- Хязгаарлалттай хувилбарт нэг мөрөнд хамгийн ихдээ 2 алдаа (данс + данс) үлдсэн. Эдгээр нь давуу эрэмбийг шалгадаг давхар нөхцөл.
- `transfer-neg` нь нэг мөрөнд хамгийн ихдээ 1 алдаа оруулсан тул мөр бүр нэг алдааны ангийг цэвэр шалгана. 4 happy path мөр гарсан.

**`~` хэрэглэхэд A2-ын хязгаарлалт хэрэгтэй хэвээр үү?** Масклалтыг (олон алдаа) таслах хязгаарлалт хэрэггүй болсон, учир нь `~` нь нэг мөрөнд ≤ 1 сөрөг утга оруулдаг. Харин [IF] төрлийн утга учрын хязгаарлалт хэрэгтэй хэвээр. Үүнгүйгээр ExactBalance нь ~Missing from-той хосолж, үлдэгдэл байхгүй данс дээр "яг үлдэгдэл" гэсэн утгагүй мөр гарна. Тиймээс `transfer-neg.model`-д ExactBalance-ийн хязгаарлалтыг үлдээсэн.

### Бодит тест кейс (гаралтын мөрүүдээс)

Ханш: 1 USD = 3500 MNT. A1 нь MNT данс (үлдэгдэл 1,000,000; өдрийн лимит 5,000,000, ашигласан 0), A2 байгаа данс.

| # | PICT мөр | Setup | Оролт (transfer body) | Oracle |
|---|---|---|---|---|
| 1 | `transfer-neg.txt`: `Valid  Valid  Normal  Different` (happy path) | A1 хүчинтэй, A2 хүчинтэй | `{"from":"A1","to":"A2","amount":100,"currency":"USD"}` (350,000 MNT) | 201 `{"result":"OK"}` |
| 2 | `transfer-neg.txt`: `Valid  Valid  ~OverBalance  Different` (нэг алдаатай) | Ижил | `{"from":"A1","to":"A2","amount":500,"currency":"USD"}` (1,750,000 MNT > 1,000,000) | 200 `{"result":"ERROR_INSUFFICIENT"}` |
| 3 | `transfer.txt`: `Frozen  SameAsFrom  Normal  Same` (олон алдаатай) | A1 царцаагдсан | `{"from":"A1","to":"A1","amount":100000,"currency":"MNT"}` | 200 `{"result":"ERROR_FROZEN"}` |

Тест 3-ын oracle нь A хэсэгт тогтоосон давуу эрэмбээс (from → to) гарна: from шалгалт түрүүлэх тул ERROR_SAME_ACCOUNT биш ERROR_FROZEN буцна.

### B3. Дүгнэлт

Гараар хийсэн A3 хүснэгт нь алдааны семантикийг илүү сайн барьдаг: мөр бүрт хүлээгдэх үр дүн, давуу эрэмбэ, ERROR/SINGLE ялгаа тодорхой бөгөөд хязгаарын утга (ExactBalance), хөрвүүлсний дараах шалгалт зэрэг утга учиртай тохиолдлыг хүн өөрөө бодож оруулна. Гэхдээ гараар хийх нь том параметрийн олонлогт хэцүү, хос бүрийн хамралтыг баталгаажуулах арга байхгүй. PICT нь эсрэгээрээ хосын хамралтыг математикаар баталгаажуулдаг: 90 хослолыг 15–18 мөрөнд багтаасан. Гэхдээ PICT нь алдааны утга учрыг ойлгодоггүй тул хязгаарлалтгүй бол нэг мөрөнд 3 алдаа хүртэл хольж error masking үүсгэсэн. `~` тэмдэглэгээ нь нэг мөрөнд нэг алдаа оруулж, category-partition-ийн [ERROR]-тай дүйцсэн ч [IF] төрлийн хязгаарлалтыг орлохгүй. Мөн PICT-ийн гаралт oracle өгдөггүй тул хүлээгдэх үр дүнг давуу эрэмбээс өөрөө гаргах хэрэгтэй. Greedy алгоритмын улмаас PICT (10 мөр) нь гараар олсон 9 мөрөөс бага зэрэг илүү гарсан ч хурдтай, давтагдах боломжтой. Хамгийн их бодол шаардсан алхам нь давуу эрэмбийг тогтоох, түүгээр олон алдаатай мөрийн oracle-ийг үндэслэх явдал байлаа. Практикт хоёр аргыг хослуулах нь зөв: category-partition-оор утга, хязгаарлалт, oracle-оо тодорхойлж, PICT-ээр хослолыг автоматаар үүсгэх.

---

## Репозиторийн бүтэц

```
README.md
browser.model  transfer-nc.model  transfer.model  transfer-neg.model
results/  pict-build.txt  browser-2way.txt  browser-3way.txt
          transfer-nc.txt  transfer.txt  transfer-neg.txt
Screenshot/  (PICT build-ийн дэлгэцийн зураг)
```
