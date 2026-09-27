# Note: Predict Household Income Across Regions

## Dataset qeydi
Layihə əvvəlcə Azərbaycan rayonları üçün nəzərdə tutulmuşdu, lakin rayon səviyyəsində açıq demoqrafik/iqtisadi data tapmaq mümkün olmadığından, task.md-də göstərilən **US Census Demographic Data** (Kaggle, muonneutrino) dataseti ilə davam edildi. Metodologiya eynidir, yalnız coğrafi vahid "rayon" əvəzinə ABŞ-ın **county**-lərini əhatə edir (3220 county).

## Data hazırlığı
- **Target:** `Income` (median household income, county üzrə)
- **Leakage səbəbindən çıxarılan sütunlar:** `IncomePerCap`, `IncomePerCapErr`, `IncomeErr` — bunlar riyazi olaraq Income-dən törəndiyi üçün model üçün "saxta" (leaked) siqnal yaradırdı.
- **Boş dəyər:** `ChildPoverty` sütununda 1 boş dəyər var idi (Kalawao County, Hawaii — ABŞ-ın ən kiçik county-si, cəmi 86 əhali). Median ilə dolduruldu (median seçildi, çünki datada kəskin outlier-lər var idi, məsələn `TotalPop`-da min=74, max=10 milyon — median bu cür ekstremal dəyərlərdən mean-ə nisbətən daha az təsirlənir).
- **Multicollinearity:** `Poverty` və `ChildPoverty` demək olar eyni məlumatı daşıyırdı (korrelyasiya hər ikisində ~-0.75). `ChildPoverty` modeldən çıxarıldı — RMSE demək olar dəyişmədi (4905 → 4897), bu da onun redundant olduğunu təsdiqlədi.

## Model
- **Alqoritm:** Random Forest Regressor (regression problemi, çünki Income davamlı ədəddir)
- **Train/Test split:** 80/20, `random_state=42` (nəticələrin təkrarlana bilməsi üçün)
- **Nəticə: RMSE ≈ 4897 dollar**, bu, ortalama gəlirin (~48,995$) təxminən **10%-inə** bərabərdir — qəbuledilən, yaxşı səviyyəli nəticə.

## Feature Importance
| Feature | Əhəmiyyət |
|---|---|
| Poverty | 0.629 (üstünlük təşkil edir) |
| Asian | 0.088 |
| Professional | 0.078 |
| Employed | 0.044 |
| Digərləri | hər biri <0.02 |

`Poverty` təkbaşına modelin qərarlarının ~63%-ni izah edir. Maraqlıdır ki, sadə korrelyasiyada zəif görünən `Asian` sütunu (korrelyasiya 0.45) feature importance-da `Professional`-ı (korrelyasiya 0.59) keçdi — bu, Random Forest-in xətti korrelyasiyanın tuta bilmədiyi qeyri-xətti/kombinasiya əlaqələrini də tuta bildiyini göstərir.

## Fairness Analizi
State-səviyyəli müqayisə (ilkin cəhd) etibarsız çıxdı, çünki bəzi state-lərdə test nümunəsi ölçüsü çox kiçik idi (məsələn Wyoming=3, Hawaii=1) — bu, kiçik say səbəbindən süni şəkildə ekstremal orta xətalara səbəb oldu.

Düzgün yanaşma: county-ləri `White` faizinin medianına görə iki bərabər qrupa böldük (hər qrupda 322 nümunə):

| Qrup | Orta Mütləq Xəta |
|---|---|
| Aşağı White % (azlıq irqi qrupların çoxluq təşkil etdiyi) | 3563 $ |
| Yüksək White % | 3215 $ |

**Nəticə:** Model, azlıq irqi qruplarının üstünlük təşkil etdiyi county-lərdə təxminən 10-11% daha yüksək proqnoz xətası göstərir. Fərq kəskin deyil, amma diqqətəlayiqdir — model tam neytral deyil. Bunun səbəbi, irqlə statistik əlaqəli digər feature-lar (`Construction`, `PublicWork`, `Drive` və s.) vasitəsilə modelin dolayı yolla irqi profil "öyrənməsi" ola bilər (fairness through unawareness problemi — irq sütununu çıxarmaq təkbaşına qərəzi aradan qaldırmır).

## Choropleth Xəritə
Real gəlir və model proqnozları ABŞ county sərhədləri üzərində vizuallaşdırıldı (Plotly + GeoJSON, FIPS kodları ilə uyğunlaşdırılaraq). İlkin vizual müqayisədə proqnoz xəritəsi real xəritədən daha "səs-küylü" görünürdü, lakin bu, fərqli miqyasda (zoom) çəkilmiş şəkillərin aldadıcı müqayisəsi idi. Kəmiyyət yoxlaması (state-daxili standart sapma: real 10504 vs proqnoz 9821) göstərdi ki, proqnozlar real gəlirə nisbətən bənzər, hətta bir qədər daha az dəyişkəndir.

## Ümumi Nəticələr
1. Yoxsulluq faizi (Poverty) gəlirin ən güclü prediktorudur.
2. Model kifayət qədər dəqiqdir (~10% xəta), amma mükəmməl deyil.
3. Model tam "kor" deyil — azlıq irqi qrupların çoxluq təşkil etdiyi bölgələrdə bir qədər daha çox səhv edir, bu, məsuliyyətli AI tətbiqi üçün diqqət tələb edən bir tapıntıdır.
4. Modelin coğrafi/məkan məlumatı (qonşuluq) olmadığı üçün, proqnozlar arasında bəzi qonşu ərazilər gözləniləndən fərqli çıxa bilər — gələcəkdə `latitude`/`longitude` feature-ları əlavə etməklə bu araşdırıla bilər.
