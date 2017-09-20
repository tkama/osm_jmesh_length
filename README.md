# osm_jmesh_length
OpenStreetMapの総延長を３次メッシュ毎に算出し、他の統計情報と比較してみる

## 実行環境
- PostgreSQL : 空間情報処理の拡張機能であるPostGISを使って加工、集計を行います。(開発時 9.6.3 / 推奨 9.3 以上)
- QGIS : shapeデータのPostgresqlへのインポート、可視化に用います。（開発時 2.14）

# STEP1 データの取得とインポート

## データ取得
### OSMデータのダウンロード
geofablinkから調査したいエリアのshapeファイルを選択します。
[リンク先](http://download.geofabrik.de/asia/japan.html)は日本エリア一覧が表示されます。関東地方を例に説明します。[Kantō regionの.shp.zip]([http://download.geofabrik.de/asia/japan/kanto-latest-free.shp.zip])

ダウンロードしてZIPファイルを解凍して下さい。

### 国土数値情報　道路密度・道路延長メッシュデータ

国土数値情報のダウンロードサービスの[Webサイト](http://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N04.html)からダウンロードが可能です。
１次メッシュ単位でダウンドード対象を選択可能です。今回はメッシュ番号5439の平成２２年度のデータを選択します。

ダウンロードしてZIPファイルを解凍して下さい。

### データのインポート

QGJISを立ち上げ、ダウンロードしたshapeファイルをPostgreSQLにインポートします。

［手順］データベース → DBマネージャ → PostGISを選択 → データベースを選択 → レイヤー/ファイルのインポートボタンを選択 

<img src="./img/qgis1.png " width=50%>
今回は空間情報処理を行いますので、「空間インデックスを作成する」にチェックを入れて下さい。

２つのshapeデータをQGISでPostgreSQLに投入しました。

|データ種別|ファイル名|テーブル名|
|:-----------|:------------|:------------|
|OpenStreetMap関東エリアの道路|gis.osm_roads_free_1.shp|osm_jp_kanto|
|道路密度・道路延長データ|N04-10_5439-jgd-g_RoadDensityAndLengthMesh_H22.shp|h22_road_5439|

# 集計
### OpenStreetMapの基本統計 道路延長の計算

関東エリアの道路延長を計算します。
geography型にキャストした線形状の空間情報は、ST_Length関数ではメートル単位で計測できます。

下記SQLは、道路種別(fclass)毎に道路延長と道路の本数をを求めます。

```sql:osm_jmesh_length.sql
SELECT
	fclass
	, SUM( ST_Length( the_geom :: geography ) ) AS road_length 
	, count(*)
FROM public.osm_jp_kanto
GROUP BY 1 ;
```

集計結果は[こちら( kanto_2017_09.csv )](kanto_2017_09.csv) でご覧になれます。

### ３次メッシュ毎の道路延長の算出

国土数値情報　道路密度・道路延長メッシュデータの３次メッシュの内部にあるOSMの道路を特定し、内部にある道路だけを切り出して、道路長を計算します。
比較対象として国土数値情報の道路延長も出力しています。

計算結果をテーブル名「ans_road_length」として保存しています。

```sql:sum_road_class_length.sql
CREATE TABLE ans_road_length AS 
SELECT
	  r.n04_001 AS mesh_code_3rd --３次メッシュコード 
	, SUM(  ST_Length( ST_Intersection( o.the_geom , r.the_geom )::geography ) ) AS osm_road_length --OSMの道路延長 
	, n04_055::int drm_road_length --道路総延長
	, r.the_geom --3次メッシュ形状
  FROM public.osm_jp_kanto AS o , public.h22_road_5439 AS r
  WHERE 
    --NOT IN 句で歩道・自転車道などを除外  
    fclass NOT
	IN('pedestrian',
	'track','footway','bridleway','steps','path','cycleway',
	'track_grade1','track_grade2','track_grade4','track_grade3','track_grade5',
	'unknown')
	AND
	ST_Intersects( o.the_geom , r.the_geom )
	AND
	n04_056 != 'unknown' -- 道路延長不明のメッシュを対象外
GROUP BY 1 , 3 , 4
	;
```

### ３次メッシュ毎の道路延長の比較

道路延長の算出結果から、OpenStreetMapと国土数値情報（DRM）とを比較します。

```sql:diff_road_class_length.sql
SELECT 
     mesh_code_3rd --3次メッシュコード
   , osm_road_length --OpenStreetMap道路延長[ｍ]
   , trunc(drm_road_length - osm_road_length) AS diff_length --道路延長差分[ｍ]
   , trunc( ((drm_road_length - osm_road_length)/drm_road_length * 100 ) :: numeric ,2) AS diff_percentage -- 道路延長比率[%] 
   , ST_AsText( the_geom ) AS wkt　-- メッシュ形状
FROM
 ans_road_length
ORDER BY 1 ;
```

結果は[こちら (osm_kanto_diff.tsv) ](osm_kanto_diff.tsv)
