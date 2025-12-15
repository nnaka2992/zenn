---
title: "PostgreSQLのインデックス作成におけるパラメータの影響の調査"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PostgreSQL", "Database"]
published: true
---

<!-- ignore-start -->
このブログは[3-shake Advent Calendar 2025](https://qiita.com/advent-calendar/2025/3-shake) および[PostgreSQL Advent Calendar 2025](https://qiita.com/advent-calendar/2025/postgresql)のクロスポストです。
<!-- ignore-end -->

---
PostgreSQLのインデックス作成のパフォーマンスには下記の2つのパラメータが特に大きく影響する。

- maintenance_work_mem
<!-- ignore-start -->
  > ほとんどのインデックスメソッドにおいて、インデックス作成速度はmaintenance_work_memの設定に依存します。 より大きな値を設定すると、インデックス作成に必要となる時間が短縮されます。 ただし、実際に使用できるメモリ量を超えるほど大きくすると、マシンがスワップ状態になり、遅くなります。

- max_parallel_maintenance_workers
<!-- ignore-start -->
  > パラレルインデックス作成ではmaintenance_work_memを増やすことで、同様の逐次インデックス作成ではほとんど恩恵がみられない場合でも恩恵があるかもしれません。 パラレルワーカーはmaintenance_work_mem全体の内、少なくとも32MBの割り当て分を持たなければならないため、maintenance_work_memは要求されるワーカープロセス数に影響を及ぼすかもしれないことに注意してください。 また、リーダープロセスに対しても32MBの割り当てを残さなければなりません。 max_parallel_maintenance_workersを増やすことで、より多くのワーカーが使用できるようになるかもしれません。これは、インデックス作成が既にI/Oバウンドであるのでない限り、インデックス作成の所要時間を減らすでしょう。 もちろん、休止している十分なCPU容量もある前提です。
<!-- ignore-end -->

ここでは比較的よく使用される下記の4パターンのインデックス作成においてそれぞれのパラメータの影響を調査する。

- btreeインデックス
- hashインデックス
- GIN(pg_bigm)インデックス
- GIN(pg_trgm)インデックス

:::message
**TL;DR**
1. 基本的に`maintenance_work_mem`は1-2GB程度でインデックス作成の効率が最大化される。
2. `max_parallel_maintenance_workers`はドキュメントにあるとおり、btreeインデックスでのみ有効。
3. btreeインデックスの作成時は`max_parallel_maintenance_workers`に合わせて、`maintenance_work_mem`を調整することでパフォーマンスを最大化できる。
:::

## 実験計画

### btree インデックス

generate_seriesを利用して作成したテーブルのUUIDv4カラムに対するインデックス作成時間を計測する。
対象レコード数: 5億件
変数とするパラメータは`maintenance_work_mem`と`max_parallel_maintenance_workers`の2つとする。
テストシナリオは下記の3パターンで検証する。
1. `maintenance_work_mem`のみ変更する
   インデックス作成時間に対する影響を検証する
2. `max_parallel_maintenance_workers`のみ変更する
   インデックス作成時間に対する影響を検証する
3. `maintenance_work_mem`と`max_parallel_maintenance_workers`を変更する
   インデックス作成時間に対する影響を検証する

### hash インデックス

generate_seriesを利用して作成したテーブルのUUIDv4カラムに対するインデックス作成時間を計測する。
対象レコード数: 5億件
変数とするパラメータは`maintenance_work_mem`と`max_parallel_maintenance_workers`の2つとする。
テストシナリオは下記の2パターンで検証する。
1. `maintenance_work_mem`のみ変更する
   インデックス作成時間に対する影響を検証する
2. `max_parallel_maintenance_workers`のみ変更する
   インデックス作成時間に影響しないことを検証する

### GIN インデックス

日本語Wikipediaのダンプデータに対するインデックス作成時間を計測する。
対象レコード数: 約1.54億件
変数とするパラメータは`maintenance_work_mem`と`max_parallel_maintenance_workers`の2つとする。
テストシナリオは下記の2パターンで検証する。
1. `maintenance_work_mem`のみ変更する
   インデックス作成時間に対する影響を検証する
2. `max_parallel_maintenance_workers`のみ変更する
   インデックス作成時間に影響しないことを検証する

## 検証環境設定
<!-- ignore-start -->
### ホストマシン (Proxmox)
| 項目 | 値 |
|------|-----|
| Proxmox VE | 8.3.2 (kernel: 6.8.12-5-pve) |
| CPU | AMD Ryzen 9 6900HX (8C/16T) |
| RAM | 60GB |
| Storage | KINGSTON OM8PGP41024Q 953.9GB NVMe |
| Storage Backend | LVM-thin (local-lvm) |

### 仮想マシン
| 項目 | 値 |
|------|-----|
| vCPU | 12 (6 cores × 2 sockets) |
| RAM | 32GB (balloon無効) |
| Disk | 300GB virtio-scsi-single (iothread有効, SSD) |
| OS | Debian 12 (bookworm) |

### PostgreSQL
| 項目 | 値 |
|------|-----|
| Version | 17.7 |
| pg_bigm | 1.2 |
| pg_trgm | 1.6 |
| shared_buffers | 128MB |
| effective_cache_size | 4GB |
| work_mem | 4MB |
| maintenance_work_mem | 64MB (検証時に変更) |
| max_parallel_maintenance_workers | 2 (検証時に変更) |
| max_parallel_workers | 8 |
| max_worker_processes | 24 |

### 検証データ
| テーブル | レコード数 | サイズ |
|----------|-----------|--------|
| uuids | 5億件 | 42GB |
| article_contents | 約1.55億件 | 31GB |
| article_metadata | 約303万件 | 464MB |
<!-- ignore-end -->

## 検証

:::details `maintenance_work_mem`の検証
```
./measure_mem.sh --database postgres --table uuids --column uuidv4 --index-type btree
./measure_mem.sh --database postgres --table uuids --column uuidv4 --index-type hash
./measure_mem.sh --database postgres --table uuids --column uuidv4
./measure_mem.sh --database postgres --table article_contents --column content  --index-type gin_bigm
./measure_mem.sh --database postgres --table article_contents --column content  --index-type gin_trgm
```
:::

:::details `max_parallel_maintenance_workers`の検証
```
./measure_worker.sh --database postgres --table uuids --column uuidv4 --index-type btree
./measure_worker.sh --database postgres --table uuids --column uuidv4 --index-type hash
./measure_worker.sh --database postgres --table uuids --column uuidv4
./measure_worker.sh --database postgres --table article_contents --column content  --index-type gin_bigm
./measure_worker.sh --database postgres --table article_contents --column content  --index-type gin_trgm
```
:::

:::details `maintenance_work_mem`と`max_parallel_maintenance_workers`の検証
```
./measure_worker_and_mem.sh --database postgres --table uuids --column uuidv4 --index-type btree
```
:::

## 検証結果
### `maintenance_work_mem`の検証

インデックス種別により最もパフォーマンスにすぐれるメモリの設定は異なるものの、おおむね1-2GB程度でインデックス作成のパフォーマンスの最大化が確認できる。
また、2GBを越えるとパフォーマンスが劣化し始める傾向にある。
![](/images/performance_measurement_on_pg_index_creation/index_creation_mem.png)

:::details raw data
1カラム目は割り当てたRAMの容量で、2カラム目はduration(ms)。
- btree
  ```
  memory_setting,duration_seconds
  64MB,215.436
  128MB,187.092
  256MB,186.141
  512MB,192.46
  1GB,179.475
  2GB,170.642
  4GB,183.122
  8GB,161.792
  16GB,172.704
  24GB,141.884
  ```
- hash
  ```
  memory_setting,duration_seconds
  64MB,512.628
  128MB,492.638
  256MB,489.282
  512MB,495.789
  1GB,494.966
  2GB,482.46
  4GB,479.897
  8GB,477.335
  16GB,485.344
  24GB,524.438
  ```
- bigm
  ```
  memory_setting,duration_seconds
  64MB,3173.16
  128MB,2789.47
  256MB,2554.69
  512MB,2473.48
  1GB,2455.88
  2GB,2506.91
  4GB,2540.37
  8GB,2707.35
  16GB,2961.47
  24GB,3121.17
  ```

- trgm
  ```
  memory_setting,duration_seconds
  64MB,6107.61
  128MB,4723.11
  256MB,3822.1
  512MB,3155.58
  1GB,2817.15
  2GB,2702.52
  4GB,2736.89
  8GB,2877.32
  16GB,2921.72
  24GB,3125.49
  ```
:::

### `max_parallel_maintenance_workers`の検証

公式のドキュメントにあるとおり、btreeインデックス以外ではパフォーマンスに影響しないことがわかる。
また割り当てるメモリが1GBの場合、1以上の値を設定しても劇的な改善は見込めない。
![](/images/performance_measurement_on_pg_index_creation/index_creation_worker.png)

::: details raw data
- bigm
  ```
  parallel_workers,duration_seconds
  0,3068.29
  1,3020.13
  2,3039.78
  4,3051.23
  8,3044.98
  16,3055.48
  24,3044.51
  ```

- btree
  ```
  parallel_workers,duration_seconds
  0,296.099
  1,201.215
  2,204.896
  4,194.001
  8,197.582
  16,190.474
  24,190.682
  ```

- hash
  ```
  parallel_workers,duration_seconds
  0,491.499
  1,504.584
  2,492.4
  4,495.359
  8,493.355
  16,488.513
  24,494.994
  ```

- trgm
  ```
  parallel_workers,duration_seconds
  0,5919.78
  1,5921.21
  2,5902.11
  4,5902.26
  8,5870.49
  16,5946.31
  24,5978.33
  ```
:::

### `maintenance_work_mem`と`max_parallel_maintenance_workers`の検証

割り当てるメモリの容量に比べ、ワーカ数のほうがパフォーマンスへの影響が強い。
一方でワーカ数にを大きくしたときに、最大のパフォーマンスを得るには適切なメモリ設定が必要になる。
![](/images/performance_measurement_on_pg_index_creation/index_creation_worker_and_mem.png)

::: details raw data
```
maintenance_work_mem,parallel_workers,duration_seconds
64MB,0,311.977
64MB,1,205.422
64MB,2,198.177
64MB,4,200.563
64MB,8,201.57
64MB,16,190.196
64MB,24,203.228
128MB,0,270.598
128MB,1,188.438
128MB,2,163.977
128MB,4,139.589
128MB,8,136.348
128MB,16,148.212
128MB,24,141.687
256MB,0,264.638
256MB,1,178.706
256MB,2,148.237
256MB,4,138.937
256MB,8,111.936
256MB,16,114.122
256MB,24,113.955
512MB,0,256.302
512MB,1,185.003
512MB,2,165.687
512MB,4,128.459
512MB,8,110.218
512MB,16,111.761
512MB,24,111.019
1GB,0,241.457
1GB,1,176.298
1GB,2,139.059
1GB,4,130.203
1GB,8,114.306
1GB,16,111.007
1GB,24,114.169
2GB,0,248.76
2GB,1,172.654
2GB,2,147.458
2GB,4,119.384
2GB,8,110.368
2GB,16,116.58
2GB,24,110.665
4GB,0,237.736
4GB,1,174.099
4GB,2,141.656
4GB,4,129.716
4GB,8,119.444
4GB,16,111.457
4GB,24,115.834
8GB,0,243.674
8GB,1,183.93
8GB,2,155.915
8GB,4,147.023
8GB,8,127.197
8GB,16,122.925
8GB,24,120.0
```
:::

## 参考 
- [CREATE INDEX | PostgreSQL 17.6文書](https://www.postgresql.jp/document/17/html/sql-createindex.html)

## おまけ1. 検証用データの準備

### GIN / GiST用データを作成する
Wikipediaのデータ(日本語)をダウンロードし、展開する。
```
mkdir dataload
cd dataload/
curl -LO https://ftp.acc.umu.se/mirror/wikimedia.org/dumps/jawiki/20251120/jawiki-20251120-pages-articles.xml.bz2
bunzip2 jawiki-20251120-pages-articles.xml.bz2
```

### データを整形する
単一のXMLファイルからプレーンテキストに分解する。
```
python3 parse_xml.py ./dataload/jawiki-20251120-pages-articles.xml
```
```
Parsing MediaWiki dump: ./dataload/jawiki-20251120-pages-articles.xml
Output directory: output
File size: 17.8GB
Starting from article number: 1

[██████████████████████████████████████████████████] 17.8GB/17.8GB (100.0%) | Articles: 3035125 | ETA: 0m0s

Completed! Processed 3035125 articles (numbered 1 to 3035125).
Articles saved in 'output' directory, organized by number ranges.
```
プレーンテキストを分割し、CSVにする。
```
python3 generate_csv.py output/ article
```

```
Input directory: output/
Output prefix: article

Found 3035125 article files in 3036 directories
Processing articles...
[██████████████████████████████████████████████████] 3035125/3035125 files (100.0%)

Completed in 264.8 seconds!
Total articles processed: 3035125
Total content lines: 153923860
Average lines per article: 50.7

Files created:
  - Metadata: article_metadata.csv
  - Content: article_content.csv
```

### 検証用テーブルを作成する
btree / hash用テーブルを作成する。
```sql
CREATE TABLE uuids (
  id bigint NOT NULL PRIMARY KEY,
  uuidv4 uuid DEFAULT gen_random_uuid(),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```
GIN用テーブルを作成する。
```sql
CREATE TABLE article_metadata (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  article_id INT UNIQUE NOT NULL,
  title TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE article_contents (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  article_id BIGINT REFERENCES article_metadata(article_id),
  row_id BIGINT,
  content TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (article_id, row_id)
);
```
### 検証用データをロードする
btree / hash用データを作成する。
```sql
# 5億レコードをINSERTする
INSERT INTO uuids (id) SELECT s FROM generate_series(1, 500000000) as s;
```
```
INSERT 0 500000000
Time: 1070722.400 ms (17:50.722)
```
GIN用データをロードする。
```sql
\timing
\COPY article_metadata (article_id, title)
FROM '/home/debian/article_metadata.csv'
WITH (FORMAT csv, HEADER true);
```
```
COPY 3035125
Time: 14758.601 ms (00:14.759)
```
```sql
\COPY article_contents (article_id, row_id, content)
FROM '/home/debian/article_content.csv'
WITH (FORMAT csv, HEADER true);
```
```
COPY 153923860
Time: 3943091.799 ms (01:05:43.092)
```

## おまけ2. データ整形用スクリプト
@[gist](https://gist.github.com/nnaka2992/8c31fe92c3eb16197f569ec334190cde)


