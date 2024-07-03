# Web Scraping ve Veri Entegrasyonu için Otomatik Veri Toplama ve Analiz
## 1. Giriş
Web teknolojilerinin hızla gelişmesiyle birlikte, internet üzerindeki verilere erişim ve bu verilerin analizi, bilimsel araştırmalarda ve endüstriyel uygulamalarda büyük önem kazanmıştır. Web scraping, bu bağlamda web sitelerinden otomatik olarak veri toplama işlemlerini ifade eder. Otomatik veri toplama işlemleri, metin madenciliği, duygu analizi, içerik önerileri ve bilgi yönetimi gibi alanlarda kullanılmaktadır.
>[IMPORTANT]
>Bu doküman içinde, Selenium ve Sentence Transformers gibi güçlü araçlar kullanılarak gerçekleştirilen bir web scraping ve veri entegrasyon projesi sunulmaktadır. Proje kapsamında, Neo4j graf veritabanı kullanılarak toplanan verilerin depolanması, ilişkilendirilmesi ve analiz edilmesi amaçlanmıştır.

## 2 Kullanılan Teknolojiler ve Kütüphaneler
- **Web Scraping Teknolojileri:** Web scraping, web sitelerinden veri çekme işlemidir ve bu işlem genellikle otomatik botlar veya scriptler aracılığıyla gerçekleştirilir. Selenium, Python tabanlı bir otomasyon aracıdır ve web tarayıcıları üzerinde işlem yaparak dinamik web sayfalarından veri çekebilir. Bu projede, Selenium'un web tarayıcıları üzerindeki güçlü kontrol yetenekleri kullanılarak veri toplama işlemi gerçekleştirilmiştir.
- **Metin Gömme Vektörleri ve Analiz:** Veri toplama sürecinde, Sentence Transformers kütüphanesi kullanılarak metinlerin gömme vektörleri oluşturulmuştur. Gömme vektörleri, metinlerin semantik benzerliklerini ölçmek için kullanılan matematiksel temsillerdir. Bu proje kapsamında, sayfa içeriğinin anlamsal yapıları Sentence Transformers aracılığıyla analiz edilmiş ve gömme vektörleri elde edilmiştir.
- **Neo4j Veri Yönetim Sistemi:** Toplanan veriler, Neo4j graf veritabanı yönetim sistemi kullanılarak depolanmıştır. Neo4j, ilişkisel veri tabanlarına alternatif olarak graf yapıları üzerinde etkili bir şekilde veri modelleme ve sorgulama imkanı sunar. Bu proje, toplanan sayfa içeriği, bağlantılar ve anahtar kelimelerin Neo4j veritabanına entegrasyonunu ve analizini içermektedir.

```python
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.by import By
import undetected_chromedriver.v2 as uc
from pyvirtualdisplay import Display
from sentence_transformers import SentenceTransformer
from transformers import AutoTokenizer, AutoModelForTokenClassification, AutoModel
from transformers import pipeline
from neo4j import GraphDatabase
```


## 3. Projenin Amacı ve Katkısı
Bu projenin amacı, otomatik veri toplama ve analiz süreçlerini kullanarak web üzerindeki bilgiye erişimi kolaylaştırmak ve bu bilgileri anlamlı yapılar içinde organize ederek bilgi yönetimini iyileştirmektir. Ayrıca, Sentence Transformers gibi yapay zeka modellerinin ve Neo4j gibi veritabanı teknolojilerinin pratik uygulamalarını göstererek bu alanlarda kullanım potansiyellerini ortaya koymaktır.
 
## 4. Uygulama Aşamaları
### **4.1. Veri Toplama ve Analiz:**
- Selenium aracılığıyla Neo4j dokümantasyon sitesinden veri toplama:
    - Metin içeriği ve bağlantılarının çıkarılması.
    - Anahtar kelimelerin belirlenmesi ve metinlerin gömme vektörlerinin oluşturulması.
### **4.2. Veri Entegrasyonu:**
- Toplanan verilerin analiz edilmesi:
  - Sayfa istatistikleri ve içerik özellikleri (metin varlığı, 404 durumu).
  - Anahtar kelimelerin ve metin gömme vektörlerinin kullanımıyla içerikler arası benzerlik analizleri.
 
### **4.3 Fonksiyonlar ve İşlemleri:**
- Bu fonksiyon, belirtilen bir HTML sınıf adına sahip elementten metin çıkarmak için kullanılır. **wd** global değişkeni üzerinden Selenium WebDriver ile etkileşim sağlanır.
```python
def extract_text_by_class(class_name):
    global wd
    try:
        content = wd.find_element(By.CLASS_NAME, class_name)
        return content.text
    except:
        return ""
```
- Bu fonksiyon, belirtilen XPath ile bağlantıları (linkleri) çıkarmak için kullanılır. Bağlantıların filtrelenmesi ve temizlenmesi işlemleri de bu fonksiyonda gerçekleştirilir.
```python
def extract_links_by_xpath(xpath):
    global wd
    links = set()
    try:
        a_elems = wd.find_elements(By.XPATH, xpath)
        for elem in a_elems:
            link = elem.get_attribute("href")
            if link == "javascript:void(0)":
                continue
            # Resimlere ve çeşitli dosyaları filtreleme işlemi
            if (
                link.endswith(".png")
                or link.endswith(".json")
                or link.endswith(".txt")
                or link.endswith(".svg")
                or link.endswith(".ipynb")
                or link.endswith(".jpg")
                or link.endswith(".pdf")
                or link.endswith(".mp4")
                or "mailto" in link
                or len(link) > 300
            ):
                continue
            # Remove anchors
            link = link.split("#")[0]
            # Remove parameters
            link = link.split("?")[0]
            # Remove trailing forward slash
            link = link.rstrip("/")
            links.add(link)
        return list(links)
    except:
        return []
```
- Belirli bir doğal dil işleme (NLP) modelinin tanımlanması için kullanılır. 
```python
tokenizer = AutoTokenizer.from_pretrained("yanekyuk/bert-uncased-keyword-extractor")
model = AutoModelForTokenClassification.from_pretrained(
    "yanekyuk/bert-uncased-keyword-extractor"
)

nlp = pipeline("ner", model=model, tokenizer=tokenizer)
```
- Bu fonksiyon, verilen metinden anahtar kelimeleri çıkarmak için kullanılır. nlp pipeline'i ile Named Entity Recognition (NER) kullanılarak anahtar kelimeleri belirler.
```python
def extract_keywords(text):
    """
    Verilen metinden anahtar kelimeleri çıkarmak için kullanılır.
    """
    result = list()
    keyword = ""
    for token in nlp(text):
        if token["entity"] == "I-KEY":
            keyword += (
                token["word"][2:]
                if token["word"].startswith("##")
                else f" {token['word']}"
            )
        else:
            if keyword:
                result.append(keyword)
            keyword = token["word"]
    # Son anahtar kelimeyi ekle
    result.append(keyword)
    return list(set(result))
```
- Bu fonksiyon, verilen metnin gömülmesini (embedding) oluşturmak için Sentence Transformers modelini kullanır. Bir diğer adı ile vektörel işlemlere çevrilme işlemidir.
```python
model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")
def generate_embeddings(text):
    embeddings = model.encode(text)
    return [float(x) for x in embeddings.tolist()]
```
>[!IMPORTANT]
>- Bu bölümde, Selenium kullanılarak belirli bir web sitesinden veri toplama ve bu verileri işleme işlemleri gerçekleştirilir. Metin çıkarma, bağlantıları toplama, gömme oluşturma ve anahtar kelimeleri çıkarma işlemleri yapılır. Üstteki fonksiyonların kullanıldığı Main methodu olarak düşünülebilir.
```python
options = webdriver.ChromeOptions()
options.add_argument("--no-sandbox")
wd = webdriver.Chrome(options=options)

entry_url = "https://neo4j.com/docs"
data = dict()
visit_list = [entry_url]
already_visited = []
visited_links_count = 0

while visit_list and visited_links_count < 20:
    current_url = visit_list.pop()
    if current_url in already_visited:
        continue
    try:
        wd.get(current_url)
    except:
        print(f"Couldn't open {current_url}")
        already_visited.append(current_url)
        continue
    
    # Sayfa url'lerini temizle
    try:
        actual_url = wd.current_url.rstrip("/").split("#")[0].split("?")[0]
        if actual_url != current_url:
            data[current_url] = {
                "links": [],
                "text": None,
                "embeddings": [],
                "keywords": [],
                "redirects": [actual_url],
            }
            already_visited.append(current_url)
            current_url = actual_url
    except:
        pass
    # Metin çıkar
    text = extract_text_by_class("content")
    if not text:
        text = extract_text_by_class("article")
    if not text:
        text = extract_text_by_class("page")
    if not text:
        text = extract_text_by_class("single-user-story")
    try:
        if "Sorry, page not found" in wd.find_element(By.TAG_NAME, "body").text:
            text = "404"
    except:
        pass
    if text:
        embeddings = generate_embeddings(text)
        keywords = extract_keywords(text)
    else:
        embeddings = []
        keywords = []
    # Bağlantıları çıkar
    links = extract_links_by_xpath("//div[@class='content']//a[@href]")
    if not links:
        links = extract_links_by_xpath("//article[@class='article']//a[@href]")
    if not links:
        links = extract_links_by_xpath("//article//a[@href]")
    # Veri dict ekle
    data[current_url] = {
        "links": [l for l in links if l != current_url],
        "text": text,
        "embeddings": embeddings,
        "keywords": keywords,
        "redirects": [],
    }
    already_visited.append(current_url)
    visited_links_count += 1
    visit_list.extend(
        [
            l
            for l in list(links)
            if ("neo4j.com" in l)
            and (not l in already_visited)
            and (not "community.neo4j.com" in l)
            and (not "sandbox.neo4j.com" in l)
        ]
    )
wd.quit()
```
>[!IMPORTANT]
>- Neo4j Database Bağlantı bilgilerinin girildiği ve aktarıldığı kısımdır. Kendi bilgilerinize göre işlem yapmalısınız!
```python
uri = "n"
user = ""
password = ""

try:
    driver = GraphDatabase.driver(uri, auth=(user, password))
    with driver.session() as session:
        result = session.run("RETURN 1")
        for record in result:
            print(record)
except Exception as e:
    print("Error:", e)
```

## 5. Öneriler ve Gelecek Çalışmalar
- Sistemin işlemini kodlar üzerinden çalışması yerine *Streamlit Kütüphanesi* veya *Django Kütüphaneleri* kullanılarak arayüz tasarlanması önerilir.  
- **Doğal dil işleme (NLP)** modelleri ve **Uzman Sistemler** kullanılarak sitenin her **Front-End** ve **Content** içerikleri hakkında yorumlar ve geliştirme önerileri verilebilmesi hedeflenmiştir.

## 6. Sonuçlar ve Analiz
- Toplanan verilerin analiz edilmesi:
  - Sayfa istatistikleri ve içerik özellikleri (metin varlığı, 404 durumu).
  - Anahtar kelimelerin ve metin gömme vektörlerinin kullanımıyla içerikler arası benzerlik analizleri.
- Graf Veri Tabanı içindeki görünüşü

| Görsel | Açıklama |
| --- | --- |
| ![Kuş bakışı graf veri tabanı](https://github.com/BurakErkemen/Graph_Neo4j/assets/84676805/52a62bee-edcb-4be2-8b42-b635e373f291) | Kuş bakışı graf veri tabanı gösterilmiştir. |
| ![Yakından düğümler ve ilişkiler](https://github.com/BurakErkemen/Graph_Neo4j/assets/84676805/18ceea9b-332c-4b78-b31d-7084251b480b) | Yakından düğümler ve ilişkileri incelenmiştir. |
| ![Sayfa düğümü bilgileri](https://github.com/BurakErkemen/Graph_Neo4j/assets/84676805/f5c2dcfa-bba7-4a83-b595-33f3c3f942b1) | Sayfa düğümü bilgileri gösterilmiştir. |
| ![Keyword içeriğinin düğümü bilgileri](https://github.com/BurakErkemen/Graph_Neo4j/assets/84676805/a2d71311-fa76-4dd5-b899-df9c5f0e1345) | Keyword içeriğinin düğümü bilgileri gösterilmiştir. |
| ![Neo4j RAW sorgu işlemi ve veri tabanı bilgileri](https://github.com/BurakErkemen/Graph_Neo4j/assets/84676805/6436abee-e158-4719-905b-c7ab012c742b) | Neo4j RAW içerisindeki sorgu işlemi ve veri tabanı bilgileri gösterilmiştir. |
| ![Embedding yapılan metinsel ifadeler](https://github.com/BurakErkemen/Graph_Neo4j/assets/84676805/dc313fd8-5eeb-4b54-a542-d385d143fddb) | Embedding yapılan metinsel ifadeler gösterilmiştir. |

