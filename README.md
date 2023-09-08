# wragglingproject

Objective dari analisis yang dilakukan pada projek kali ini adalah untuk mengetahui:
1) 5 kategori produk paling populer setiap tahunnya
2) 5 kategori produk dengan pendapatan paling besar setiap tahunnya
3) Growth pemesanan produk untuk kategori produk berdasarkan jumlah transaksi
4) Growth pemesanan produk untuk kategori produk berdasarkan pendapatan
5) Jumlah seller untuk masing-masing kategori produk
6) Jumlah jenis produk untuk masing-masing kategori produk
7) 10 produk dengan rata-rata harga paling mahal

# Data Overview
Data yang digunakan adalah data tentang order dan  produk yang meliputi informasi mengenai:
1) Order _id		                : Kode unik order
2) Order_item_id			: Nomor sub-order pada order yang sama
3) Product_id				: Kode unik produk
4) Product_category_name_english	: Kategori produk dalam bahasa inggris
5) Seller_id				: Kode unik penjual
6) Price				: Harga produk yang dipesan
7) Order_status		            	: Status pesanan (terkirim, cancel, dsb)
8) Order_purchase_timestamp   		: Waktu pesanan

Data tersebut diperoleh dari select and join data Products_dataset, product_category_name_translation, order_items_dataset, order_dataset dari database olist yang dirujuk oleh Pacmann.id sebagai raw data untuk projek ini.

# Pengecekan data
Sebelum diolah lebih lanjut dilakukan pengecekan kualitas data, yaitu:
1) Cek missing value
   
![nomor1](https://github.com/elleferrd/wragglingproject/assets/137087598/6f5a809c-0671-42f9-9c7f-2da332681d95)

2) Cek Outlier

![nomor1](https://github.com/elleferrd/wragglingproject/assets/137087598/29d9b402-9955-4be3-9671-0a919cf13d88)

3) Cek inconsistent format

![nomor1](https://github.com/elleferrd/wragglingproject/assets/137087598/5d8fa333-bdd8-465c-958c-43173253a1c5)


4) Cek duplicate data

![nomor1](https://github.com/elleferrd/wragglingproject/assets/137087598/eb7b0745-6261-4886-9705-244f5ff43c6b)


# Tahapan Pemrosesan data
	
1) Data wraggling 
Tahap pertama yang dilakukan adalah membuka data, dan menggabungkan tabel-tabel dengan informasi yang diperlukan menjadi 1 tabel. Projek ini menggabungkan 4 data yaitu Products_dataset, product_category_name_translation, order_items_dataset, order_dataset dari database dengan menggunakan metode temporary table & left join. Tabel kunci dipilih berdasarkan data dengan jumlah row terbanyak untuk mencegah hilangnya populasi data. Penggabungan data tersebut dilakukan dengan skrip dibawah ini:

		#menyusun 2 tabel sementara & join 
		CREATE TEMPORARY TABLE produk as
		SELECT product_id, olist_products_dataset.product_category_name, product_category_name_english FROM olist_products_dataset
		LEFT JOIN product_category_name_translation
		ON olist_products_dataset.product_category_name = product_category_name_translation.product_category_name;
		
		CREATE TEMPORARY TABLE temp_orders as
		SELECT olist_order_items_dataset.order_id, order_item_id, product_id, seller_id, price, order_status, order_purchase_timestamp FROM olist_order_items_dataset
		LEFT JOIN olist_order_dataset
		ON olist_order_items_dataset.order_id = olist_order_dataset.order_id;
		
		SELECT temp_orders.order_id, order_id, temp_orders.product_id, product_category_name_english, seller_id, price, order_status, order_purchase_timestamp FROM temp_orders
		LEFT JOIN produk
		ON temp_orders.product_id = produk.product_id;

Setelah data digabung menjadi 1 tabel, data tersebut dipindahkan ke python dengan menggunakan skrip dibawah ini:

		import pandas as pd
		import numpy as np
		data = pd.read_csv("TEST/wrag.csv")

2) Data Cleaning
   
Sebelum diolah lebih lanjut, dilakukan pengecekan kualitas data, yaitu missing value, outlier, inconsistent format dan data duplikat. Dari pengecekan yang dilakukan, diketahui:

a) Terdapat missing value pada kolom ‘product_category_name_english’, sehingga dilakukan pengisian sel kosong dengan ‘unknown’.

b) Ditemukan outlier data pada variabel harga. Sehingga perhitungan rata-rata harga produk (objektif ke 7) tidak menyertakan data outlier

c) Ditemukan beberapa inconsistent format pada kolom  ‘product_category_name_english’ dan dillakukan regrouping. Terdapat beberapa inkonsistensi seperti pada kategori 'Home_appliances_2' dan 'home_appliances',  'home_confort' dan 'home_comfort_2',   'la_cuisine’ dan ‘food’, ‘signaling_and_security’, 'security_and_services', typo pada  'fashio_female_clothing'. Sehingga kategori tersebut diubah dengan menggunakan metode replace. Untuk mengatasi hal ini, tidak hanya dilakukan penyeragaman format, namun juga regrouping kategori produk karena kategori dianggap terlalu banyak. Regrouping dilakukan dengan cara membuat dataset baru dengan 2 kolom yaitu kolom ‘product_category_name_english’ dan ‘product_category’ yang diinginkan.

d) Tidak terdapat data duplikat

	Pengecekan dan penyesuaian data tersebut dilakukan dengan skrip dibawah ini:

		#cek missing value
		data.isna().sum() /data.count()
		data['product_category_name_english']=data['product_category_name_english'].fillna("unknown")
		data.isna().sum() /data.count()
		
		#cek dan handling data outlier
		import seaborn as sns
		sns.scatterplot(data = data2, x='price', y='product_category_name_english')
		#buat fungsi quantile
		def upper_quantile_1_5(group):
		    quantile_75 = group.quantile(0.75)
		    return quantile_75 * 1.5
		#menjalankan fungsi kuantil & merapihkan data
		result = data2.groupby('product_category_name_english')['price'].apply(upper_quantile_1_5)
		result = pd.DataFrame(result)
		result.reset_index(drop=False, inplace = True)
		#merge hasil acuan kuantil ke dataset
		cekoutlier = pd.merge(data2, result, on='product_category_name_english')
		#tagging outlier
		cekoutlier['cek'] = cekoutlier['price_x'] <= cekoutlier['price_y']
		
		#cek dan handling inconsistent format
		data['product_category_name_english'].unique()
		data['order_status'].unique()
		#merubah format yang inkonsisten
		replace_dict = { 'home_appliances_2':'home_appliances', 'home_confort':'home_comfort_2',  'la_cuisine': 'food', 'signaling_and_security':'security_and_services', 'fashio_female_clothing':'fashion_female_clothing'} 
		data2['product_category_name_english'].replace(replace_dict, inplace=True)  
		#import data & merge data
		kategori = pd.read_csv("TEST/kategori.csv")
		data2 = data.merge(kategori)
		
		#cek data duplikat
		data[data.duplicated()].shape[0]
		
		#Sebagai tambahan dilakukan filter data dengan status pemesanan yang telah berhasil atau telah terkirim. Dilakukan dengan skrip dibawah ini:
		data2 = data2[data2['order_status'] == 'delivered']


4) Data Manipulation

a)  Mengetahui 5 kategori produk paling populer setiap tahunnya
Dalam rangka melihat tren produk yang paling populer, pertama-tama (1) dilakukan parse date untuk melihat tren per tahun. Kemudian (2) dibuat pivot table untuk menghitung jumlah transaksi per kategori produk. Lalu (3) dilakukan sort value untuk mengetahui urutan produk paling populer. (4) kemudian data dirapihkan dengan mengubah data menjadi dataframe, merubah index dan melakukan merging data dengan fungsin concatenate. Hal tersebut dilakukan dengan skrip dibawah ini:

		#Parse Date
		from datetime import datetime
		data2['order_purchase_timestamp'] = pd.to_datetime(data2['order_purchase_timestamp'])
		data2['year'] = data2['order_purchase_timestamp'].dt.year
		data2['month'] = data2['order_purchase_timestamp'].dt.month
		
		#Mengurutkan kategori produk dengan jumlah transaksi terbanyak
		#(1) buat pivot table untuk menghitung jumlah tansaksi
		pivot_produk_popular = data2.pivot_table(values='order_id', index='product_category', columns='year',aggfunc='count')
		
		#melakukan (2) sort value, (3) ubah menjadi dataframe, (4) ubah index, (5) merge data dengan fungsi concat
		data3 = pivot_produk_popular.sort_values(by=2016, ascending=False)
		pd.DataFrame(data3)
		top_2016=pd.DataFrame(data3[2016])
		top_2016.reset_index(drop=False, inplace = True)
		
		data3 = pivot_produk_popular.sort_values(by=2017, ascending=False)
		pd.DataFrame(data3)
		top_2017=pd.DataFrame(data3[2017])
		top_2017.reset_index(drop=False, inplace = True)
		
		data3 = pivot_produk_popular.sort_values(by=2018, ascending=False)
		pd.DataFrame(data3)
		top_2018=pd.DataFrame(data3[2018])
		top_2018.reset_index(drop=False, inplace = True)
		
		Datatren = pd.concat([top_2016, top_2017, top_2018], axis=1)
	
Hasil:

![nomor1](https://github.com/elleferrd/wragglingproject/assets/137087598/93fc39e7-25ab-4613-8268-68c37dbe4a45)


b) Mengetahui 5 kategori produk dengan pendapatan paling besar setiap tahunnya
Cara yang digunakan sama dengan poin 3a) untuk mengetahui 5 kategori produk paling populer setiap tahunnya. Perbedaannya adalah pemilihan values pada pivot dan data yang dirujuk. Berikut adalah skrip yang digunakan:
		#membuat pivot tabel & mengurutkan dari yang paling tinngi
		pivot_produk_penjualan = data2.pivot_table(values='price', index='product_category', columns='year',aggfunc='sum')
		data4 = pivot_produk_penjualan.sort_values(by=2016, ascending=False)
		pd.DataFrame(data4)
		#memisahkan pada per tahun untuk diurutkan kembali
		top_2016=pd.DataFrame(data4[2016])
		top_2016.reset_index(drop=False, inplace = True)
		data4 = pivot_produk_penjualan.sort_values(by=2017, ascending=False)
		pd.DataFrame(data4)
		top_2017=pd.DataFrame(data4[2017])
		top_2017.reset_index(drop=False, inplace = True)
		data4 = pivot_produk_penjualan.sort_values(by=2018, ascending=False)
		pd.DataFrame(data4)
		top_2018=pd.DataFrame(data4[2018])
		top_2018.reset_index(drop=False, inplace = True)
		#join kembali
		datapenjualan = pd.concat([top_2016, top_2017, top_2018], axis=1)

Hasil:

![nomor1](https://github.com/elleferrd/wragglingproject/assets/137087598/464401be-3f4e-437d-baf8-36e98ca0c0df)


c) Growth pemesanan produk untuk kategori produk berdasarkan jumlah pesanan
Pertumbuhan pemesanan produk dihitung dengan memasukan rumus “(jumlah pesanan tahun n dikurangi  jumlah pesanan tahun n-1)/jumlah pesanan tahun n-1” yang merujuk data pivot produk yang telah dibuat pada poin a). Berikut adalah skrip yang digunakan:
		pivot_produk_popular['growth_2017']=(pivot_produk_popular[2017]-pivot_produk_popular[2016])/pivot_produk_popular[2016]
		pivot_produk_popular['growth_2018']=(pivot_produk_popular[2018]-pivot_produk_popular[2017])/pivot_produk_popular[2017]
		pivot_produk_popular

Hasil:

![data1](https://github.com/elleferrd/wragglingproject/assets/137087598/7e5fb27d-ce1f-4348-bcc0-ed3aa3b5bd2f)


d) Growth pemesanan produk untuk kategori produk berdasarkan jumlah pendapatan
Pertumbuhan penjualan produk dihitung dengan memasukan rumus “(jumlah nilai penjualan tahun n dikurangi  jumlah nilai penjualan tahun n-1)/jumlah nilai pesanan tahun n-1” yang merujuk data pivot produk yang telah dibuat pada poin b). Berikut adalah skrip yang digunakan:
		pivot_produk_penjualan['growth_2017']=(pivot_produk_penjualan[2017]-pivot_produk_penjualan[2016])/pivot_produk_penjualan[2016]
		pivot_produk_penjualan['growth_2018']=(pivot_produk_penjualan[2018]-pivot_produk_penjualan[2017])/pivot_produk_penjualan[2017]
		pivot_produk_penjualan

Hasil:
![data1](https://github.com/elleferrd/wragglingproject/assets/137087598/f57b521a-35d1-4e2b-885b-0b9859927229)

e) Jumlah seller untuk masing-masing kategori produk
Dalam rangka menghitung jumlah seller untuk masing-masing kategori produk, pertama-tama (1) dilakukan perhitungan jumlah seller per kategori produk dan tahun dengan menggunakan metode groupby dan nuninque. Kemudian (2) dibuat pivot table untuk merubah tahun menjadi kolom untuk mempermudah pembaca dalam melihat data. Kemudian, (3) dihitung growth per seller sebagai informasi tambahan yang dapat dipertimbangkan dan (4) dilakukan pengurutan data dari jumlah seller terbanyak pada tahun 2018. Berikut adalah skrip yang digunakan:

		data_seller =data2.groupby(['product_category','year'])['seller_id'].nunique().reset_index()
		pivot_seller = data_seller.pivot_table(values='seller_id', index='product_category', columns='year',aggfunc='sum')
		pivot_seller['growth_2017']=(pivot_seller[2017]-pivot_seller[2016])/pivot_seller[2016]
		pivot_seller['growth_2018']=(pivot_seller[2018]-pivot_seller[2017])/pivot_seller[2017]
		pivot_seller = pivot_seller.sort_values(by=2018, ascending=False)
		pivot_seller

	Hasil:

![data1](https://github.com/elleferrd/wragglingproject/assets/137087598/6d55e81a-c6e8-4a8a-8640-62d5341ab3a8)


f) Jumlah jenis produk untuk masing-masing kategori produk
Pengolahan data untuk mengetahui jumlah jenis produk untuk masing-masing kategori produk sama dengan cara pengolahan data untuk mengetahui jumlah seller untuk masing-masing kategori produk, namun rujukan data dibedakan. Berikut adalah skrip yang digunakan:

		data_product =data2.groupby(['product_category','year'])['product_id'].nunique().reset_index()
		pivot_product = data_product.pivot_table(values='product_id', index='product_category', columns='year',aggfunc='sum')
		pivot_product['growth_2017']=(pivot_product[2017]-pivot_product[2016])/pivot_product[2016]
		pivot_product['growth_2018']=(pivot_product[2018]-pivot_product[2017])/pivot_product[2017]
		pivot_product = pivot_product.sort_values(by=2018, ascending=False)
		pivot_product

	Hasil:

 ![data1](https://github.com/elleferrd/wragglingproject/assets/137087598/bceb54bb-924d-4bcf-bfcc-b76f0498a73b)


g) Produk dengan harga rata-rata paling mahal
Pengolahan rata-rata harga produk tidak menyertakan harga produk yang terlalu tinggi, Sehingga, langkah pertama yang dilakukan adalah filter data yang bukan outlier, kemudian dilakukan perhitungan rata-rata(mean) per produk dan sort values. Dan yang terakhir filter 10 data teratas. Skrip yang digunakan adalah sebagai berikut: 
	
		cekoutlier =cekoutlier[cekoutlier['cek'] == True]
		rerataharga = cekoutlier.groupby('product_category_name_english')['price_x'].mean()
		rerataharga = pd.DataFrame(rerataharga).sort_values(by ='price_x', ascending =False)
		rerataharga =rerataharga.head(10)

	Hasil:

![data1](https://github.com/elleferrd/wragglingproject/assets/137087598/da5f5415-2350-4da2-b579-f00c8894e187)



# Hasil analisis

1) Mengetahui 5 kategori produk paling populer setiap tahunnya
Home appliances merupakan produk paling populer pada tahun 2016-2018. Selain dari itu kosmetik, produk hobi, elekronik dan kategori lainnya selalu menjadi top 5 produk paling populer dengan urutan yang berbeda-beda setiap tahunnya. Kosmetik menempati posisi kedua paling populer pada tahun 2016, keempat pada tahun 2017, ketiga pada tahun 2028. Hobi menempati posisi ketiga pada tahun 2016, dan kedua pada tahun 2017 dan 2018. Kategori lain-lain menempati posisi keempat pada tahun 2016, ketiga pada tahun 2017 dan kelima pada tahun 2018. Elektronik menempati posisi kelima pada tahun 2016 dan 2017, serta posisi keempat pada tahun 2018. Sehingga dapat disimpulkan bahwa popularitas kategori produk hobi dan elektronik mengalami peningkatan secara relatif.

2) Mengetahui 5 kategori produk paling populer setiap tahunnya
Produk hobi, home appliances, kosmetik, lainnya, dan elektronik merupakan top 5 kategori produk dengan nilai penjualan paling tinggi pada tahun 2016 dan 2017. Sedangkan pada 2018, produk fashion menggantikan produk elektronik. Urutan produk penyumbang penjualan terbanyak pada tahun 2016 adalah (1) hobi, (2) kosmetik, (3) home appliances, (4) lainnya, (5) elektronik. Pada tahun 2017 adalah (1) home appliances, (2) hobi, (3) lainnya, (4) kosmetik, (5) elektronik. Pada tahun 2018 adalah (1) home appliances, (2) hobi, (3) kosmetik, (4) fashion, (6) lainnya.

3) Growth pemesanan produk untuk kategori produk berdasarkan jumlah pesanan
Top 5 growth pemesanan tertinggi pada tahun 2017 adalah (1) f&b, (2) office supplies, (3) fashion, (4) electronics, (5) automotives. Pada tahun 2018 adalah (1) konstruksi, (2) f&b, (3) otomotif, (4) electronics, (5) cosmetics. Mayoritas kategori produk dengan growth yang relatif tertinggi bukan merupak produk tepopular kecuali elektronik dan kosmetik. Hal tersebut dapat dikarenan basis pemesanan yang masih relatif rendah sehingga lebih mudah untuk bertumbuh. Pada saat yang sama elektronik merupakan produk yang potensial

4) Growth pemesanan produk untuk kategori produk berdasarkan jumlah pendapatan
Top 5 growth pemesanan tertinggi pada tahun 2017 adalah (1)elektronik, (2) f&b, (3) otomotif, (4) others, (5) fashion, pada tahun 2018 adalah (1) konstruksi, (2) f&b, (3) automotives, (4) kosmetik, (5) fashion. Sebagai tambahan diketahui bahwa terjadi penurunan pesanan pada kategori others pada tahun 2018.

5) Jumlah seller untuk masing-masing kategori produk
Dari hasil pengolahan top 5 seller terbanyak banyak adalah (1) home apliances diikuti oleh (2) hobbies, (3) others, (4) kosmetik, dan (5) elektronik. Seluruhnya termasuk dalam jajaran 5 kategori produk yang paling sering dibeli. Sedangkan top 5 growth jumlah seller pada tahun 2017 adalah (1) office, (2) f&b, (3) others, (4) automotives, dan (5) electronics dimana 4 antaranya termasuk dalam 5 produk dengan growth tertinggi. Pada tahun 2028, top 5 growth jumlah seller adalah (1) konstruksi, (2) f&b, (3) kosmetik, (4) office, (5) automotives dimana 4 antaranya termasuk dalam 5 produk dengan growth tertinggi.  

6) Jumlah jenis produk untuk masing-masing kategori produk
Dari hasil pengolahan top 5 produk terbanyak banyak pada tahun 2016 adalah (1) home appliances, (2) hobbies, (3) cosmetics, (4) others, (5) fashion. Pada tahun 2017 adalah (1) home appliances, (2) hobbies, (3) others, (4) fashion, (5) cosmetics. Pada tahun 2018 adalah (1) home appliances, (2) hobbies, (3) cosmetics, (4) others, (5) electronics yang mana minimal 4 diantaranya merupakan produk yang paling banyak dipesan pada tahun bersangkutan.
edangkan top 5 growth jumlah produk pada tahun 2017 adalah (1) office, (2) f&b, (3) fashion, (4) electronics, (5) automotives, dimana seluruhnya termasuk dalam 5 produk dengan growth tertinggi. Pada tahun 2028, top 5 growth jumlah seller adalah (1) konstruksi, (2) otomotif, (3) f&b, (4) kosmetik, (5) office, dimana 4 antaranya termasuk dalam 5 produk dengan growth tertinggi.  

7) Produk dengan harga rata-rata paling mahal
Produk dengna harga rata-rata paling mahal adalah komputer, oven dan mesin kopi, industri agro, furnitur kamar, alat keselamatan konstruksi, furnitur kantor, ac, alat musik,  alat lampu konstruksi, dan jam tangan.

Kesimpulan yang dapat ditarik dari pengolahan data tersebut adalah semakin banyak jumlah seller dan variasi barang, maka akan semakin banyak pula transaksi. Sehingga, aksi bisnis yang direkomendasikan adalah menarik lebih banyak seller yang memiliki variasi barang yang banyak dan beragam, terutama untuk produk-produk yang memiliki nilai jual tinggi seperti komputer, oven dan mesiun kopi, industri agro dan furnitur.

# Respiratory & visualisasi data


![produk popular](https://github.com/elleferrd/wragglingproject/assets/137087598/6a3cf08d-0be2-4a76-aa04-9247d14c31bb)

![penjualan](https://github.com/elleferrd/wragglingproject/assets/137087598/aa06d246-c9fa-4e10-b307-63fd9027d10d)


![pie chart](https://github.com/elleferrd/wragglingproject/assets/137087598/cc74f889-c1fa-485b-9767-111b32562857)


#Skrip grafik pertama
		#Sample data for the bar plots
		categories = pivot_produk_popular['product_category']
		values1 = pivot_produk_popular[2016]
		values2 = pivot_produk_popular[2017]
		values3 =  pivot_produk_popular[2018]
		#Sort the data for each bar plot in descending order
		sorted_indices1 = np.argsort(values1)[::-1]
		sorted_categories1 = [categories[i] for i in sorted_indices1]
		sorted_values1 = [values1[i] for i in sorted_indices1]
		sorted_indices2 = np.argsort(values2)[::-1]
		sorted_categories2 = [categories[i] for i in sorted_indices2]
		sorted_values2 = [values2[i] for i in sorted_indices2]
		sorted_indices3 = np.argsort(values3)[::-1]
		sorted_categories3 = [categories[i] for i in sorted_indices3]
		sorted_values3 = [values3[i] for i in sorted_indices3]
		#Create subplots with three columns
		fig, axs = plt.subplots(1, 3, figsize=(15, 5))
		#Plot the first sorted bar plot
		axs[0].bar(sorted_categories1, sorted_values1, color='skyblue')
		axs[0].set_title('2016')
		#Plot the second sorted bar plot
		axs[1].bar(sorted_categories2, sorted_values2, color='salmon')
		axs[1].set_title('2017')
		#Plot the third sorted bar plot
		axs[2].bar(sorted_categories3, sorted_values3, color='lightgreen')
		axs[2].set_title('2018)')
		#Rotate x-axis labels for better readability (optional)
		for ax in axs:
		    ax.set_xticklabels(ax.get_xticklabels(), rotation=45, ha='right')
		#Add labels and titles
		for ax in axs:
		    ax.set_xlabel('Categories')
		    ax.set_ylabel('Values')
		#Adjust the layout to prevent overlapping
		plt.tight_layout()
		#Display the plots
		plt.show()

Skrip grafik kedua:
		#Sample data for the bar plots
		categories = pivot_produk_penjualan['product_category']
		values1 = pivot_produk_penjualan[2016]
		values2 = pivot_produk_penjualan[2017]
		values3 =  pivot_produk_penjualan[2018]
		#Sort the data for each bar plot in descending order
		sorted_indices1 = np.argsort(values1)[::-1]
		sorted_categories1 = [categories[i] for i in sorted_indices1]
		sorted_values1 = [values1[i] for i in sorted_indices1]
		sorted_indices2 = np.argsort(values2)[::-1]
		sorted_categories2 = [categories[i] for i in sorted_indices2]
		sorted_values2 = [values2[i] for i in sorted_indices2]
		sorted_indices3 = np.argsort(values3)[::-1]
		sorted_categories3 = [categories[i] for i in sorted_indices3]
		sorted_values3 = [values3[i] for i in sorted_indices3]
		#Create subplots with three columns
		fig, axs = plt.subplots(1, 3, figsize=(15, 5))
		#Plot the first sorted bar plot
		axs[0].bar(sorted_categories1, sorted_values1, color='skyblue')
		axs[0].set_title('2016')
		#Plot the second sorted bar plot
		axs[1].bar(sorted_categories2, sorted_values2, color='salmon')
		axs[1].set_title('2017')
		#Plot the third sorted bar plot
		axs[2].bar(sorted_categories3, sorted_values3, color='lightgreen')
		axs[2].set_title('2018)')
		#Rotate x-axis labels for better readability (optional)
		for ax in axs:
		    ax.set_xticklabels(ax.get_xticklabels(), rotation=45, ha='right')
		#Add labels and titles
		for ax in axs:
		    ax.set_xlabel('Categories')
		    ax.set_ylabel('Values')
		#Adjust the layout to prevent overlapping
		plt.tight_layout()
		#Display the plots
		plt.show()

Skrip grafik ketiga:
		#mengisi sel kosong
		pivot_seller.reset_index(drop=False, inplace = True)
		pivot_product.reset_index(drop=False, inplace = True)
		pivot_seller = pivot_seller.fillna(0)
		pivot_product= pivot_product.fillna(0)
		import matplotlib.pyplot as plt
		import numpy as np
		#Sample data for the bar plots
		x = pivot_seller['product_category']
		y1 = pivot_seller[2016]
		y2 = pivot_seller[2017]
		y3 = pivot_seller[2018]
		#Create a figure with three subplots (one for each pie chart)
		fig, axs = plt.subplots(1, 3, figsize=(15, 5))
		#Create the first pie chart using 'Value1'
		axs[0].pie(pivot_seller[2016], labels=pivot_seller['product_category'], autopct='%1.1f%%')
		axs[0].set_title('2016')
		#Create the second pie chart using 'Value2'
		axs[1].pie(pivot_seller[2017], labels=pivot_seller['product_category'], autopct='%1.1f%%')
		axs[1].set_title('2017')
		#Create the third pie chart using 'Value3'
		axs[2].pie(pivot_seller[2018], labels=pivot_seller['product_category'], autopct='%1.1f%%')
		axs[2].set_title('2018')
		#Adjust the layout and display the charts
		plt.tight_layout()
		plt.show()

