RAG Tabanlı Restoran Soru-Cevap Uygulaması (Langflow)

Bu proje, Langflow kullanılarak geliştirilmiş Retrieval-Augmented Generation (RAG) tabanlı bir soru-cevap uygulamasıdır. Sistem, bir PDF dokümanındaki bilgileri vektör veritabanına aktararak kullanıcı sorularına bağlama dayalı cevap üretir.

Uygulama tamamen görsel node yapısı ile oluşturulmuş olup herhangi bir manuel kod geliştirme süreci içermemektedir.

Proje Amacı

Bu çalışmanın amacı:

RAG mimarisini uygulamalı olarak kurmak

Vektör veritabanı ve LLM entegrasyonunu gerçekleştirmek

Session-based mesaj geçmişi yönetimini uygulamak

Langflow üzerinden uçtan uca bir yapay zeka akışı tasarlamak

Sistem Mimarisi

Akış şu adımlardan oluşmaktadır:

PDF dosyası okunur.

Metin parçalara (chunk) bölünür.

Her parça embedding model ile vektörleştirilir.

Vektörler Astra DB üzerinde saklanır.

Kullanıcı soru sorduğunda semantic search yapılır.

İlgili metin parçaları bağlam (context) olarak alınır.

Context + mesaj geçmişi + soru birleştirilerek modele gönderilir.

Model cevap üretir.

Kullanılan Teknolojiler

Langflow

OpenAI (LLM ve Embedding)

DataStax Astra DB (Vector Store)

Akış Bileşenleri
Katman	Node
Doküman Yükleme	Read File
Metin Bölme	Split Text (1000 chunk, 200 overlap)
Embedding	OpenAI Embeddings (text-embedding-3-small)
Vector Store	Astra DB (Serverless Vector)
Bellek	Message History (Session ID = Name)
Prompt	Prompt Template
Model	OpenAI (gpt-4o-mini)
Çıktı	Chat Output
Kurulum
1. Gereksinimler

Python 3.10+

OpenAI API Key

Astra DB hesabı

2. Langflow Kurulumu
pip install langflow --pre --force-reinstall
langflow run

Tarayıcıda:

http://localhost:7860

Yeni proje oluşturmak için:
New Project → Blank Flow

3. Astra DB Kurulumu

Astra DB hesabı oluşturulur.

Create Database → Serverless Vector seçilir.

Aşağıdaki bilgiler alınır:

Application Token

API Endpoint

Collection Name

Bu bilgiler Langflow içinde credential olarak tanımlanır.

4. OpenAI API Key

OpenAI panelinden API Key oluşturulur ve Langflow içerisinde credential olarak kaydedilir.

Kullanım

Flow çalıştırılır.

Name alanına kullanıcı adı girilir.

Chat Input alanına soru yazılır.

Örnek sorular:

What are the restaurant opening hours?

Do you offer vegetarian options?

What payment methods are accepted?

Her kullanıcı için mesaj geçmişi ayrı tutulur. Name alanı değiştirildiğinde yeni bir oturum başlatılır.

Teknik Ayarlar
Parametre	Değer
Chunk Size	1000
Chunk Overlap	200
Embedding Model	text-embedding-3-small
LLM Model	gpt-4o-mini
Temperature	0.18
Dosya Yapısı
.
├── rag_flow.json
├── Resturaunt_QA.pdf
└── README.md
Güvenlik Notları

API anahtarları repoya eklenmemelidir.

Astra DB token bilgileri paylaşılmamalıdır.

Credential sistemi kullanılmalıdır.

Büyük dosyalar için .gitignore yapılandırılmalıdır.
