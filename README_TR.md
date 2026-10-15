# LuckyLick — Türkçe Teknik Dokümantasyon

> **⚠️ Bu depo yalnızca dokümantasyon amacıyla kamuya açıktır.**
> Kaynak kodların tamamı private bir depoda tutulmaktadır.
> İletişim: [SliferSoft](mailto:info@slifersoft.com)

---

## Kısaca Ne Yapıyor?

LuckyLick, ISP düzeyinde internet sansürünü VPN kullanmadan aşmayı araştıran **deneysel bir çalışmadır**. theuselessweb.com mantığından esinlenen platformda, kullanıcı butona tıkladığında şans faktörüne göre önceden derlenmiş veri havuzlarından ve API servislerinden rastgele bir içerik seçilir. Test ortamı olarak yetişkin video platformlarının tercih edilme sebebi; Türkiye'de en katı idari engellere tabi olmaları ve çok katmanlı CDN altyapılarıyla tünelleme algoritmaları için en zorlu teknik ölçütü (benchmark) sunmalarıdır. Sistemin temel başarısı; normal koşullarda VPN gerektiren bu engelli platformların video akışlarını kullanıcı tarafında hiçbir VPN, tarayıcı eklentisi veya DNS ayarına ihtiyaç bırakmaksızın, yerel proxy ve DoH tünellemesiyle doğrudan tarayıcı üzerinden sorunsuz oynatabilmesidir.

---

## Proje Nedir?

**LuckyLick**, ISP (İnternet Servis Sağlayıcı) düzeyinde uygulanan DNS engellerini **VPN kullanmadan** aşmayı deneysel olarak kanıtlamak amacıyla geliştirilmiş bir web platformudur.

Projenin temel tezi şudur:

> *Standart bir tarayıcı üzerinden, herhangi bir VPN, tarayıcı eklentisi veya kullanıcı taraflı yapılandırma gerektirmeksizin, yalnızca sunucu tarafı DNS tünellemesi ve HTTPS proxy yönlendirmesi ile ISP tarafından engellenen içeriklere erişilebilir.*

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

## İçerik Türü ve Şans Mekaniği
 
Araştırma, Türkiye'deki ISP'ler tarafından güvenilir biçimde engellenen ve başarılı bir bypass'ın teknik açıdan zorlu olduğu bir test alanı gerektiriyordu. Yetişkin video platformları her iki kriteri de karşılamaktadır: Türkiye'de toptan düzenleyici yasaklara tabidir; çoklu CDN dağıtım yapıları, imzalı token URL'leri ve dinamik video akış manifestleri sunması nedeniyle tünelleme algoritmaları için zorlu bir kıstas sağlar.

Kullanıcı arayüzünde uygulanan şans mekaniği ([theuselessweb.com](https://theuselessweb.com/) ilhamıyla) ise bu deneyimi etkileşimli hale getirir: Butona tıklandığında veri havuzları ve API servisleri taranarak şans faktörüne göre rastgele bir içerik seçilir ve hiçbir VPN bağlantısına gerek kalmadan anında tünellenip oynatılır. İçerik türü yalnızca sansür direnci yüksek bir ölçüt olduğu için tercih edilmiştir.

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
