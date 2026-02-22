# Langflow ile RAG Tabanlı Restoran S&C Uygulaması

Langflow kullanılarak kurulmuş **Retrieval-Augmented Generation (RAG)** tabanlı bir soru-cevap uygulaması. PDF dokümanına dayalı, kod yazmadan görsel node’larla oluşturulmuş chatbot.

---

##  İçindekiler

- [Genel Bakış](#genel-bakış)
- [Proje Yapısı (Akış Adımları)](#proje-yapısı-akış-adımları)
- [İş Akışı (Flow) Açıklaması](#iş-akışı-flow-açıklaması)
- [Kurulum](#kurulum)
- [Kullanım](#kullanım)
- [GitHub Notları](#github-notları)

---

## Genel Bakış

Uygulama şu senaryoyu hedefler:

- **Restoran Q&A PDF** gibi bir dokümandaki sık sorulan sorulara (çalışma saatleri, konum, ödeme seçenekleri, menü vb.) otomatik cevap vermek.
- Kullanıcı adına göre **konuşma geçmişini (message history)** saklamak; farklı kullanıcılar kendi geçmişlerini görür.
- **Vector store** (Astra DB) ile dokümandan anlamsal arama yapıp, LLM'e sadece ilgili bağlamı vererek doğru ve güncel cevaplar üretmek.

Tüm bu akış Langflow’da **görsel node’lar** ile bağlanarak oluşturulur; kod yazılmaz.

---

## Proje Yapısı (Akış Adımları)

### 1. Kurulum ve Langflow’u Çalıştırma

- **Python 3.10+** gereklidir.
- Langflow prerelease kurulumu:
  - Windows: `pip install langflow --pre --force-reinstall`
  - Mac/Linux: `pip3 install langflow --pre --force-reinstall`
- Çalıştırma: `langflow run` → Tarayıcıda `localhost` üzerinden arayüz açılır.
- Yeni proje: **New Project** → **Blank Flow** seçilir. Akışlar JSON olarak saklanır; import/export mümkündür.

### 2. PDF ve Temel Girdiler

- Kullanılacak **PDF** (örneğin restoran S&C) proje klasöründe veya Langflow’da erişilebilir bir yerde tutulur (örn. `Resturaunt Q&A.pdf`).
- **Inputs** kısmından:
  - **Text Input** → Kullanıcı adı (Name) için.
  - **Chat Input** → Kullanıcının soracağı soru için.
  - **Prompt** (Prompt Template) → LLM’e gidecek metni şablonla oluşturmak için eklenir.

### 3. Prompt Şablonu ve Değişkenler

Prompt Template’te kullanılan şablon örneği:

```
Hey, answer the users question based on
the following context.
The context in this {context}
And this is the message history: {history}
```

- **Değişkenler:** `{context}`, `{history}`, `{question}` (veya `question` alanı).
- **context** → Vector store’dan gelen ilgili metin (RAG sonucu).
- **history** → Message History’den gelen konuşma geçmişi.
- **question** → Chat Input’tan gelen kullanıcı sorusu.

Bu üçü Prompt Template’e bağlanır.

### 4. Mesaj Geçmişi (Message History)

- **Message History** (veya Chat Memory) komponenti eklenir.
- **Session ID** olarak **Name** (Text Input) verilir. Kullanıcı adı değiştikçe farklı oturumlar oluşur; her kullanıcı kendi mesaj geçmişini görür.
- **Message** → Chat Input’tan gelen mesaj.
- **Stored Messages** çıktısı Prompt Template’teki **history** alanına bağlanır.
- Böylece LLM hem bağlam hem de son birkaç mesajı görerek cevap verir.

### 5. OpenAI Entegrasyonu

- **Models** bölümünden **OpenAI** seçilir.
- [OpenAI API Keys](https://platform.openai.com/api-keys) sayfasından **API Key** oluşturulur. (Hesabın fonlanmış olması gerekebilir.)
- Langflow’da bu anahtar **Credential** tipinde bir değişkene (örn. `OPENAI_API_Key`) atanır; güvenli saklama için credential kullanılır.
- Prompt Template’in çıktısı **OpenAI** node’unun **Input**’una bağlanır.
- **Chat Output** eklenir; OpenAI’nin **Model Response** çıktısı Chat Output’a verilir, **Sender Name** "AI" yapılır.

Bu aşamada RAG olmadan basit bir chatbot çalışır; “Run” ile test edilir.

### 6. Vector Store (Astra DB) Kurulumu

- **DataStax Astra DB** kullanılır (ücretsiz tier mevcut).
- [Astra DB](https://dtsx.io/3vZk6n2) üzerinden hesap açılır, **Create Database** → **Serverless Vector** seçilir.
- Database adı (örn. `langflow_tutorial`), provider ve bölge seçilir. Oluşturulduktan sonra:
  - **Token** (Application Token)
  - **API Endpoint** (URL)
  - **Collection** adı  
  alınır. Bunlar Langflow’da değişken olarak (credential/generic) tanımlanır.

### 7. Doküman İşleme ve Veritabanına Yükleme

- **Read File** (veya Generic File Loader): PDF seçilir (örn. `Resturaunt Q&A.pdf`).
- **Split Text**: 
  - **Chunk Size**: 1000 (karakter)
  - **Chunk Overlap**: 200
  - Böylece metin küçük parçalara bölünür; overlap bağlam kaybını azaltır.
- **OpenAI Embeddings**:
  - Model: `text-embedding-3-small`
  - Aynı OpenAI API Key kullanılır.
- **Astra DB** node’u:
  - Token, Endpoint, Database, Collection bağlanır.
  - **Ingest Data** → Split Text’in **Chunks** çıktısı.
  - **Embedding Model** → OpenAI Embeddings’in çıktısı.
  Bu sayede PDF chunk’ları vektörlenip Astra DB’de saklanır. (İlk çalıştırmada vector store oluşturulur.)

### 8. RAG: Arama ve Prompt’a Bağlama

- **Chat Input** (kullanıcı sorusu), **Astra DB** node’undaki **Search Query** (veya ilgili arama girişi) alanına bağlanır.
- Astra DB, bu sorguyu aynı **OpenAI Embeddings** ile vektörleyip veritabanında benzerlik araması yapar; sonuçlar **Search Results** olarak döner.
- **Parser** (Stringify modunda): Search Results’ı metne çevirir → bu çıktı Prompt Template’teki **context** alanına verilir.
- Artık her kullanıcı sorusunda:
  1. Soru Astra DB’de aranır.
  2. İlgili parçalar `context` olarak alınır.
  3. `context`, `history` ve `question` Prompt Template’te birleştirilir.
  4. OpenAI cevap üretir; Chat Output’ta gösterilir.

### 9. Test ve Ek Notlar

- **İlk soru** bazen anlamsız cevap verebilir; vector store ilk kez doldurulurken gecikme olabilir. İkinci sorudan itibaren RAG düzgün çalışır.
- **İsim değiştirme**: Name alanı değiştirilince Session ID değişir; mesaj geçmişi o kullanıcıya özel olur.
- **Export**: Akış **Export** ile JSON olarak indirilebilir; başka bir Langflow kurulumuna **Import** edilebilir.

---

## İş Akışı (Flow) Açıklaması

Akıştaki node’ların özeti:

| Bölüm | Node'lar | Açıklama |
|--------|----------|----------|
| **Doküman girişi** | Read File → Split Text | PDF okunur, 1000 karakterlik chunk’lara (200 overlap) bölünür. |
| **Embedding** | OpenAI Embeddings | Chunk’lar `text-embedding-3-small` ile vektörlenir. |
| **Vector store** | Astra DB | Chunk’lar + embedding model ile Astra’ya yazılır; ayrıca kullanıcı sorusu burada aranır (Search Query). |
| **Kullanıcı girişi** | Chat Input, Text (Name) | Soru ve isim girilir; Name Message History’de Session ID olarak kullanılır. |
| **Geçmiş** | Message History | Chat mesajları saklanır; çıktı Prompt’taki `history` alanına gider. |
| **RAG birleştirme** | Parser → Prompt Template | Astra’dan gelen Search Results Parser’da string’e çevrilir → `context`. `context` + `history` + `question` ile prompt oluşturulur. |
| **Üretim** | OpenAI (gpt-4o-mini) | Düşük temperature (~0.18–0.19) ile cevap üretilir. |
| **Çıktı** | Chat Output | Model Response kullanıcıya “AI” adıyla gösterilir. |

**RAG’ın anlamı:** Kullanıcı soru sorduğunda önce PDF’ten ilgili kısımlar vector search ile bulunur, sonra bu bağlam LLM’e verilir; böylece cevap dokümana dayalı ve daha isabetli olur.

---

## Kurulum

1. **Python 3.10+** yüklü olsun.
2. Langflow’u kurun ve çalıştırın:
   ```bash
   pip install langflow --pre --force-reinstall
   langflow run
   ```
3. Tarayıcıda açılan arayüzden **New Project** → **Blank Flow** ile yeni akış oluşturun (veya bu repodaki JSON flow’u import edin).
4. **OpenAI API Key** ve **Astra DB** (Token, Endpoint, Collection) bilgilerini Langflow’daki değişkenlere/credential alanlarına girin.
5. **Read File** node’unda kullanacağınız PDF’i (örn. `Resturaunt Q&A.pdf`) seçin veya proje klasörüne koyup yolunu verin.

---

## Kullanım

1. Langflow’da **Run** ile akışı başlatın.
2. **Name** alanına kullanıcı adı yazın.
3. **Chat Input** alanına sorunuzu yazın (örn. “Tell me about the hours of the store”, “Can you tell me about the specials and the menu?”).
4. Cevap, PDF bağlamı ve konuşma geçmişi kullanılarak sağ taraftaki Chat Output’ta görünür.

---

## GitHub Notları

- **Flow JSON**: Langflow’dan **Export** ile akışı JSON olarak indirip repoya ekleyin (örn. `rag_flow.json`). Başkaları **Import** ile aynı akışı kurabilir.
- **PDF**: `Resturaunt Q&A.pdf` (veya kullandığınız PDF) repoya eklenebilir; `.gitignore` ile büyük dosyaları hariç tutabilirsiniz.
- **Gizli bilgiler**: API key ve Astra token’ları repoya eklemeyin. Langflow’da credential/variable kullanın; OpenAI API Key ve Astra bilgilerini kullanıcılar kendi ortamlarında girer.
- **Collection adı**: Astra tarafında kullandığınız collection adıyla Langflow’daki ayarın tutarlı olmasına dikkat edin.

---

## Kaynaklar

- [Langflow Docs / Install](https://docs.langflow.org/) · [Langflow GitHub](https://github.com/langflow-ai/langflow)  
- [Astra DB](https://dtsx.io/3vZk6n2)

 
 
