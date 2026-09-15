# GA4 / BigQuery Analiz Notları

Funnel ve cohort retention analizleri — herkese açık GA4 e-ticaret veri seti üzerinde.

---

## Amaç ve kapsam

Kullanıcı davranışını olay (event) verisi üzerinden ölçmeyi öğrenmek için yapılmış bir çalışma. İki soruya odaklandım:

1. Kullanıcılar satın alma akışının hangi adımında düşüyor?
2. İlk kez gelen kullanıcıların kaçı geri dönüyor?

Her sorgunun altında ne bulduğumu ve yol boyunca hangi hatayı fark edip düzelttiğimi yazdım. Düzeltmeler burada bilerek duruyor — analizin nasıl olgunlaştığını göstermek analizin kendisi kadar önemli.

**Ortam:** Google Cloud BigQuery Sandbox (ücretsiz katman, aylık 1 TB sorgu kotası).

**Veri:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce` — Google Merchandise Store'un anonimleştirilmiş GA4 verisi. İnceleme aralığı: 1–31 Ocak 2021.

**Verinin yapısı:** Her satır bir olaydır — kullanıcı, oturum veya sipariş değil. Bir kullanıcının onlarca satırı olabilir. Bu yüzden kullanıcı bazlı her soru, önce satırların `user_pseudo_id` altında toplanmasını gerektiriyor.

Tablolar günlük parçalı (`events_20210101`, `events_20210102`, …). `events_*` hepsini tek tablo gibi birleştiriyor, `_TABLE_SUFFIX` filtresi taranan aralığı sınırlıyor. BigQuery dönen satıra değil taranan veriye göre ücretlendirdiği için bu filtre hem maliyet hem hız açısından gerekli.

---

## Sorgu 0 — Keşif: elimde hangi olaylar var?

Funnel kurmadan önce hangi adımların veride mevcut olduğunu görmek gerekiyordu.

```sql
SELECT
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS users
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
GROUP BY event_name
ORDER BY event_count DESC;
```

17 farklı olay türü döndü. Funnel için seçtiklerim:

| Olay | Olay sayısı | Kullanıcı |
|---|---|---|
| session_start | 116.549 | 93.552 |
| view_item | 86.971 | 19.629 |
| add_to_cart | 15.522 | 3.832 |
| begin_checkout | 11.034 | 1.924 |
| purchase | 1.204 | 1.069 |

Olay sayısı ile kullanıcı sayısının ayrışması beklenen bir şey: bir kullanıcı aynı olayı defalarca tetikleyebilir. `page_view` tarafında oran daha da belirgin — 419.004 olay, 94.630 kullanıcı, yani kişi başı ortalama dört-beş sayfa.

---

## Sorgu 1 — Funnel

**Soru:** Satın alma akışında kullanıcılar hangi adımda düşüyor?

```sql
WITH kullanici_adimlari AS (
  SELECT
    user_pseudo_id,
    MAX(IF(event_name = 'session_start',  1, 0)) AS s1_oturum,
    MAX(IF(event_name = 'view_item',      1, 0)) AS s2_urun_gordu,
    MAX(IF(event_name = 'add_to_cart',    1, 0)) AS s3_sepete_ekledi,
    MAX(IF(event_name = 'begin_checkout', 1, 0)) AS s4_odemeye_basladi,
    MAX(IF(event_name = 'purchase',       1, 0)) AS s5_satin_aldi
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
  GROUP BY user_pseudo_id
)
SELECT
  COUNTIF(s1_oturum = 1) AS adim1_oturum,
  COUNTIF(s1_oturum = 1 AND s2_urun_gordu = 1) AS adim2_urun,
  COUNTIF(s1_oturum = 1 AND s2_urun_gordu = 1 AND s3_sepete_ekledi = 1) AS adim3_sepet,
  COUNTIF(s1_oturum = 1 AND s2_urun_gordu = 1 AND s3_sepete_ekledi = 1
          AND s4_odemeye_basladi = 1) AS adim4_odeme,
  COUNTIF(s1_oturum = 1 AND s2_urun_gordu = 1 AND s3_sepete_ekledi = 1
          AND s4_odemeye_basladi = 1 AND s5_satin_aldi = 1) AS adim5_satin_alma
FROM kullanici_adimlari;
```

**Yöntem notu.** `MAX(IF(...))` kalıbı, kullanıcının onlarca olay satırını tek bir 1/0 bayrağına indiriyor: en az bir kez o olayı yaptıysa 1, hiç yapmadıysa 0. `GROUP BY user_pseudo_id` ile her kullanıcı tek satıra iniyor. Dıştaki `COUNTIF` bu bayrakları huni sırasına göre sayıyor — her adımda önceki adımların şartı da yazıldığı için gerçek bir geçiş dizisi çıkıyor.

### Sonuç

| Adım | Kullanıcı | Bir önceki adımdan geçiş |
|---|---|---|
| Oturum açtı | 93.552 | — |
| Ürün gördü | 19.262 | %20,6 |
| Sepete ekledi | 3.778 | %19,6 |
| Ödemeye başladı | 1.439 | %38,1 |
| Satın aldı | 845 | %58,7 |

Uçtan uca dönüşüm: **%0,90**.

### Bulgu 1 — Kayıp huninin üstünde

Girenlerin %79'u tek bir ürün sayfasına bile gitmeden çıkıyor. Buna karşılık huninin alt basamaklarında oranlar sürekli iyileşiyor: sepete ekleyenin %38'i ödemeye başlıyor, ödemeye başlayanın %59'u satın alıyor.

Bu şekil, sorunun ödeme akışında değil trafiğin niteliğinde olduğunu düşündürüyor. Niyeti olan kullanıcı akışı tamamlayabiliyor; sorun o niyete sahip kullanıcının azlığı. Doğrulamak için trafik kaynağı kırılımı gerekir — bu çalışmanın kapsamı dışında bıraktım.

### Bulgu 2 — Funnel tanımı sonucu değiştiriyor

Aynı adımları iki farklı tanımla saydım:

- **Katı:** kullanıcı önceki adımların hepsini de yapmış olmalı.
- **Şartsız:** kullanıcı o olayı yapmış olsun, öncesi önemli değil (Sorgu 0'daki `users` sütunu).

| Adım | Katı | Şartsız | Fark |
|---|---|---|---|
| Ürün gördü | 19.262 | 19.629 | %2 |
| Sepete ekledi | 3.778 | 3.832 | %1 |
| Ödemeye başladı | 1.439 | 1.924 | **%25** |
| Satın aldı | 845 | 1.069 | %21 |

İlk iki adımda fark ihmal edilebilir, ödeme adımında birden sıçrıyor: 485 kullanıcı, sepete ekleme kaydı olmadan ödemeye başlamış.

Olası açıklamalar: önceki aydan devreden kayıtlı sepet, tek tıkla satın alma gibi alternatif bir akış, veya `add_to_cart` olayının bazı durumlarda tetiklenmemesi. Hangisi olduğu tek başına veriden çıkmıyor; ürün ve geliştirme ekibine sorulacak bir soru. Ama sorabilmek için önce iki tanımı karşılaştırmak gerekiyordu.

---

## Sorgu 2 — Cohort retention

**Soru:** İlk kez gelen kullanıcıların kaçı sonraki günlerde geri dönüyor?

### İlk deneme (hatalı)

```sql
WITH ilk_gunler AS (
  SELECT
    user_pseudo_id,
    MIN(PARSE_DATE('%Y%m%d', event_date)) AS ilk_gun
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
  GROUP BY user_pseudo_id
),
aktif_gunler AS (
  SELECT DISTINCT
    user_pseudo_id,
    PARSE_DATE('%Y%m%d', event_date) AS aktif_gun
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
)
SELECT
  DATE_DIFF(a.aktif_gun, i.ilk_gun, DAY) AS gun_no,
  COUNT(DISTINCT a.user_pseudo_id) AS kullanici
FROM aktif_gunler a
JOIN ilk_gunler i USING (user_pseudo_id)
GROUP BY gun_no
ORDER BY gun_no;
```

**Yöntem notu.** `event_date` metin olarak tutuluyor (`'20210119'`), tarih aritmetiği için önce `PARSE_DATE` ile gerçek tarihe çevrilmesi gerekiyor. İki CTE'den ilki her kullanıcının başlangıç gününü, ikincisi aktif olduğu her günü veriyor; `DATE_DIFF` ikisinin farkını alarak "ilk günden kaç gün sonra" sorusunu cevaplıyor.

Sonuç: D0 94.790 kullanıcı, D1 %4,15, D7 %0,56, D14 %0,22.

### Hata: maruz kalma yanlılığı

Eğrinin 13. günde 179, 14. günde 213 kullanıcıya çıktığını fark ettim. Monoton düşmesi gereken bir eğride bu sıçrama olmamalıydı.

Nedeni: kohortlar eşit değil. 1 Ocak'ta gelen kullanıcının 14. gününü gözlemleyebiliyoruz, 30 Ocak'ta gelenin ise hiç şansı yok — veri 31'de bitiyor. Yani ileri günlerdeki sayılar "dönmeyenler" ile "dönme fırsatı hiç olmayanlar"ı birbirine karıştırıyor ve eğri gerçekte olduğundan kötü görünüyor. Aynı anda farklı kohortların üst üste binmesi de hafta içi/hafta sonu etkisini bulaştırıyor.

### Düzeltilmiş hali

Tek bir başlangıç gününe sabitledim, böylece bütün kullanıcıların önünde eşit pencere kaldı:

```sql
WITH ilk_gunler AS (
  SELECT
    user_pseudo_id,
    MIN(PARSE_DATE('%Y%m%d', event_date)) AS ilk_gun
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
  GROUP BY user_pseudo_id
  HAVING ilk_gun = DATE '2021-01-01'
),
aktif_gunler AS (
  SELECT DISTINCT
    user_pseudo_id,
    PARSE_DATE('%Y%m%d', event_date) AS aktif_gun
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
)
SELECT
  DATE_DIFF(a.aktif_gun, i.ilk_gun, DAY) AS gun_no,
  COUNT(DISTINCT a.user_pseudo_id) AS kullanici
FROM aktif_gunler a
JOIN ilk_gunler i USING (user_pseudo_id)
GROUP BY gun_no
ORDER BY gun_no;
```

`HAVING` kullanmak zorunlu: `ilk_gun` bir aggregate sonucu olduğu için gruplama sonrası oluşuyor, `WHERE` ise gruplama öncesi çalışıyor.

### Sonuç ve karşılaştırma

| Gün | Karışık kohort | 1 Ocak kohortu |
|---|---|---|
| 0 | 94.790 | 2.160 |
| 1 | %4,15 | %3,66 |
| 2 | %1,63 | %1,57 |
| 7 | %0,56 | %0,56 |
| 14 | %0,22 | %0,14 |

İlk günlerde iki yöntem birbirine yakın, ileri günlerde ayrışıyor — beklenen davranış. Karışık versiyon 14. günü olduğundan iyi gösteriyordu. 13-14. gündeki sıçrama da düzeltilmiş versiyonda kayboldu, yani teşhis doğruydu.

### Bulgu 3 — Düzeltmenin bedeli

Kohortu sabitlemek doğruluğu artırdı ama örneklemi 94.790'dan 2.160'a düşürdü. Bunun sonucu ileri günlerde görülüyor: 10. günde 4, 11. günde 8 kullanıcı. Bu dalgalanma artık kohort karışıklığından değil, küçük sayıların gürültüsünden kaynaklanıyor — dört kişiyle sekiz kişi arasındaki fark yorumlanabilir değil.

Ayrıca 1 Ocak kohort başlangıcı olarak ideal değil: yılbaşı günü, bir e-ticaret sitesinde tipik trafik göstermez. Sıradan bir iş gününü (ör. 12 Ocak, salı) başlangıç alıp sonucun ne kadar değiştiğine bakmak, bu analizin doğal sonraki adımı.

---

## Sınırlamalar

- **Zaman penceresi.** Analiz yalnızca Ocak 2021 ile sınırlı. Aralık ayında ürüne bakıp ocakta satın alan bir kullanıcı, funnel'da adımları atlamış gibi görünür. Şemadaki `user_first_touch_timestamp` alanı gerçek ilk temas anını tutuyor; kohort tanımını ay penceresi yerine bu alana dayandırmak daha doğru olurdu.
- **Kimlik.** `user_pseudo_id` cihaz/tarayıcı bazlı. Telefonda gezip bilgisayarda satın alan kullanıcı iki ayrı kişi olarak sayılır.
- **Ölçümleme.** Olayların eksik tetiklenmesi ihtimali dışarıdan doğrulanamaz; Bulgu 2'deki farkın bir kısmı bundan kaynaklanıyor olabilir.
- **Tek veri seti.** Bulgular bu örnek veri setine özgüdür, genel bir e-ticaret davranışı iddiası taşımaz.

## Çalışmanın maliyeti

Bir aylık pencerede funnel sorgusu ~38,7 MB tarıyor. BigQuery taranan veri üzerinden ücretlendirdiği için `_TABLE_SUFFIX` filtresi ve gereksiz sütunlardan kaçınmak doğrudan maliyet kalemi. Tüm çalışma Sandbox'ın ücretsiz kotası içinde kaldı.
