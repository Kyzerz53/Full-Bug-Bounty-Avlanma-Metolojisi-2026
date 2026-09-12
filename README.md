# 🕵️‍♂️ Tam Bug Bounty Avcılığı & Red Team Metodolojisi 2026 

> **vNez** tarafından hazırlanmıştır — Hedefleyen Red Team Operatörü ve Bug Bounty Avcısı. Bu repository; web uygulama güvenlik testleri için metodolojimi, notlarımı ve iş akışımı belgeler.


---

## Felsefe

Çoğu avcı Recon'ı bir kontrol listesi olarak ele alır. Ben ise onu bir istihbarat operasyonu olarak değerlendiririm.

Amaç her aracı çalıştırmak değil — saldırı yüzeyini herkesten daha hızlı ve derinlemesine haritalamaktır. Her aşama bir sonrakini besler. Her bulgu bir pivot noktasıdır.

Bu metodoloji sinyal-gürültü oranına göre sıralanmıştır: önce geniş ve pasif başlayın, ardından hiçbir exploit göndermeden önce agresif biçimde daraltın.

---

## 📌 İçindekiler

- [Aşama 1 — Pasif Subdomain Tespiti](#aşama-1--pasif-subdomain-tespiti)
- [Aşama 2 — Aktif Subdomain Tespiti & Kaba Kuvvet](#aşama-2--aktif-subdomain-tespiti--kaba-kuvvet)
- [Aşama 3 — Altyapı Haritalama (ASN / CIDR / IP'ler)](#aşama-3--altyapı-haritalama-asn--cidr--ipler)
- [Aşama 4 — WAF Bypass & Kaynak IP Tespiti](#aşama-4--waf-bypass--kaynak-ip-tespiti)
- [Aşama 5 — Birleştirme, Çözümleme & Canlı Host Tespiti](#aşama-5--birleştirme-çözümleme--canlı-host-tespiti)
- [Aşama 6 — Virtual Host Tespiti](#aşama-6--virtual-host-tespiti)
- [Aşama 7 — URL & Endpoint Keşfi](#aşama-7--url--endpoint-keşfi)
- [Aşama 8 — JavaScript Analizi & Gizli Bilgi Çıkarımı](#aşama-8--javascript-analizi--gizli-bilgi-çıkarımı)
- [Aşama 9 — Dizin & Hassas Dosya Keşfi](#aşama-9--dizin--hassas-dosya-keşfi)
- [Aşama 10 — GitHub & Kaynak Kod İstihbaratı](#aşama-10--github--kaynak-kod-istihbaratı)
- [Aşama 11 — Port Tarama & Servis Parmak İzi](#aşama-11--port-tarama--servis-parmak-i̇zi)
- [Aşama 12 — Otomatik Zafiyet Taraması](#aşama-12--otomatik-zafiyet-taraması)
- [Aşama 13 — Subdomain Takeover Tespiti](#aşama-13--subdomain-takeover-tespiti)
- [Wordlist Referansları](#wordlist-referansları)
- [OSINT Platform Referansları](#osint-platform-referansları)

---

## ⚙️ Gereksinimler & Ortam Kurulumu

Aşağıdaki komutları çalıştırmadan önce ortam değişkenlerinizin doğru şekilde ayarlandığından emin olun:

```bash
export DNS_WORDLIST="/path/to/subdomains-wordlist.txt"
export WEB_WORDLIST="/path/to/directories-wordlist.txt"
export GITHUB_TOKEN="your_github_personal_access_token"
export PDCP_API_KEY="your_projectdiscovery_chaos_key"
```

---

## Aşama 1 — Pasif Subdomain Tespiti

Hedefe hiçbir trafik gönderilmez. Tamamen kamuya açık kaynaklardan istihbarat toplanır.

### Temel Araçlar

```bash
# subfinder — hızlı, API destekli pasif subdomain tespiti
subfinder -d target.com -all -recursive -o subs_subfinder.txt

# assetfinder — ilgili alan adlarını ve subdomainleri bulur
echo target.com | assetfinder -subs-only > subs_assetfinder.txt

# amass — derin OSINT motoru
amass enum -d target.com -o subs_amass.txt
amass enum -d target.com -brute -w $DNS_WORDLIST -o subs_amass_brute.txt

# findomain — çok kaynaklı pasif recon
findomain -t target.com -u subs_findomain.txt

# chaos — ProjectDiscovery'nin derlenmiş veri seti (API anahtarı gerektirir)
export PDCP_API_KEY=YOUR_KEY
chaos -d target.com -o subs_chaos.txt

# github-subdomains — geliştirici kodlarından gizli endpointleri çıkarır
github-subdomains -d target.com -t $GITHUB_TOKEN -o subs_github.txt
```

### Sertifika Şeffaflığı (SSL)

En az değer verilen pasif kaynaklardan biri — her SSL sertifikası kamuya açık kayıttır.

```bash
curl -s "https://crt.sh/?q=%25.target.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | tr ',' '\n' \
  | grep -oE "[A-Za-z0-9._-]+\.target\.com" \
  | sort -u > subs_crt.txt
```

### Toplu Çalıştırma (Her Şeyi Aynı Anda)

```bash
# subenum — birden fazla aracı tek seferde çalıştırır
./subenum.sh -d target.com \
  -u wayback,crt,abuseipdb,Findomain,Subfinder,Amass,Assetfinder \
  -o subs_subenum.txt

# Birden fazla hedef için
./subenum.sh -l targets.txt \
  -u wayback,crt,abuseipdb,Findomain,Subfinder,Amass,Assetfinder \
  -o subs_subenum.txt
```

---

## Aşama 2 — Aktif Subdomain Tespiti & Kaba Kuvvet

Bu aşamada hedefe dokunmaya başlıyoruz — domain'leri çözümlüyor ve DNS'i kaba kuvvetle tarıyoruz.

### DNS Kaba Kuvvet

```bash
# puredns — genel resolver'larla yüksek hızlı DNS çözümleme
sudo wget -q https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt
puredns bruteforce $DNS_WORDLIST target.com -r resolvers.txt -o subs_puredns.txt

# dnsx — wildcard filtrelemeli hafif DNS resolver
dnsx -silent -d target.com -w $DNS_WORDLIST -o subs_dnsx.txt

# dnscan — Python tabanlı, daha yavaş ve gizli taramalar için
python dnscan.py -d target.com -w $DNS_WORDLIST -t 300 | tee subs_dnscan.txt
```

### Subdomain Permütasyonu & Tahmini

Zaten keşfedilmiş subdomainlerden akıllı mutasyonlar üretin. Gizli iç sunucular genellikle öngörülebilir isimlendirme kalıplarını takip eder.

```bash
# gotator — bilinen subdomainlerden permütasyonlar üretir
gotator -sub subs_all.txt -perm permutations.txt -depth 1 -numbers 3 -md | sort -u > subs_permuted.txt

# puredns ile çözümle
puredns resolve subs_permuted.txt -r resolvers.txt -o subs_permuted_alive.txt
```

### ffuf ile Subdomain Fuzzing

```bash
# Standart subdomain fuzzing
ffuf -u https://FUZZ.target.com -w $DNS_WORDLIST -mc 200,301,302,403

# Tire içeren kalıplar (dev-target.com, api-target.com)
ffuf -u https://FUZZ-target.com -w $DNS_WORDLIST -mc 200,301,302,403

# Önek kalıpları (www-old, www-beta, www-test)
ffuf -u https://FUZZwww.target.com -w $DNS_WORDLIST -mc 200,301,302,403
```

### Canlı Sertifika İzleme

Geliştirici onları güvenli hale getirmeden önce, yeni subdomain'leri tam yayımlandıkları anda yakalayın.

```bash
# gungnir — gerçek zamanlı sertifika şeffaflığı izleme
gungnir -d target.com
```

---

## Aşama 3 — Altyapı Haritalama (ASN / CIDR / IP'ler)

Çoğu avcı subdomainlerde durur. Burada daha derine iniyoruz — kuruluşun tüm IP altyapısını haritalıyoruz.

### Adım 1 — ASN'yi Bul

```bash
# asnmap — domain'i ASN'ye çözer
asnmap -d target.com

# Manuel whois yaklaşımı
dig target.com +short  # önce bir IP al
whois <IP> | grep -i "origin\|as\|route"

# spk — şirket adına göre tüm ASN'leri bulur
spk -json -s "Tesla"
```

Web alternatifleri:
- [https://bgp.he.net](https://bgp.he.net) — şirket adına göre arama
- [https://bgp.tools](https://bgp.tools) — temiz arayüz
- [https://asnlookup.com](https://asnlookup.com) — org adı, ASN veya CIDR'e göre arama

### Adım 2 — ASN'den IP Aralıklarına (CIDR)

```bash
# asnmap — doğrudan CIDR çıkarımı
asnmap -a AS33905 -silent

# whois tabanlı yaklaşım — yönlendirme veritabanlarından da çıkarır
whois -h whois.radb.net -- '-i origin AS33905' \
  | grep -Eo "([0-9.]+){4}/[0-9]+" \
  | sort -u > cidr_ranges.txt

# Güçlü hamle: CIDR aralığındaki her IP için PTR kayıtlarını çözümle
whois -h whois.radb.net -- '-i origin AS20461' \
  | grep -Eo "([0-9.]+){4}/[0-9]+" \
  | mapcidr -silent \
  | dnsx -ptr -resp-only -retry 3 -silent > ptr_domains.txt

# metabigor — birden fazla kaynaktan bir kuruluşa kayıtlı tüm IP'leri çeker
echo "Tesla" | metabigor net --org
echo "ASN33905" | metabigor net --asn
```

### Adım 3 — CIDR'dan Tek IP'lere

```bash
# mapcidr — CIDR'ı temiz bir şekilde tek tek IP'lere böler
echo 10.10.10.0/24 | mapcidr

# prips — bir aralık için tam IP listesi üretir
prips 2.18.48.0/21 > ips_asn.txt
```

### Adım 4 — IP Aralıklarında Tersine DNS

```bash
# dnsx PTR — IP bloklarından hostname'leri çözümler
echo 66.211.170.0/23 | dnsx -silent -resp-only -ptr

# hakrevdns — geniş ölçekte tersine DNS
hakrevdns -d target.com -R resolvers.txt

# resolveDomains — subdomainlerin canlı IP'lere çözümlenip çözümlenmediğini kontrol eder
resolveDomains -d all_subs.txt > resolved.txt
awk '{print $3}' resolved.txt | sort -u > unique_ips.txt
```

### TLD Genişletme

`target.com`'a sahip bir şirket, `target.io`, `target.net`, `target.xyz`'yi çoğunlukla ihmal eder — daha zayıf savunmalarla tamamen ayrı bir saldırı yüzeyi.

```bash
# tldbrute — tüm kayıtlı TLD varyantlarını keşfeder
tldbrute -d target.com

# Tam IANA TLD listesi yaklaşımı
wget -q https://data.iana.org/TLD/tlds-alpha-by-domain.txt
cat tlds-alpha-by-domain.txt \
  | tr '[:upper:]' '[:lower:]' \
  | while read t; do echo "target.$t"; done \
  | httpx -mc 200 > tlds_alive.txt

# Aynı mantığı subdomain listesinde uygula
cat all_subs.txt | while read sub; do
  cat tlds-alpha-by-domain.txt \
    | tr '[:upper:]' '[:lower:]' \
    | sed "s/^/$sub./"
done | dnsx -silent > subs_tld_expanded.txt
```

---

## Aşama 4 — WAF Bypass & Kaynak IP Tespiti

Cloudflare ve benzeri WAF'lar, bug bounty hedeflerinin %70'inden fazlasını korur. Kaynak IP'yi bulmak ham sunucuyu açığa çıkarır — güvenlik duvarı yok, hız limiti yok.

### Yöntem 1 — Favicon Hash'i (En Güvenilir)

Şirketler aynı favicon'u tüm altyapılarında kullanır. Hash, Shodan'da arayabileceğiniz bir parmak izidir.

```bash
# favUp — favicon hash'i + Shodan üzerinden kaynak IP'yi bulur
python3 favUp.py -ff favicon.ico --shodan-cli
python3 favUp.py --web target-behind-cloudflare.com -sc

# favirecon — hafif favicon recon
favirecon -u https://target.com/ -v

# FavFreak — subdomain listenizde benzersiz favicon hash'lerini tespit eder
# Farklı hash'e sahip her şey = farklı altyapı = araştırmaya değer
cat subs.txt | python3 favfreak.py
```

Manuel yaklaşım:
1. [https://favicons.teamtailor-cdn.com/](https://favicons.teamtailor-cdn.com/) adresine gidin → hedef URL'yi yapıştırın → favicon'u alın
2. [https://favicon-hash.kmsec.uk/](https://favicon-hash.kmsec.uk/) adresine gidin → favicon URL'sini yapıştırın → hash'i alın
3. Shodan'da arayın: `http.favicon.hash:-382492124`

Bir IP hedefin favicon'unu döndürüyor ama Cloudflare IP'si değilse → o sizin kaynak sunucunuzdur.

### Yöntem 2 — Tarihsel DNS Kayıtları

```bash
# originiphunter — tarihsel IP'ler için birden fazla kaynağı sorgular
echo "target.com" | originiphunter
cat domains.txt | originiphunter
```

Tarihsel IP'ler için OSINT kaynakları:
- [https://securitytrails.com](https://securitytrails.com) — DNS geçmişi
- [https://viewdns.info/reverseip/](https://viewdns.info/reverseip/) — tersine IP araması
- [https://search.censys.io](https://search.censys.io) — `parsed.names: target.com` arama
- [https://www.shodan.io](https://www.shodan.io) — `ssl.cert.subject.cn:target.com` arama
- [https://netlas.io](https://netlas.io) — derin altyapı araması

### Yöntem 3 — Google Analytics ID'si

Tek bir Analytics ID'si tüm kurumsal aileyi açığa çıkarabilir — yan kuruluşlar, satın alınan şirketler, uluslararası domain'ler.

```bash
# ID'yi ve bağlantılı domain'leri keşfet
cat subdomains.txt | analyticsrelationships

# Manuel arama
# https://builtwith.com/relationships/target.com
# https://api.hackertarget.com/analyticslookup/?q=target.com
# https://api.hackertarget.com/analyticslookup/?q=UA-16316580
```

---

## Aşama 5 — Birleştirme, Çözümleme & Canlı Host Tespiti

Her aşama çıktı dosyaları üretir. Burada konsolide ediyor, tekilleştiriyor ve gerçekten canlı olanları tespit ediyoruz.

### Tüm Subdomain Kaynaklarını Birleştir

```bash
cat subs_*.txt ptr_domains.txt subs_permuted_alive.txt \
  | anew \
  | tee all_subs.txt

wc -l all_subs.txt
```

### httpx ile Canlı Host Tespiti

```bash
# Temel — sadece canlı hostları al
cat all_subs.txt | httpx -silent -o alive_subs.txt

# Zenginleştirilmiş — durum kodları, başlıklar, web sunucusu, yanıt boyutu
cat all_subs.txt | httpx \
  -status-code -content-length -web-server -title \
  -follow-redirects -o alive_enriched.txt

# Duruma göre filtrele — sadece 200'ler
cat all_subs.txt | httpx -status-code -follow-redirects -match-code 200

# Gürültüyü filtrele — 400'leri hariç tut
cat all_subs.txt | httpx -status-code -follow-redirects -filter-code 400

# Yanıt kodu mantığı:
# 404 → waybackurls + fuzzing dene
# 403 → bypass teknikleri dene
```

### Görsel Recon — Her Şeyin Ekran Görüntüsünü Al

500 subdomain'i elle açamazsınız. Aracın tarayıp ekran görüntüsü almasına izin verin — sonuçlara göz atın ve hedefleri seçin.

```bash
# gowitness — tüm canlı hostların ekran görüntüsünü alır
gowitness file -f alive_subs.txt -P ./screenshots/ --no-http

# eyewitness — raporlama ile birlikte
python3 EyeWitness.py -f alive_subs.txt --web -d ./eyewitness_output
```

### CMS Tespiti

```bash
# whatweb — CMS ve teknolojilerin parmak izini çıkarır
whatweb -i alive_subs.txt -a 3 -t 50 --log-brief=cms_results.txt

# wappalyzer CLI — teknoloji yığını parmak izi
wappalyzer https://target.com
```

---

## Aşama 6 — Virtual Host Tespiti

Bazı hostlar yalnızca doğru Host başlığıyla erişildiğinde yanıt verir — normal tarayıcılara görünmezler. VHOST tespiti, IP'lere eşlenmiş iç servisleri açığa çıkarır.

```bash
# Keşfedilen IP'lerde iç virtual hostları fuzz'la
ffuf -u http://<IP> \
  -w $DNS_WORDLIST \
  -H "Host: FUZZ.target.com" \
  -fs 0 \
  -mc 200,301,302,401,403

# HTTPS varyantı
ffuf -u https://target.com \
  -w $DNS_WORDLIST \
  -H "Host: FUZZ.target.com" \
  -mc 200,301,302,401,403
```

### VHOST Hedeflerine Erişim

```bash
# Yöntem 1 — Host başlığıyla doğrudan curl
curl -H "Host: dev.target.com" http://<IP>

# Yöntem 2 — /etc/hosts enjeksiyonu (tarayıcı erişimi için)
sudo nano /etc/hosts
# Ekle: 23.7.244.99  dev.target.com internal.target.com admin.target.com

# Sonra tarayıcıda aç: http://dev.target.com
# OWASP Top 10 dene: IDOR, Auth, Mantık, API kötüye kullanımı
```

### SSL Sertifikaları Üzerinden IP'lerden Hostname'lere

```bash
# hosthunter — IP listesindeki SSL sertifikalarından hostname'leri çıkarır
python3 hosthunter.py ips.txt

# httpx — hangi IP'lerin web içeriği sunduğunu kontrol eder
cat ips.txt | httpx -ports 80,443,8080,8000,8888 -status-code -title
```

---

## Aşama 7 — URL & Endpoint Keşfi

Buradaki amaç, uygulamanın geçmişte ve günümüzde açığa çıkardığı her URL'yi kapsayan kapsamlı bir harita oluşturmaktır.

### Tarihsel URL Toplama

```bash
# waybackurls
cat alive_subs.txt | waybackurls > urls_wayback.txt

# waymore — daha akıllı, tarihe ve sonuç sayısına göre filtreler
waymore -i alive_subs.txt -mode U -l 1000 -from 2021 -oU urls_waymore.txt

# gau — birden fazla kaynaktan toplar
cat alive_subs.txt | gau --threads 200 > urls_gau.txt

# gauplus — iyileştirmelerle birlikte gau
gauplus -t 200 -random-agent < alive_subs.txt > urls_gauplus.txt
```

### Aktif Crawling

```bash
# katana — modern JS destekli crawler (genel olarak en iyi)
katana -u alive_subs.txt \
  -jc -kf all -d 5 \
  -headless -fx -aff \
  -fs rdn -f url -silent > urls_katana.txt

# gospider
gospider -S alive_subs.txt -t 20 -d 3 --js --sitemap --robots -o ./gospider_output/
gospider -S alive_subs.txt \
  | sed -n 's/.*\(https:\/\/[^ ]*\)]*.*/\1/p' >> urls_gospider.txt

# hakrawler
cat alive_subs.txt | hakrawler -subs -u -insecure > urls_hakrawler.txt
```

### Parametre Keşfi

```bash
# paramspider — Wayback verilerinden parametreleri bulur
paramspider -d target.com -o urls_params.txt

# x8 — HTTP yanıt karşılaştırması üzerinden gizli parametre keşfi
x8 -u "https://target.com/endpoint" -o urls_x8.txt
```

### Tüm URL'leri Birleştir

```bash
cat urls_wayback.txt urls_waymore.txt urls_gau.txt urls_gauplus.txt \
    urls_katana.txt urls_gospider.txt urls_hakrawler.txt urls_params.txt \
  | anew \
  | tee all_urls.txt

wc -l all_urls.txt
```

### Yüksek Değerli URL Kategorilerini Çıkar

```bash
# JavaScript dosyaları
cat all_urls.txt | grep -iE '\.js(\?|$)' | grep -iv '\.json' | sort -u > js_urls.txt

# API endpointleri
cat all_urls.txt | grep -Ei '\.(json|xml|graphql|gql)(\?|$)' > api_urls.txt

# Backend dosyaları (PHP, ASP, JSP)
cat all_urls.txt | grep -Ei '\.(php|asp|aspx|jsp|cfm|cgi)(\?|$)' > backend_urls.txt

# Giriş & kimlik doğrulama akışları
cat all_urls.txt | grep -Ei "login|signin|auth|oauth|reset|password" > auth_urls.txt

# Dosya yükleme endpointleri
cat all_urls.txt | grep -Ei "upload|file|download|image|media" > upload_urls.txt

# Yönetim panelleri
cat all_urls.txt | grep -Ei "admin|dashboard|internal|manage" > admin_urls.txt

# Hassas dosya uzantıları
cat all_urls.txt | grep -Ei '\.(env|bak|config|sql|log)(\?|$)' > sensitive_urls.txt

# IDOR adayları (sayısal ID içeren URL'ler)
cat all_urls.txt | grep -Ei '[0-9]{2,}' > idor_candidates.txt

# Açık yönlendirme adayları
cat all_urls.txt | grep -Ei "redirect|callback|goto|return|dest=|r=|u=|url=" > redirect_urls.txt

# Bulut kimlik bilgisi ifşası
cat all_urls.txt | grep -Ei "aws|s3|bucket|gcp|azure|token|apikey|secret" > cloud_urls.txt

# İlginç her şey tek seferde
cat all_urls.txt | urinteresting
```

### Parametreleri Çıkar & Filtrele

```bash
# Parametreli tüm URL'leri çıkar
cat all_urls.txt | grep "=" | anew params.txt

# Fuzzing için hazırla
cat all_urls.txt | grep "=" | qsreplace "FUZZ" | anew param_fuzz.txt

# Parametre adına göre tekilleştir (değer gürültüsünü kaldır)
cat all_urls.txt | grep '=' | sed 's/=[^&]*/=/g' | sort -u > params_clean.txt

# arjun ile gizli parametreleri keşfet
arjun -i backend_urls.txt -o arjun_params.json
arjun -u https://target.com/endpoint -m POST
```

### Canlı URL'leri Bul

```bash
cat all_urls.txt | httpx -status-code -content-length -silent > live_urls.txt
```

---

## Aşama 8 — JavaScript Analizi & Gizli Bilgi Çıkarımı

JavaScript dosyaları altın madenidir. Hardcoded API anahtarları, iç endpointler, kimlik doğrulama mantığı ve bazen tam backend altyapı haritaları içerirler.

### JS'den Gizli Bilgileri Çıkar

```bash
# subjs — URL listesinden JS dosyalarını çeker
cat all_urls.txt | subjs | tee js_files.txt

# mantra — JS için regex tabanlı gizli tarayıcı
cat js_urls.txt | mantra

# jsecret — JS dosyalarında hassas kalıpları bulur
cat js_urls.txt | jsecret

# jsleak — eşzamanlı JS gizli tarayıcı
cat js_urls.txt | xargs -P 20 -I {} \
  jsleak -s -l -k -e {} >> jsleak_output.txt

# Herhangi bir JS dosyasında hızlı regex taraması
grep -E "api[_-]?key|token|secret|password|bearer|client_id" target.js
```

### Kaynak Haritası Sömürüsü

Kaynak haritaları production'da açık bırakıldığında, tam derlenmiş öncesi kaynak kodunu kurtarabilirsiniz.

```bash
# Wayback üzerinden .map dosyalarını bul
curl -s "https://web.archive.org/cdx/search/cdx?url=*.target.com/*&collapse=urlkey&output=text&fl=original&filter=original:.*.js.map$"

# İndir ve çıkar
wget https://target.com/static/app.js.map
node -e "
const map = require('./app.js.map');
map.sources.forEach((src, i) => {
  require('fs').writeFileSync(src.split('/').pop(), map.sourcesContent[i]);
});
"
```

### TruffleHog — Derin Git Geçmişi Taraması

```bash
# Kamuya açık GitHub repo'yu tara (dakikalar önce silinmiş sırları bile bulur)
trufflehog git https://github.com/target/repo --results=verified

# Tüm organizasyonu tara
trufflehog github --org=target \
  --token=$GITHUB_TOKEN \
  --only-verified \
  --threads=20 \
  --json > trufflehog_target.json

# Yerel dosya sistemini tara
trufflehog filesystem ./js_files/ --json > trufflehog_local.json
```

### Lazyegg

```bash
# JS dosyalarından linkleri, API'leri, IP'leri tarar ve çıkarır
python lazyegg.py https://target.com
python lazyegg.py https://target.com/js/auth.js

# Derin kapsam için waybackurls ile birleştir
waybackurls target.com \
  | grep '\.js$' \
  | awk -F '?' '{print $1}' \
  | sort -u \
  | xargs -I{} bash -c 'python lazyegg.py "{}" --js_urls --domains --ips' \
  > lazyegg_output.txt
```

---

## Aşama 9 — Dizin & Hassas Dosya Keşfi

Tamamen yamalı uygulamalar bile unutulmuş dosyalar, yedek arşivler ve yanlış yapılandırılmış dizinler aracılığıyla hassas veri sızdırır.

### dirsearch

```bash
# Tam özellikli dizin taraması
dirsearch -u https://target.com \
  -e php,asp,aspx,jsp,json,xml,txt,log,ini,cfg,conf,bak,old,backup,zip,tar,gz,rar,sql,swp,db \
  -t 80 -r -R 3 \
  --deep-recursive \
  --random-agent \
  --full-url \
  -i 200,204,301,302,307,308,401,403 \
  -x 404,500,502,503,504 \
  -o dirsearch_results.txt

# Tüm canlı subdomainleri aynı anda tara
dirsearch -l alive_subs.txt \
  --full-url \
  -e php,env,json,yaml,bak,zip,sql,conf \
  -o dirsearch_all.txt
```

### ffuf

```bash
# Standart dizin fuzzing
ffuf -u https://target.com/FUZZ \
  -w $WEB_WORDLIST \
  -t 80 \
  -e .html,.php,.asp,.aspx,.js,.json,.xml,.config,.bak,.old,.zip,.rar \
  -mc 200,204,301,302,307,401,403 \
  -of json -o ffuf_dirs.json

# Özyinelemeli fuzzing
ffuf -u https://target.com/FUZZ \
  -w $WEB_WORDLIST \
  -recursion -recursion-depth 3 \
  -mc 200,204,301,302,307,401,403

# 403 Bypass — alternatif başlıkları dene
ffuf -u https://target.com/admin \
  -w https://github.com/Karanxa/Bug-Bounty-Wordlists/raw/main/403_header_payloads.txt \
  -H "FUZZ" \
  -mc 200,301,302

# IP başlığı sahteciliği üzerinden WAF bypass
ffuf -u https://target.com/FUZZ \
  -w $WEB_WORDLIST \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "X-Forwarded-Host: 127.0.0.1" \
  -H "X-Custom-IP-Authorization: 127.0.0.1" \
  -H "X-Original-URL: /FUZZ" \
  -mc 200,301,302,307,401

# Akıllı otomatik kalibrasyon (wildcard 200 yanıtlarını engeller)
ffuf -u https://target.com/FUZZ \
  -w $WEB_WORDLIST \
  -mc all -fc 404 -ac -sf -s
```

> **Önemli İpucu:** Bir yol son karakter olarak `/` ile yanıt verdiğinde, arkasında keşfedilecek daha fazlası var demektir. Her zaman özyinelemeli tarayın.

### feroxbuster

```bash
feroxbuster -u https://target.com \
  -w $WEB_WORDLIST \
  -t 300 -k -d 3 \
  -x php,html,json,js,log,txt,bak,old,zip,tar,gz
```

### Wayback Hassas Dosya Madenciliği

```bash
# Kesin hassas dosya kalıbı — her hedefte çalıştırın
waybackurls https://target.com \
  | grep -E "\.(xls|xlsx|csv|sql|db|bak|backup|old|tar\.gz|tgz|zip|7z|rar|pdf|pem|key|crt|env|json|yml|yaml|conf|config|git|htpasswd|log|dump|DS_Store)" \
  | sort -u > sensitive_wayback.txt
```

### Yedek Dosya Denetleyicisi

```bash
# bfac — kazara açığa çıkan yedek dosyaları bulur
bfac --url https://target.com \
  --detection-technique all \
  --level 3 \
  --exclude-status-codes 404,500
```

### GitHub Endpointleri — Sızdırılmış API Yollarını Bul

```bash
# github-endpoints — geliştirici repo'larını iç yollar için tarar
github-endpoints -q -k -d target.com -t $GITHUB_TOKEN

# Örnek bulgu: var api_url = "https://dev-test.target.com/api/v1/debug"
```

### Robots.txt Geçmişi

```bash
# roboxtractor — gizli yolları bulmak için tarihsel robots.txt'yi çeker
cat alive_subs.txt | roboxtractor -m 1 -wb
```

### 404'ten Altına — Tarihsel Sayfa Kurtarma

```bash
# Adım 1: Tarihsel URL'leri topla
waybackurls https://target.com | grep "webstat" > old_pages.txt

# Adım 2: Wayback Machine anlık görüntülerini kontrol et
# https://web.archive.org/web/*/https://target.com/webstat/*

# Adım 3: O yol etrafındaki bağlantılı kaynaklar için yeniden tara
gospider -s https://target.com -a -r \
  | grep -oE 'https?://[^[:space:]"]+' \
  | grep "/webstat/"
```

### Google Sheets Sızıntı Avı

```bash
# Kuruluşlar kazara iç sheet'leri açığa çıkarır
site:*.target.com intext:"docs.google.com/spreadsheets"
site:docs.google.com/spreadsheets "target.com"
site:docs.google.com/spreadsheets "@target.com"
site:docs.google.com/spreadsheets "password" "target.com"
```

---

## Aşama 10 — GitHub & Kaynak Kod İstihbaratı

Geliştiriciler sürekli olarak sırları kazara push'lar. Bu aşama, kamuya açık depolarda API anahtarları, tokenlar, şifreler ve iç altyapı detaylarını avlar.

### GitDorker — Hedefli GitHub Araması

```bash
python3 GitDorker.py \
  -tf $GITHUB_TOKEN \
  -q target.com \
  -d dorks/medium_dorks.txt \
  -o gitdorker_results.txt

# Metadata'da bulunan çalışan adlarına göre de ara
python3 GitDorker.py -tf $GITHUB_TOKEN -q "john.doe@target.com" -d dorks/medium_dorks.txt
```

Dork kaynakları:
- [https://github.com/Proviesec/github-dorks](https://github.com/Proviesec/github-dorks)
- [https://mr-koanti.github.io/github.html](https://mr-koanti.github.io/github.html)

### TruffleHog — Doğrulanmış Gizli Tespiti

```bash
# Org'u tara — silinmiş commit'lerdeki sırları bile bulur
trufflehog github \
  --org=target \
  --token=$GITHUB_TOKEN \
  --only-verified \
  --threads=20 \
  --json > trufflehog_target.json

# Docker varyantı
docker run --rm -it trufflesecurity/trufflehog:latest \
  github --only-verified --org=target
```

### shhgit — Gerçek Zamanlı GitHub İzleme

GitHub'a şu anda push edilen sırları izleyin — geliştirici silemeden önce.

```bash
# Yaygın gizli kalıplar için küresel izleme
shhgit --search-query \
  'path:*.env OR "DB_PASSWORD=" OR "AWS_ACCESS_KEY_ID=" OR "-----BEGIN RSA PRIVATE KEY-----"'

# Gerçek zamanlı olarak belirli bir hedef org'u izle
shhgit --search-query \
  'target.com (path:*.env OR "DB_PASSWORD=" OR "api_key=")'
```

### git-wild-hunt — Dosya Uzantısı Araması

```bash
# Hedefin repo'larında belirli dosya türlerini bul
python git-wild-hunt.py -s "org:Target extension:json filename:creds language:JSON"
python git-wild-hunt.py -s "org:Target extension:sql filename:backup"
python3 git-wild-hunt.py -s "target.com gitlab_token"
```

### GitLab — Özel Altyapı

Büyük şirketler GitHub yerine iç GitLab örneklerini kullanır — gerçek sırlar burada yaşar.

```bash
# GitLab örneğini keşfet
# Dene: gitlab.target.com | git.target.com | code.target.com

# Bulunan bir tokenla kimlik doğrula
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.target.com/api/v4/user"

# Erişilebilir projeleri listele
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.target.com/api/v4/projects?membership=true&simple=true"

# Erişilebilir tüm repo'larda sırları derinlemesine tara
gitleaks detect \
  --source https://gitlab.target.com \
  --access-token $GITLAB_TOKEN -v
```

### Metadata Çıkarımı

```bash
# metafinder — kamuya açık belgeleri indirir ve metadata'yı çıkarır
# Açığa çıkarır: kullanıcı adları, yazılım sürümleri, iç dosya yolları, e-posta kalıpları
metafinder -d "target.com" -l 10 -go -bi -ba -o metadata.txt

# Subdomainlerde de dene
metafinder -d "dev.target.com" -l 10 -go -bi -ba -o metadata_dev.txt
```

> **Bu neden önemli:** Bir PDF'nin metadata'sı `C:\Users\john.doe\Projects\InternalAPI\` gibi bir şey ortaya çıkarabilir — bu, daha fazla recon için kullanabileceğiniz bir kullanıcı adı ve proje adıdır.

---

## Aşama 11 — Port Tarama & Servis Parmak İzi

Web portları beklenir. Standart dışı portlar ise yanlış yapılandırılmış servislerin, yönetim panellerinin ve açığa çıkmış API'lerin gizlendiği yerlerdir.

### Adım 1 — Hızlı Port Taraması (İlk 1000)

```bash
# naabu — son derece hızlı Go tabanlı port tarayıcı
naabu -l unique_ips.txt \
  -exclude-ports 80,443 \
  -rate 2000 \
  -o open_ports.txt

# Doğrudan subdomain listesine karşı
naabu -list alive_subs.txt -p - -rate 1000 -c 50 -o ports_full.txt
```

### Adım 2 — Tam Port Taraması (65535)

```bash
naabu -l unique_ips.txt \
  -p - \
  -exclude-ports 80,443,8080,8000,8888 \
  -o ports_fullscan.txt
```

### Adım 3 — Servis Sürüm Tespiti

```bash
# nmap — keşfedilen açık portlarda sürüm + varsayılan script taraması
nmap -sV -sC -iL open_ports.txt -oN nmap_versions.txt

# naabu + nmap'i tek pipeline'da birleştir
naabu -list unique_ips.txt -p - -rate 1000 -c 50 \
  -nmap-cli 'nmap -sV -sC' \
  -o ports_with_services.txt
```

### Adım 4 — Servislerde Zafiyet Taraması

```bash
# nuclei — ağ katmanı zafiyetlerini tara
nuclei -l unique_ips.txt \
  -t nuclei-templates/network/ \
  -H "X-Forwarded-For: 127.0.0.1" \
  -mhe 4 -rl 30 -es info

# Nessus (GUI) — mevcut ise tam CIDR aralığını ver
# Besleme: 2.18.48.0/21
# CVE'ler, yanlış yapılandırmalar ve kimlik bilgisi sorunları için taratın
```

---

## Aşama 12 — Otomatik Zafiyet Taraması

### nuclei — CVE & Yanlış Yapılandırma Tespiti

```bash
# Bilinen CVE'ler + açıklar için tüm canlı subdomainleri tara
nuclei -l alive_subs.txt \
  -t nuclei-templates/http/ \
  -severity critical,high,medium \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "X-Forwarded: 127.0.0.1" \
  -mhe 4 -rl 30 -es info \
  -o nuclei_results.txt

# Özellikle ifşa şablonlarını hedefle
nuclei -l alive_subs.txt \
  -t nuclei-templates/http/exposures/ \
  -o nuclei_exposures.txt

# Tor proxy — her istekte IP rotasyonu, tüm hız limitlerini aşar
nuclei -u https://target.com \
  -p socks5://127.0.0.1:9050
```

### SQL Enjeksiyonu

```bash
# Parametreli URL'lerde sqlmap
sqlmap -u "https://target.com/page.php?id=1" \
  --dbs --banner --batch --random-agent

# Kaydedilmiş istek dosyasından (Burp dışa aktarımı)
sqlmap -r request.txt --dbs --banner --batch --random-agent
```

### Subdomain Takeover Kontrolü

Aşama 13'e bakın.

### API Anahtarı Doğrulama Pipeline'ı

```bash
# JS/JSON/config dosyalarında API anahtarlarını bul, ardından doğrula
echo "https://target.com" \
  | gau \
  | grep -E '\.js$|\.json$|\.xml$|\.env$|\.config$' \
  | httpx -silent -mc 200 \
  | parallel -j 50 "curl -s {} \
    | grep -oP '(?:api[_-]?key|secret|token)[\"'\''']?\s*[:=]\s*[\"'\''']?([A-Za-z0-9_\-]{20,})' \
    | tee -a api_keys.txt"
```

---

## Aşama 13 — Subdomain Takeover Tespiti

Kaydı silinmiş bir üçüncü taraf servise (Heroku, S3, Zendesk vb.) işaret eden bir subdomain, herhangi biri tarafından sahiplenilebilir. Bulunduğunda binlerce dolar değerindedir.

```bash
# subzy — hızlı takeover tarayıcı
subzy run --targets all_subs.txt --hide_fails --vuln \
  | grep -v -E "Akamai|available|\-"

# dnsx — tüm subdomainler için CNAME kayıtlarını al
dnsx -retry 3 -a -aaaa -cname -ns -ptr -mx -soa \
  -resp -silent \
  -l all_subs.txt \
  | tee dns_records.txt
```

### Dikkat Edilmesi Gerekenler

```
api.target.com [CNAME] clusters.heroku.com     ← kayıtlı olup olmadığını kontrol et
help.target.com [CNAME] target.zendesk.com     ← müsait olup olmadığını kontrol et
cdn.target.com [CNAME] storage.s3.amazonaws.com ← bucket adını kontrol et
```

### Takeover Adaylarını Doğrula

```bash
# Şüpheli subdomain'i derinlemesine incele
dig help.target.com CNAME

# Google Dig Aracını kontrol et
# https://toolbox.googleapps.com/apps/dig/#TXT/

# TXT kaydı yoksa → subdomain ele geçirilmeye hazır demektir
```

### S3 Takeover Pipeline'ı

```bash
subfinder -d target.com -silent \
  | dnsx -silent -cname \
  | grep "s3.amazonaws" \
  | httpx -mc 404 \
  | while read sub; do
      aws s3 mb "s3://${sub#https://}" && echo "CLAIMED: $sub"
    done
```

---

## Wordlist Referansları

| Amaç | Yol |
|------|-----|
| DNS — İlk 100K | `/usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt` |
| DNS — Jhaddix | `/usr/share/wordlists/seclists/Discovery/DNS/dns-Jhaddix.txt` |
| DNS — İlk 1M | `/usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt` |
| DNS — CommonSpeak2 | `/usr/share/wordlists/commonspeak2-wordlists-master/subdomains/subdomains.txt` |
| DNS — En İyi | `sudo wget https://wordlists-cdn.assetnote.io/data/manual/best-dns-wordlist.txt` |
| Web — Büyük Dizinler | `/usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt` |
| Web — Büyük Dosyalar | `/usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt` |
| Web — Yaygın | `/usr/share/wordlists/seclists/Discovery/Web-Content/common.txt` |
| Web — Orta | `/usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` |
| Parametreler | `/usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt` |

---

## OSINT Platform Referansları

| Platform | Birincil Kullanım |
|----------|-------------------|
| [https://securitytrails.com](https://securitytrails.com) | DNS geçmişi, subdomain verileri |
| [https://shrewdeye.app](https://shrewdeye.app) | Hızlı pasif subdomain bulucu |
| [https://netlas.io](https://netlas.io) | Derin altyapı araması |
| [https://urlscan.io](https://urlscan.io) | URL ve sayfa analizi |
| [https://search.censys.io](https://search.censys.io) | İnternet geneli host taraması |
| [https://www.shodan.io](https://www.shodan.io) | IoT ve servis keşfi |
| [https://otx.alienvault.com](https://otx.alienvault.com) | Tehdit istihbaratı |
| [https://crt.sh](https://crt.sh) | Sertifika şeffaflığı kayıtları |
| [https://bgp.he.net](https://bgp.he.net) | ASN ve BGP yönlendirme verileri |
| [https://search.dnslytics.com/cidr](https://search.dnslytics.com/cidr) | CIDR tabanlı domain araması |
| [https://viewdns.info](https://viewdns.info) | Tersine IP ve DNS geçmişi |
| [https://www.virustotal.com](https://www.virustotal.com) | Çok kaynaklı domain istihbaratı |
| [https://builtwith.com](https://builtwith.com) | Teknoloji yığını ve GA ID araması |
| [https://api.hackertarget.com](https://api.hackertarget.com) | Analytics ilişki haritalama |
| [https://asnlookup.com](https://asnlookup.com) | Org adına göre ASN araması |
| [https://bgp.tools](https://bgp.tools) | Modern BGP/ASN gezgini |
| [https://subdomainfinder.c99.nl](https://subdomainfinder.c99.nl) | Hızlı subdomain araması |
| [https://www.favihash.com](https://www.favihash.com) | Manuel favicon hash üretici |

---

> Metodoloji gerçek görevlerden oluşturulmuştur. Buradaki her komut canlı hedeflere karşı çalıştırılmıştır.

---

 © 2026
