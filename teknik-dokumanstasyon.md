# Elastic SIEM Kurulumu ve Log Yönetimi

Bu dokümantasyon, VirtualBox üzerinde kurulan 5 farklı Ubuntu Server 24.04 sunucusunun log yönetimi, gerekli servislerin kurulumu ve ağ yapılandırması üzerine kuruludur.

### Projenin Amacı

Bu projede, Elastic SIEM sistemini sanal makineler üzerine kurarak log yönetimi yapılabilecek bir mimari oluşturmak hedeflenmiştir. Log yönetimi için Elasticsearch, logların görselleştirilmesi için Kibana, log kaynağı olarak ağ trafiğindeki tehditler için SNORT, donanım kullanım yüzdesini takip eden syslog formatında log üreten bir bash scripti ve NGINX erişim logları kullanılmıştır. Elasticsearch düğümlerine yük dağıtımı ve Kibana’ya ters proxy işlemleri için NGINX yapılandırılmıştır. Sunucuların birbirleriyle haberleşebilmesi için UFW ile port düzenlemeleri yapılmıştır.

### Kullanılan Teknolojiler

- **VirtualBox**: Sanal makinelerin oluşturulması ve yönetimi için kullanılmıştır.
- **Ubuntu Server 24.04**: Hafif ve tutarlı bir server olarak her bir sanal makineye işletim sistemi olarak kurulmuştur.
- **Elasticsearch**: Log verilerinin indekslenmesi ve yönetimi için kullanılmıştır.
- **Kibana**: Elasticsearch ile entegre olarak logların görselleştirilmesi için kullanılmıştır.
- **Snort**: Ağda oluşan trafiği belirli kurallar dahilinde inceleyip loglamak için kullanılmıştır.
- **Nginx**: Kibanaya ters proxy ve elasticsearch düğümlerine yük dağıtımı yaparken kullanılmıştır.
- **Ufw**:Sunucu üzerindeki portların erişim izinlerini ve sunucuya erişilebilecek ip adresslerini tanımlarken kullanılmıştır.
- **Filebeat**: Sistemde oluşan log dosyalarını elasticsearch düğümlerine göndermek için kullanılmıştır.
  
## Elasticsearch Sunucusu Dokümantasyonu

### Sunucunun Amacı

Bu sunucu, log verilerinin indekslenmesi ve depolanması için kullanılır. Projemizde, iki Elasticsearch sunucusu kullanılarak bir cluster oluşturulmuştur. Bu clusterda bir master & data, bir de data node bulunuyor.

### Kurulum Adımları
1. **Java Kurulumu**: Elasticsearch java dilinde yazılmıştır. Bu yüzden javayı sunucumuza kurmamız gerekir.
    ``` 
    sudo apt update
    sudo apt install openjdk-11-jdk
    ```
2. **Elasticsearch Kurulumu**: Elasticsearch'ü apt reposu üzerinden indirelim.
    ```
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    sudo apt install apt-transport-https
    echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
    sudo apt update && sudo apt install elasticsearch
    ```
3. **Elasticsearch yapılandırması**: Elasticsearch yapılandırma dosyasının yolu: ``/etc/elasticsearch/elasticsearch.yml``
    + **Master & Data Düğüm İçin**:
        ```
        cluster.name: siem_cluster  // İki düğümde de aynı olmalı

        node.name: "node-1" // Eşsiz olmalı

        network.host: 0.0.0.0
        http.port: 9200

        discovery.seed_hosts: ["192.168.1.8","192.168.1.11"] // Cluster içinde bulunan sunucuların ip adresleri
        cluster.initial_master_nodes: ["node-1"] // Master olacak sunucunun ismi

        node.master: true
        node.data: true

        path.data: /var/lib/elasticsearch
        path.logs: /var/log/elasticsearch
        ```
    + **Data Düğüm İçin**:
        ```
        cluster.name: siem_cluster

        node.name: "node-2"

        network.host: 0.0.0.0
        http.port: 9200

        node.master: false
        node.data: true

        discovery.seed_hosts: ["192.168.1.8", "192.168.1.11"]

        path.data: /var/lib/elasticsearch
        path.logs: /var/log/elasticsearch
        ```
4. **Elasticsearch servisini başlatma**:
    ```
    sudo systemctl enable elasticsearch
    sudo systemctl start elasticsearch
    ```
5. **Cluster'ı Doğrulama**: 
  Elasticsearch sunucumuzun REST API'sine istek atarak cluster'da kaç tane sunucu çalıştığını kontrol edelim.
    ``` 
    curl -X GET "http://192.168.1.8:9200/_cat/nodes?v" 
    ```
    **Beklenen çıktı birden fazla düğüm olması**:
    ```
    ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
    192.168.1.11           62          93   6    0.00    0.00     0.00 cdfhilrstw  -      node-2
    192.168.1.8            26          82   6    0.02    0.02     0.00 cdfhilmrstw *      node-1    
    ```
    
6. **Ufw ile Gerekli Portları Açalım**: Ufw ile Elasticsearch için gerekli olan portları açalım. 9200 portu Elasticsearch'ün REST API hizmetini sunduğu port. 9300 portu ise düğümlerin birbirileri arasında gRPC protokolü ile haberleştiği port bu yüzden ikisini de açmamız gerekiyor.
    ```
        sudo ufw allow 9200/tcp
        sudo ufw allow 9300/tcp
    ```
8. **Sonuç**: Elasticsearch sunucuları başarıyla kurulmuş ve yapılandırılmıştır. İki düğümlü bir cluster oluşturulmuştur.

## Kibana Sunucusu Dokümantasyonu

1. ### Sunucunun Amacı
    Kibana, Elasticsearch düğümlerindeki verileri görselleştiren bir analiz aracıdır. Kibana, Elasticsearch REST API'lerine istekler göndererek verileri bize gösterecektir.

2. ### Kurulum Adımları
   1. **Gerekli Paketlerin Kurulması**: Kibana'yı apt reposu üzerinden kurabiliriz.
        ```
        sudo apt update
        sudo apt install kibana
        ```
   2. **Kibana Yapılandırması**: Kibana'nın yapılandırma dosyası ``/etc/kibana/kibana.yml`` dosyasındadır.
        ```
            server.port: 5601
            server.host: "0.0.0.0"
            elasticsearch.hosts: ["http://192.168.1.12:9200"] //nginx'in ip adresini girdik. Nginx düğümlere yönlendiricek.
        ```
   3. **Kibana Servisini Başlatma**: 
        ```
        sudo systemctl enable kibana
        sudo systemctl start kibana
        ```
3. **Ufw ile Gerekli Portları Açalım**: Ufw ile Elasticsearch için gerekli olan portları açalım. 9200 portu Elasticsearch'ün REST API hizmetini sunduğu port. 9300 portu ise düğümlerin birbirileri arasında gRPC protokolü ile haberleştiği port bu yüzden ikisini bir den açmamız gerekiyor.
    ```
    sudo ufw allow 5601/tcp
    ```
4. ### Sonuç: 
    Kibana, başarıyla kurularak Elasticsearch ile entegre edilmiştir. Log verilerinin görselleştirilmesi için kullanılmaktadır.

## Snort Sunucu Dokümantasyonu
1. ### Sunucunun Amacı
    Bu sunucu, ağ trafiğini izleyip potansiyel tehditleri tespit etmek için kullanılır. Projemizde, ağ güvenliği izleme ve log kaynağı olarak kullanılmaktadır.

2. ### Kurulum Adımları
   1. **Gerekli Paketlerin Kurulması**: Kibana'yı apt reposu üzerinden kurabiliriz.
        ```
        sudo apt update
        sudo apt install snort
        ```
    2. **Community kurallarının yüklenmesi**:
        ```
        wget https://www.snort.org/rules/community -O /etc/snort/rules/community.rules
        ```
   3. **Snort Yapılandırması**: Snort'un yapılandırma dosyası /etc/snort/snort.conf içerisinde yer alır.
        ```
        include /etc/snort/rules/community-rules/community.rules  
        ```
   4. **Snort'u Test Etme**: Snort’un doğru çalışıp çalışmadığını test edelim.
        ```
        sudo snort -T -c /etc/snort/snort.conf
        ```
    
3. **Logları Kontrol Edelim**:
   + Snort loglarını /var/log/snort/alert dizininde saklamaktadır. Bu loglar daha sonra Filebeat kullanılarak Elasticsearch'e gönderilecektir.
4. **Ufw ile Gerekli Portları Açalım**: Ufw ile Kibana'nın çalıştığı portu açalım.
    ```
    sudo ufw allow 5601/tcp
    ```
5. **Snort'u Çalıştıralım**: enp0s3 arayüzündeki ağdaki trafiği dinleyecek ve community.rules içerisindeki kurallara göre tehdit içerenleri -l ile belirttiğimiz yere loglayacaktır.
    ```
    sudo snort -A fast -q -c /etc/snort/snort.conf -i enp0s3 -l /var/log/snort/
    ```
6. ### Sonuç: 
    Snort sunucusu başarıyla kurulmuş ve ağ trafiğini izlemek üzere yapılandırılmıştır. Loglar, belirli kurallara göre oluşturulacak ve dosyalarda loglanıcaktır. Sonrasında Filebeat ile bu log dosyalarını belirli bir formatta Elasticsearch düğümlerin göndereceğiz.

## Nginx Sunucu Dokümantasyonu
1. ### Sunucunun Amacı
    Bu sunucu, ters proxy ve yük dağıtımı işlemlerini yapmak için kullanılır. Projemizde, Nginx'in bulunduğu sunucunun 80 ve 443 portuna gelen istekleri Kibana'nın bulunduğu sunucuya ve porta yönlendirecektir. 9200 portuna gelen istekleri ise Elasticsearch düğümlerine dağıtarak yönlendirecektir. Sırasıyla bu işlemlere ters proxy ve yük dağıtımı deniyor.

2. ### Kurulum Adımları
   1. **Gerekli Paketlerin Kurulması**: Kibana'yı apt reposu üzerinden kurabiliriz.
        ```
        sudo apt update
        sudo apt install nginx
        ```
    2. **SSL Sertifikası Oluşturma**: Openssl ile kendi imzaladığımız ssl sertifikasını oluşturalım. İnternette geçerliliği olan bir sertifika değildir. Tarayıcılardaki belirli otoritelerin imzalaması lazım öyle olması için. 
        ```
        sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
        ```
   3. **Nginx Yapılandırması**: Nginx yapılandırma dosyası ``/etc/nginx/nginx.conf`` altındadır. Ters proxy ve yük dağıtımı yapılandırmasını yapalım.
        ```
            upstream elasticsearch {
                server 192.168.1.8:9200; #elasticsearch node1
                server 192.168.1.11:9200; #elasticsearch node2
            }

            server {
                listen 9200;

                location / {
                    proxy_pass http://elasticsearch;  //yukarıda tanımladığımız upstream
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                }
            }

            server {
                listen 80;

                location / {
                    proxy_pass http://192.168.1.13:5601;  //kibana'nın çalıştığı sunucu ip adresi ve portu
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection 'upgrade';
                    proxy_set_header Host $host;
                    proxy_cache_bypass $http_upgrade;
                }
            }

            server {
                listen 443 ssl;

                ssl_certificate /etc/nginx/certs/fullchain.pem;
                ssl_certificate_key /etc/nginx/certs/privkey.pem;

                location / {
                    proxy_pass https://192.168.1.13:5601;
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection 'upgrade';
                    proxy_set_header Host $host;
                    proxy_cache_bypass $http_upgrade;
                }
            }
        ```
    4. **nginx.conf Dosyasını Test edelim**:
        ```
        sudo nginx -t
        ```
   4. **Nginx Servisini Başlatma**:
        ```
        sudo systemctl enable nginx
        sudo systemctl start nginx
        ```
    5. **Ufw ile gerekli portların açılması**: ufw ile 80, 443, 9200 portlarını açalım.
        ```
        sudo ufw allow 80/tcp
        sudo ufw allow 443/tcp
        sudo ufw allow 9200/tcp
        ```
3. **Doğrulama ve Testler**:
   1. **Ters proxy Kontrolü**: ``https://192.168.1.12:443``(Nginx'in bulunduğu sunucudaki ip adresi ve 443 portuna) gittiğimizde Kibana'ya erişebiliyor muyuz? 
    2. **Yük Dağıtımı Testi**: ``https://192.168.1.12:9200``(Nginx'in bulunduğu sunucudaki ip adresi ve 9200 portuna) gittiğimizde Elasticsearch düğümleri hakkında yanıt(response) alabiliyor muyuz?
4. ### Sonuç: 
    Nginx sunucusu başarıyla ters proxy ve yük dağıtımı yapılandırılmıştır. SSL sertifikası ile güvenli bağlantı sağlanmış ve Nginx, Kibana ve Elasticsearch arasındaki istekler başarıyla yönlendirilmiştir.

## Filebeat Kurulumu
1. ### Filebeat'in Amacı
    Bu yazılım, Snort'un logları, Nginx'in erişim logları ve yazdığımız scriptin ürettiği logları, Elasticsearch düğümlerinin REST API'lerine istek atarak gönderir.
2. ### Kurulum Adımları
   1. **Filebeat Paketini İndirin**:
        ```
        wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.14.0-amd64.deb
        ```
    2. **DEB Paketini Yükleyin**:
        ```
        sudo dpkg -i filebeat-7.14.0-amd64.deb
        ```
   2. **Filebeat Yapılandırması**: Filebeat'in yapılandırma dosyası /etc/filebeat/filebeat.yml içinde yer alır.
        ```
        sudo nano /etc/filebeat/filebeat.yml
        ````
        ```
        filebeat.inputs:
            - type: log
                enabled: true
                paths:
                - /var/log/snort/snort.alert.fast
                fields:
                log_type: snort
                fields_under_root: true
                index: "snort-logs-%{+yyyy.MM.dd}"
            - type: log
            enabled: true
            paths:
                - /var/log/nginx/access.log
            fields:
                log_type: nginx_access
            fields_under_root: true

            index: "nginx-access-logs-%{+yyyy.MM.dd}"  // buradaki nginx erişim log yapılandırması nginx sunucusunun içinde
            - type: log
                enabled: true
                paths:
                - /var/log/syslog
                fields:
                log_type: syslog
                fields_under_root: true
                index: "syslog-logs-%{+yyyy.MM.dd}"
        output.elasticsearch:
            hosts: ["http://192.168.1.12:9200"]
        ```
   3. **Filebeat’i Başlatma**: 
        ```
        sudo systemctl start filebeat
        sudo systemctl enable filebeat
        ```
3. **Çalıştığını Kontrol Edelim**:
    ```
    sudo systemctl status filebeat
    ```
4. **Config Dosyasını Test Edelim**:
    ```
    sudo filebeat test config
    ```
5. ### Sonuç: 
    Snort sunucusu başarıyla kurulmuş ve ağ trafiğini izlemek üzere yapılandırılmıştır. Loglar, belirli kurallara göre oluşturulacak ve dosyalarda loglanıcaktır. Sonrasında Filebeat ile bu log dosyalarını belirli bir formatta Elasticsearch düğümlerin göndereceğiz.

## Elastic SIEM Mimarisi

**Mimarimin resim halini aşağıdaki linkte bulabilirsiniz.**
```
https://kendime-ozel-public-bucket.s3.amazonaws.com/Elastic_SIEM_Network.png
```
**Mimariyi anlattığım videoya da aşağıdaki linkten ulaşabilirsiniz.**
```
https://kendime-ozel-public-bucket.s3.amazonaws.com/unknown_2024.10.19-20.49.mp4
```