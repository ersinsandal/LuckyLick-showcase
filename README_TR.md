# LuckyLick — Türkçe Teknik Dokümantasyon

> **⚠️ Bu depo yalnızca dokümantasyon amacıyla kamuya açıktır.**  
> Kaynak kodların tamamı private bir depoda tutulmaktadır.  
> İletişim: [SliferSoft](mailto:info@slifersoft.com)

🇬🇧 [Click here for English documentation → README.md](./README.md)

---

## Kısaca Ne Yapıyor?

LuckyLick, şans tabanlı bir içerik keşif platformudur. Kullanıcının tek bir etkileşimiyle, derlenmiş bir havuzdan rastgele bir yetişkin video seçilir ve doğrudan tarayıcı üzerinden akıtılır. Platformun belirleyici özelliği, Türkiye'de ISP düzeyinde erişime kapatılmış sitelerden gelen içerikleri VPN, tarayıcı eklentisi veya herhangi bir istemci taraflı yapılandırma gerektirmeksizin sunabilmesidir. Sansür aşma işlemi tamamen sunucu tarafında gerçekleşir ve son kullanıcı açısından şeffaf biçimde işler.

---

## Proje Nedir?

**LuckyLick**, ISP (İnternet Servis Sağlayıcı) düzeyinde uygulanan DNS engellerini **VPN kullanmadan** aşmayı deneysel olarak kanıtlamak amacıyla geliştirilmiş bir web platformudur.

Projenin temel tezi şudur:

> *Standart bir tarayıcı üzerinden, herhangi bir VPN, tarayıcı eklentisi veya kullanıcı taraflı yapılandırma gerektirmeksizin, yalnızca sunucu tarafı DNS tünellemesi ve HTTPS proxy yönlendirmesi ile ISP tarafından engellenen içeriklere erişilebilir.*

### İsmin Anlamı

**LuckyLick** adı bilinçli olarak çok katmanlı seçilmiştir:

- **Lucky** + **Click** → *LuckyLick* — Temel UX mekaniği tek tıklamayla öngörülemeyen bir sonuç elde etmektir; tıpkı zar atmak gibi. "Luck" şansı, "click" tıklamayı ifade eder; ikisi birleşince "LuckyLick" olur.
- **Lick** kelimesi aynı zamanda erotik bir çağrışım taşır — projenin test alanı olan yetişkin içerik platformlarına markada örtük biçimde yapılan bir göndermedir.

İsim; akılda kalıcı, hafifçe kışkırtıcı ve hem teknik mekanizmayı hem de içerik alanını doğru yansıtacak şekilde tasarlanmıştır.

---

## Teknik Problem

Türkiye'deki ISP'ler belirli web sitelerine erişimi iki temel yöntemle engeller:

1. **DNS Zehirleme (DNS Poisoning / Sinkholing):** ISP'nin DNS çözümleyicisine yapılan sorgular sahte yanıtlarla karşılanır; gerçek IP adresi yerine engellenmiş bir adrese ya da boş yanıta yönlendirilir.
2. **Derin Paket İncelemesi (Deep Packet Inspection — DPI):** Bazı ISP'ler, paket içeriğini analiz ederek belirli alan adlarına giden bağlantıları TCP düzeyinde keser.

LuckyLick bu sorunları aşmak için **yerel çalışan bir Python proxy sunucusu** kullanır. Bu sunucu:

1. **DNS sinkhole'larını atlatır:** Sabit kodlu IP haritası ve Cloudflare/Google DNS-over-HTTPS (DoH) yedeklemesiyle engelleme dışı kalır.
2. **Doğrudan HTTPS bağlantısı kurar:** Gerçek IP adresleri üzerinden, doğru TLS SNI (`_server_hostname`) ve geçerli `Origin`/`Referer` başlıklarıyla engelli sunuculara bağlanır.
3. **Tüm alt kaynakları yeniden yazar:** HTML içindeki script, resim ve medya URL'leri tarayıcı göndermeden önce proxy üzerinden geçecek şekilde düzenlenir.
4. **Medyayı doğal şekilde akıtır:** HTTP 206 Partial Content ve `Range` başlık desteğiyle chunked streaming sağlanır; tampon yok, yeniden kodlama yok.
5. **Minimal SPA arayüzü sunar:** `http://localhost:5500` açıldığında hiçbir ayar gerekmeden çalışır.

---

## Mimari Genel Bakış

```
Tarayıcı (localhost:5500)
        │
        ▼
┌───────────────────────────────────────────┐
│   Python ThreadingHTTPServer (port 5500)   │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │  Statik Dosya Sunumu (SPA)          │  │
│  │  index.html, script.js, styles.css  │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │  /api/stream_info                   │  │
│  │  • Embed sayfası HTML'i çeker       │  │
│  │  • Doğrudan MP4/HLS URL'i çıkarır  │  │
│  │  • Tüm kaynak URL'lerini proxy'ler  │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │  /api/proxy?url=<hedef>             │  │
│  │  • Şeffaf ters proxy                │  │
│  │  • DoH tabanlı DNS çözümlemesi      │  │
│  │  • SNI + doğru başlıklar            │  │
│  │  • HTTP 206 / Range desteği         │  │
│  │  • HTML/JSON URL yeniden yazımı     │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
        │
        ▼
  DoH ile gerçek IP  →  Engelli CDN sunucusu
```

---

## Temel Teknik Buluşlar

### 1. DoH + Sabit IP Haritası

ISP'nin DNS çözümleyicisi bypass edilir:

```python
DNS_MAP = {
    "www.engelli-site.com": "93.x.x.x",  # gerçek IP, sabit kodlu
    ...
}

# Yedek: Cloudflare 1.1.1.1 / Google DoH
doh_url = f"https://1.1.1.1/dns-query?name={host}&type=A"
```

### 2. TLS SNI Enjeksiyonu

Python'ın `HTTPSConnection`'ı IP adresine bağlanırken varsayılan olarak SNI bilgisini göndermez. Bu, aşağıdaki şekilde düzeltilir:

```python
conn = http.client.HTTPSConnection(gercek_ip, 443, context=ssl_ctx)
conn._server_hostname = host  # TLS SNI zorlanır
```

### 3. Platform Bazlı Origin/Referer Başlık Yönetimi

CDN sunucuları, isteği doğrulayan `Origin` başlığını kontrol eder. Proxy her platform için doğru başlığı ayarlar:

```python
if "phncdn.com" in host:
    headers["Origin"]  = "https://www.pornhub.com"
    headers["Referer"] = "https://www.pornhub.com/"
```

### 4. Erişilebilir CDN Tespiti

Her video için birden fazla CDN aynası mevcuttur. Sunucu, commit etmeden önce her CDN'nin TCP erişilebilirliğini test eder (~0.8 sn zaman aşımı):

```python
def is_host_accessible(src_url):
    s = socket.socket(); s.settimeout(0.8)
    s.connect((gercek_ip, 443))  # engellenmiş mi?
```

### 5. İstemci Tarafı JS Intercept Enjeksiyonu

Proxy HTML sayfalarını geçirirken bir script enjekte eder; bu script `fetch()`, `XMLHttpRequest` ve `HTMLMediaElement.src` atamalarını yakalar ve tüm URL'leri otomatik olarak yerel proxy üzerinden geçirir:

```javascript
window.fetch = function(inp, init) {
  inp = toProxy(inp); // tarayıcıdan çıkmadan URL yeniden yazılır
  return _fetch.call(this, inp, init);
};
```

### 6. HTTP 206 Partial Content Akışı

Video dosyaları 64 KB'lık parçalar halinde akıtılır; `Range` başlıkları tam olarak iletilir. Bu, anında oynatma başlangıcı ve sorunsuz sarma/ileri sarma sağlar:

```python
while True:
    chunk = res.read(65536)
    if not chunk: break
    self.wfile.write(chunk)
    self.wfile.flush()
```

---

## İçerik Türü Seçiminin Gerekçesi

Etkili bir sansür bypass testi için hedef içeriğin iki koşulu sağlaması gerekir:

1. **Türkiye'deki ISP'ler tarafından güvenilir biçimde engellenmesi**
2. **Alt kaynak yoğunluğu** — script'ler, CDN medyası, HLS oynatma listeleri — bypass'ın teknik açıdan gerçekten anlamlı olması için

Yetişkin video platformları test hedefi olarak seçildi çünkü bu iki koşulu mükemmel biçimde karşılar. Türkiye'de vaka bazlı kararlarla değil, toptan düzenleyici yasaklarla sürekli ve kapsamlı biçimde engellenen siteler arasındadır. Üstelik teknik altyapıları da özellikle karmaşıktır: video dağıtımı bağımsız olarak barındırılan çok sayıda CDN alt alanı, kısa geçerlilik süresine sahip imzalı token URL'leri, oynatıcı kimlik doğrulama el sıkışmaları ve uyarlanabilir bit hızı (HLS/DASH) manifestleri içerir. Bu durum, başarılı bir uçtan uca bypass'ı teknik olarak son derece anlamlı ve kopyalanması güç kılmaktadır.

Yetişkin içerik konusu bu nedenle araştırma hedefinin ikincil bir sonucudur: içeriğin kendisi için değil, doğru biçimde atlatılması en zor sansür hedefi olduğu için seçilmiştir.

Rastgele "rulet" UX elementi ise ([theuselessweb.com](https://theuselessweb.com/) ilhamıyla) deneye oyunlaştırıcı, keşifsel bir karakter kazandırmak için eklendi. Şans/kader mekaniği hem yaratıcı bir çerçeveleme aracı hem de büyük bir içerik veri seti üzerinde örnekleme yapan hafif bir havuz yöneticisi olarak işlev gördü.

---

## Sunucuyu Çalıştırma

**Gereksinimler:** Python 3.10+, harici bağımlılık yok.

```bash
python server.py 5500
```

Herhangi bir tarayıcıda `http://localhost:5500` adresini açın. VPN yok, eklenti yok, ayar yok.

---

## Proje Durumu

Bu proje bir **araştırma deneyi**dir, üretim hizmeti değildir. Bulgular:

| Durum | Bulgu |
|-------|-------|
| ✅ | Saf Python + stdlib ile tam bypass mümkün |
| ✅ | Yerel HTML5 video oynatma (iframe yok, reklam yok) |
| ✅ | Sıfır istemci tarafı yapılandırması |
| ✅ | Chrome, Firefox, Edge, Safari'de çalışır |
| ⚠️ | CDN token'larının geçerlilik süresi ~2 saattir |
| ⚠️ | Bazı CDN düğümleri TCP düzeyinde coğrafi kısıtlamaya tabidir |

---

## Proje Dosyaları

| Dosya / Klasör | Açıklama |
|----------------|----------|
| `server.py` | Çok iş parçacıklı proxy + stream_info API |
| `index.html` | SPA ana sayfası (rulet arayüzü) |
| `script.js` | Havuz yönetimi, modal, zar animasyonu, i18n |
| `styles.css` | Neon LED efektleri, dark UI |
| `data/` | Havuz parçaları (chunk JSON'lar) |
| `scripts/harvester.py` | İçerik havuzu toplayıcısı |
| `cloudflare-worker/` | İsteğe bağlı CF Worker proxy alternatifi |
| `info/index.html` | Proje bilgi sayfası |

---

## Depo

Kaynak kodların tamamı (proxy sunucu, toplayıcı scriptler, SPA arayüzü, veri havuzları) kötüye kullanımı önlemek amacıyla **private bir depoda** tutulmaktadır. Bu README yalnızca teknik dokümantasyon ve portföy referansı amacıyla sağlanmaktadır.

**Geliştirici:** [SliferSoft](https://slifersoft.com) — Ersin Sandal  
**İletişim:** info@slifersoft.com
