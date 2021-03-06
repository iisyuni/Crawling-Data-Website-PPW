# Crawling-Data-Website-PPW
------

# Web Crawling Menggunakan Python

> Nama : Iis Yuni Harianti
>
> NIM : 160411100086
>
> Semester : 6

> Mata kuliah : Pengembangan dan Pencarian Web
>
> Dosen Pengampu : Mulaab, S.Si., M.Kom

------

#### Library yang di  butuhkan untuk mengcrawl

> untuk memulai melakukan crawl pastikan python yang digunakan sudah memiliki library sebagai berikut :
>
> 1. BeautifulSoup4
> 2. requests
> 3. numpy
> 4. scipy
> 5. scikit-learn

```python
from math import log10
import requests
from bs4 import BeautifulSoup
import sqlite3
from Sastrawi.StopWordRemover.StopWordRemoverFactory import StopWordRemoverFactory
from Sastrawi.Stemmer.StemmerFactory import StemmerFactory
import csv
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score
import numpy as np
import warnings
```

> Web Crawler digunakan untuk mengambil data berupa teks, audio, foto bahkan video pada sebuah website.

#### source code untuk mengcrawl data 

> Proses crawling dalam suatu website dimulai dari mendata seluruh url dari website, menelusurinya satu-persatu, kemudian memasukkannya dalam daftar halaman pada indeks search engine, sehingga setiap kali ada perubahan pada website, akan terupdate secara otomatis.

```python
src = "https://www.kompasiana.com/olahraga/"

page = requests.get(src)
soup = BeautifulSoup(page.content, 'html.parser')

artikel = soup.findAll(class_='title mt40')

koneksi = sqlite3.connect('db_data.db')
koneksi.execute(''' CREATE TABLE if not exists kompasiana
            (judul TEXT NOT NULL,
             isi TEXT NOT NULL);''')

for i in range(len(artikel)):
    link = artikel[i].find('a')['href']
    page = requests.get(link)
    soup = BeautifulSoup(page.content, 'html.parser')
    judul = soup.find(class_='title').getText()
    isi = soup.find(class_='read-content col-lg-9 col-md-9 col-sm-9 col-xs-9')
    paragraf = isi.findAll('p')
    p = ''
    for s in paragraf:
        p+=s.getText() +' '
    cek = koneksi.execute("SELECT * FROM kompasiana where judul=?", (judul,))
    cek = cek.fetchall()
    if len(cek) == 0 :
        koneksi.execute('INSERT INTO kompasiana values (?,?)', (judul, p));

koneksi.commit()
tampil = koneksi.execute("SELECT * FROM kompasiana")
with open ('data_crawler.csv', mode='w')as employee_file :
    employee_writer = csv.writer(employee_file, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)
    for i in tampil:
        employee_writer.writerow(i)
        
tampil = koneksi.execute("SELECT * FROM kompasiana")
isi = []
for row in tampil:
    isi.append(row[1])
    #print(row)
print("crawl")
```

> Data yang telah di crawl kemudian di simpan ke dalam database yaitu SQlite

#### source code untuk VSM

> **Vector Space Model (VSM)** digunakan sebagai representasi dari kumpulan dataset dokumen teks. Dokumen dalam **Vector Space Model (VSM)** berupa matriks yang berisi bobot seluruh kata pada tiap dokumen. Bobot tersebut menyatakan kepentingan atau kontribusi kata terhadap suatu dokumen dan kumpulan dokumen.

```python
factory = StopWordRemoverFactory()
stopword = factory.create_stop_word_remover ()

factory = StemmerFactory ()
stemmer = factory.create_stemmer ()

tmp = ''
for i in isi:
    tmp = tmp + ' ' +i

hasil = []
for i in tmp.split():
    try :
        if i.isalpha() and (not i in hasil) and len(i)>1:
            # Menghilangkan Kata tidak penting
            stop = stopword.remove(i)
            if stop != "":
                stem = stemmer.stem(stop)
                hasil.append(stem)
    except:
        continue
katadasar=np.array(hasil)
print("vsm")
```

#### source code untuk menyeleksi kata dasar agar sesuai dengan KBBI

```python
#KBBI
koneksi = sqlite3.connect('KBI.db')
cur_kbi = koneksi.execute("SELECT* from KATA")
    
def LinearSearch (kbi,kata):
    found=False
    posisi=0
    while posisi < len (kata) and not found :
        if kata[posisi]==kbi:
            found=True
        posisi=posisi+1
    return found
berhasil=[]
for kata in cur_kbi :
    ketemu=LinearSearch(kata[0],katadasar)
    if ketemu :
        kata = kata[0]
        berhasil.append(kata)
print(berhasil)
katadasar = np.array(berhasil)

matrix = []
for row in isi :
    tamp_isi=[]
    for a in katadasar:
        tamp_isi.append(row.lower().count(a))
    matrix.append(tamp_isi)
print("matrix")

with open ('data_matrix.csv', mode='w')as employee_file :
    employee_writer = csv.writer(employee_file, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)
    employee_writer.writerow(katadasar)
    for i in matrix :
        employee_writer.writerow(i)
```

> kata yang sudah terseleksi kemudian di simpan ke dalam csv dalam bentuk matrix

#### source code tf-idf

> **Tf** menyatakan jumlah berapa banyak keberadaan suatu kata dalam satu dokumen. **IDF (Inverse Document Frequency)** merupakan sebuah perhitungan dari bagaimana kata didistribusikan secara luas pada koleksi dokumen yang bersangkutan.**IDF** menunjukkan hubungan ketersediaan sebuah kata dalam seluruh dokumen. **Semakin sedikit jumlah dokumen yang mengandung kata yang dimaksud, maka nilai IDF semakin besar.**

```python
df = list()
for d in range (len(matrix[0])):
    total = 0
    for i in range(len(matrix)):
        if matrix[i][d] !=0:
            total += 1
    df.append(total)

idf = list()
for i in df:
    tmp = 1 + log10(len(matrix)/(1+i))
    idf.append(tmp)

tf = matrix
tfidf = []
for baris in range(len(matrix)):
    tampungBaris = []
    for kolom in range(len(matrix[0])):
        tmp = tf[baris][kolom] * idf[kolom]
        tampungBaris.append(tmp)
    tfidf.append(tampungBaris)
tfidf = np.array(tfidf)
print("tf_idf")

with open('tf-idf.csv',  mode='w') as employee_file:
    employee_writer = csv.writer(employee_file, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)
    employee_writer.writerow(katadasar)
    for i in tfidf:
        employee_writer.writerow(i)

```

#### source code untuk seleksi fitur, clustering dan silhouette

> **Seleksi fitur** adalah salah satu tahapan praproses klasifikasi. **Seleksi fitur**dilakukan dengan cara memilih **fitur**-**fitur** yang relevan yang mempengaruhi hasil klasifikasi. **Seleksi fitur** digunakan untuk mengurangi dimensi data dan **fitur**-**fitur**yang tidak relevan.

> **Clustering** adalah metode penganalisaan data, yang sering dimasukkan sebagai salah satu metode Data Mining, yang tujuannya adalah untuk mengelompokkan data dengan karakteristik yang sama ke suatu 'wilayah' yang sama dan data dengan karakteristik yang berbeda ke 'wilayah' yang lain.

> 

```python
def pearsonCalculate(data, u,v):
    "i, j is an index"
    atas=0; bawah_kiri=0; bawah_kanan = 0
    for k in range(len(data)):
        atas += (data[k,u] - meanFitur[u]) * (data[k,v] - meanFitur[v])
        bawah_kiri += (data[k,u] - meanFitur[u])**2
        bawah_kanan += (data[k,v] - meanFitur[v])**2
    bawah_kiri = bawah_kiri ** 0.5
    bawah_kanan = bawah_kanan ** 0.5
    return atas/(bawah_kiri * bawah_kanan)
def meanF(data):
    meanFitur=[]
    for i in range(len(data[0])):
        meanFitur.append(sum(data[:,i])/len(data))
    return np.array(meanFitur)
def seleksiFiturPearson(katadasar, data, threshold):
    global meanFitur
    meanFitur = meanF(data)
    u=0
    while u < len(data[0]):
        dataBaru=data[:, :u+1]
        meanBaru=meanFitur[:u+1]
        katadasarBaru=katadasar[:u+1]
        v = u
        while v < len(data[0]):
            if u != v:
                value = pearsonCalculate(data, u,v)
                if value < threshold:
                    dataBaru = np.hstack((dataBaru, data[:, v].reshape(data.shape[0],1)))
                    meanBaru = np.hstack((meanBaru, meanFitur[v]))
                    katadasarBaru = np.hstack((katadasarBaru, katadasar[v]))
                    
            v+=1
        data = dataBaru
        meanFitur=meanBaru
        katadasar = katadasarBaru
        if u%50 == 0 : print("proses : ", u, data.shape)
        u+=1
    return katadasar, data

katadasarBaru, fiturBaru = seleksiFiturPearson(katadasar, tfidf, 0.8)

for i in range(2, len(fiturBaru)-1):
    
    kmeans = KMeans(n_clusters=i, random_state=0).fit(fiturBaru);
    
    classnya = kmeans.labels_
    s_avg = silhouette_score(fiturBaru, classnya, random_state=0)
    
    print("Silhouette untuk", i, "cluster adalah",s_avg)
    print(kmeans.labels_)

print("proses selesai")
with open('Anggota_cluster.csv', newline='', mode='w') as employee_file:
    employee_writer = csv.writer(employee_file, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)
    for i in classnya.reshape(-1,1):
        employee_writer.writerow(i)

with open('Seleksi_Fitur.csv', newline='', mode='w') as employee_file:
    employee_writer = csv.writer(employee_file, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)
    employee_writer.writerow([katadasarBaru.tolist()])
    for i in fiturBaru:
        employee_writer.writerow(i)
```

### Note: 

- Untuk menjalankan programnya pastikan terhubung dengan koneksi internet. Agar tidak terjadi error.
- Untuk database KBBI (KBI.db) pastikan tersimpan dalam satu folder dengan file programnya crawling.py
