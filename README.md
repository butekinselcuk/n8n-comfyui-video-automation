# Docker ile Uçtan Uca Video Üretim Pipeline'ı

Bu rehber, video üretimi için gerekli olan tüm servisleri Docker kullanarak kendi bilgisayarınızda kurmanızı sağlayacaktır. Kurulum tamamlandığında, birbiriyle iletişim kurabilen tam fonksiyonel bir video otomasyon sisteminiz olacaktır.

## 🧱 Sistemin Bileşenleri

* **n8n:** İş akışı otomasyonu ve tüm sürecin beyni.
* **ComfyUI:** Yapay zeka ile resim ve video üretimi için kullandığımız ana motor.
* **MinIO:** Üretilen resim, video ve ses dosyalarını saklamak için kullandığımız yerel bulut depolama.
* **Kokoro-TTS (Coqui TTS ile Değiştirildi):** Metinleri seslendirmek için kullandığımız TTS (Text-to-Speech) servisi. (Not: `kokoro-tts-cpu` yaygın bir Docker imajı olmadığı için, yerine popüler ve açık kaynaklı `Coqui TTS` kullanılmıştır. Aynı amaca hizmet eder.)
* **NCA Toolkit:** Videolara altyazı ekleme gibi çeşitli video işleme görevleri için kullandığımız araç kutusu.

---

## 🚀 Uçtan Uca Kurulum Adımları

Bu adımları sırasıyla takip ettiğinizde sisteminiz çalışmaya hazır olacaktır.

### Adım 1: Ön Koşul - Docker Desktop Kurulumu

Başlamadan önce bilgisayarınızda **Docker Desktop**'ın kurulu ve çalışır durumda olduğundan emin olun.

* [Docker Desktop'ı İndirin](https://www.docker.com/products/docker-desktop/)

### Adım 2: Proje Klasörünü Oluşturun

Tüm yapılandırma dosyalarını barındıracak bir klasör oluşturun ve o klasörün içine girin. Terminal veya Komut İstemcisi'nde aşağıdaki komutları çalıştırın:

```bash
mkdir video-pipeline
cd video-pipeline
```

### Adım 3: docker-compose.yml Dosyasını Oluşturun
Proje ana dizininizde (video-pipeline) docker-compose.yml adında bir dosya oluşturun ve aşağıdaki içeriğin tamamını bu dosyaya yapıştırın:

YAML

```# docker-compose.yml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - TZ=Europe/Istanbul
    restart: always

  comfyui:
    image: comfyanonymous/comfyui:latest
    container_name: comfyui
    ports:
      - "8188:8188"
    volumes:
      - ./comfyui_models:/app/ComfyUI/models
      - ./comfyui_input:/app/ComfyUI/input
      - ./comfyui_output:/app/ComfyUI/output
    restart: always
    # GPU KULLANIMI İÇİN AŞAĞIDAKİ "İLERİ SEVİYE" BÖLÜMÜNE BAKINIZ

  minio:
    image: minio/minio
    container_name: minio
    ports:
      - "9000:9000"  # API Portu
      - "9001:9001"  # Konsol Arayüzü
    volumes:
      - minio_data:/data
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    command: server /data --console-address ":9001"
    restart: always

  kokoro-tts-cpu:
    # Not: kokoro-tts-cpu yerine popüler Coqui TTS sunucusu kullanılmıştır.
    # n8n içinden http://kokoro-tts-cpu:5002/api/tts adresine istek atılabilir.
    image: coqui/tts-server:latest
    container_name: kokoro-tts-cpu
    ports:
      - "880:5002"
    restart: always

  nca-toolkit:
    # DİKKAT: Bu imaj bir yer tutucudur.
    # Lütfen kendi video işleme aracınızın Docker imaj adını buraya yazın.
    # Örnek: image: my-registry/my-nca-toolkit:1.0
    image: your-custom-nca-toolkit-image:latest
    container_name: nca-toolkit
    ports:
      - "8080:8080"
    restart: always

volumes:
  n8n_data:
  minio_data:
```


### Adım 4: ComfyUI Modelleri İçin Klasör Oluşturun
ComfyUI'ın çalışması için yapay zeka modellerine (checkpoint, LoRA vb.) ihtiyacı vardır. Bu modelleri bilgisayarınızda saklamak ve ComfyUI'a tanıtmak için docker-compose.yml dosyası ile aynı dizinde bir klasör oluşturalım:



İndireceğiniz .safetensors veya .ckpt uzantılı modelleri bu comfyui_models klasörünün içine atmanız gerekecektir.

### Adım 5: Tüm Sistemi Başlatın!
Artık her şey hazır. Terminalde, docker-compose.yml dosyasının bulunduğu dizindeyken aşağıdaki komutu çalıştırın:
```
docker-compose up -d
```

Bu komut, tanımlanan tüm servisleri (n8n, ComfyUI, MinIO vb.) arka planda başlatacaktır. İlk başlatma, imajların internetten indirilmesi nedeniyle birkaç dakika sürebilir.

✅ İlk Yapılandırma ve Erişim
Sisteminiz artık çalışıyor. Servislerin arayüzlerine tarayıcınızdan erişebilir ve ilk ayarları yapabilirsiniz.

n8n (İş Akışı):

URL: http://localhost:5678
Açıklama: İlk açılışta bir yönetici hesabı oluşturmanız istenecektir.
ComfyUI (Yapay Zeka Motoru):

URL: http://localhost:8188
Gereksinim: ComfyUI'ı kullanabilmek için yapay zeka modelleri indirip Adım 4'te oluşturduğunuz comfyui_models klasörüne kopyalamanız gerekir. Modelleri Civitai veya Hugging Face gibi sitelerden bulabilirsiniz.
MinIO (Depolama):

Konsol URL: http://localhost:9001
Kullanıcı Adı: minioadmin
Şifre: minioadmin
Gereksinim: n8n ve ComfyUI'ın dosya kaydedebilmesi için MinIO konsoluna giriş yapıp "Create Bucket" butonu ile bir veya daha fazla "kova" (örneğin: videos, images, audio) oluşturmanız gerekir.
Kokoro TTS (Metin Seslendirme):

URL: http://localhost:880 (Bu adres sizi Coqui TTS sunucusunun arayüzüne yönlendirir.)
API Adresi (n8n içinden): http://kokoro-tts-cpu:5002
NCA Toolkit (Video İşleme):

URL: http://localhost:8080
Not: Bu servisin çalışması için docker-compose.yml dosyasındaki your-custom-nca-toolkit-image:latest satırını kendi Docker imajınızla değiştirmiş olmalısınız.
💡 İleri Seviye: ComfyUI için GPU Kullanımı
Yapay zeka işlemleri CPU ile çok yavaş çalışır. Eğer NVIDIA bir ekran kartınız varsa, ComfyUI'ı GPU üzerinden çalıştırarak performansı 10-50 kat artırabilirsiniz.

NVIDIA Container Toolkit Kurulumu: Docker'ın ekran kartınızı kullanabilmesi için NVIDIA Container Toolkit kurulumunu yapmanız gerekir.

docker-compose.yml Dosyasını Güncelleyin: comfyui servisini aşağıdaki gibi düzenleyin.

```
# ... (dosyanın geri kalanı aynı)

  comfyui:
    image: comfyanonymous/comfyui:latest
    container_name: comfyui
    ports:
      - "8188:8188"
    volumes:
      - ./comfyui_models:/app/ComfyUI/models
      - ./comfyui_input:/app/ComfyUI/input
      - ./comfyui_output:/app/ComfyUI/output
    restart: always
    deploy: # Bu bölümü ekleyin
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

# ... (dosyanın geri kalanı aynı)
```

Değişikliklerin uygulanması için sistemi yeniden başlatın: docker-compose up -d --force-recreate

🛑 Servisleri Durdurma
Sistemi kapatmak için terminalde aşağıdaki komutu çalıştırın:


Bash
```
docker-compose down
```
