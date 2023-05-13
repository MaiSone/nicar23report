# 地点別データを地図データと組み合わせたSVGを作成する


0. npm install でパッケージをインストール。パスを通す。testディレクトリを作成
```
npm install
export PATH=$PATH:./node_modules/.bin
mkdir test
```
## 地図データの準備

1. N03-22_11_220101.geojsonをコピー・ペーストで複製してsaitama.geo.jsonに名前を変更

2. saitama.geo.jsonを編集して1行のデータにする（初めの部分）
- 開始部分
  - before
```
 {
    "type": "FeatureCollection",
    "name": "N03-22_11_220101",
    "crs": { "type": "name", "properties": { "name": "urn:ogc:def:crs:EPSG::6668" } },
    "features": [
 ```
   - after
 ```
 {
    "type": "FeatureCollection","features": [
```
- 終わりの部分
  - before
```
835145577006983, 35.965320918925443 ] ] ] } 
}
]
}
```
  - after
```
835145577006983, 35.965320918925443 ] ] ] } }]}
```

3. test/ディレクトリに移動させる


4. saitama.geo.jsonを平面の地図にする
```
cat test/saitama.geo.json \
  | geoproject 'd3.geoConicEqualArea().parallels([35.4,36.3]).rotate([86, 0]).fitSize([960, 960], d)' \
  > test/saitama_projected.geo.json
```

5. geojsonをndjson形式に分割
```
ndjson-cat test/saitama_projected.geo.json \
  | ndjson-split 'd.features' \
  > test/saitama_projected.ndjson
```

## 地点別データの準備

6. saitama_data.csvの使用するカラムを英語に変更する

- 標準地域コード：code
- 前年同月からの増減率（％）：rate
- 人口密度（人／km2）：density

7. saitama_data.csvをtest/に移動する
8. saitama_data.csvをndjsonに変換する
```
csv2json -n test/saitama_data.csv \
  | ndjson-map '{ id: d.code, rate: +d.rate, density: +d.density }' \
  > test/saitama_data.ndjson
```



## 地図データと地点別データを組み合わせる

9.  地図とcsvデータを結合する

```
ndjson-join 'd.id' 'd.properties.N03_007' \
  test/saitama_data.ndjson \
  test/saitama_projected.ndjson \
  > test/saitama_projected_marged.ndjson
```

10. 結合したndjsonに人口密度と人口の前年同月増減率を追加する
```
cat test/saitama_projected_marged.ndjson \
  | ndjson-map '
      d[1].properties.density = d[0].density,
      d[1].properties.rate = d[0].rate,
      d[1]
    ' \
  > test/saitama_projected_marged_2.ndjson
```
11. ndjsonからgeojsonを作成
```
cat test/saitama_projected_marged_2.ndjson \
  | ndjson-reduce \
  | ndjson-map '{ type: "FeatureCollection", features: d }' \
  > test/saitama_projected_marged_2.geo.json
```



12.  geojson形式からtopojson形式に変換
```
cat test/saitama_projected_marged_2.geo.json \
  | geo2topo tracts=- \
  > test/saitama_projected_marged_2.topo.json
```

13.  topojsonをきれいに整形する
```
cat test/saitama_projected_marged_2.topo.json \
  | toposimplify -p 1 -f \
  > test/saitama_projected_marged_2_simplified.topo.json
```



14. さらに整形。ファイルサイズを小さくする
```
cat test/saitama_projected_marged_2_simplified.topo.json \
  | topoquantize 1e5 \
  > test/saitama_projected_marged_2_simplified_quantized.topo.json
```


15.  色を追加する
```
cat test/saitama_projected_marged_2_simplified_quantized.topo.json \
  | topo2geo tracts=- \
  | ndjson-map \
      -r d3array=d3-array \
      -r d3scale=d3-scale \
      -r d3chroma=d3-scale-chromatic \
      '
        z = d3scale.scaleSequential(d3chroma.interpolateMagma).domain([100, 0]),
        d.features.forEach((f) => {
          f.properties.fill = z(Math.sqrt(f.properties.density));
        }),
        d
      ' \
  | ndjson-split 'd.features' \
  > test/saitama_projected_marged_2_simplified_quantized_colors.ndjson
```

16.  色をSVGに追加する
```
cat test/saitama_projected_marged_2_simplified_quantized_colors.ndjson \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > test/saitama.svg
```

17. 市区町村の境界を追加する

```
cat test/saitama_projected_marged_2_simplified_quantized.topo.json \
  | topomerge -k 'd.properties.N03_007.slice(0, 5)' counties=tracts \
  | topomerge --mesh -f 'a !== b' counties=counties \
  | topo2geo -n counties=- \
  | ndjson-map '
      d.properties = {"stroke": "#000", "stroke-opacity": 0.3},
      d
    ' \
  > test/saitama_border.ndjson
```
18. 境界をSVGに追加する

```
(
  cat test/saitama_projected_marged_2_simplified_quantized_colors.ndjson;
  cat test/saitama_border.ndjson;
) \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > test/saitama_border.svg
```


19. SVGの向きを変える

- before
```
<svg version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="960" height="960" viewBox="0 0 960 960" fill="none">
```
- after
```
<svg version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="960" height="960" viewBox="0 0 960 960" fill="none" transform="rotate(-90)">
```

20. ブラウザで表示。完成

## 番外編
- 人口密度から前年同月増減率（％）に色付けに使用するデータを変更する
- 平方根を返すMath.sqrt()をトル
https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Math/sqrt
- domain()の値を調整。今回は試しに[100,0]から[4,-4]に変更

15.  色を追加する
```
cat test/saitama_projected_marged_2_simplified_quantized.topo.json \
  | topo2geo tracts=- \
  | ndjson-map \
      -r d3array=d3-array \
      -r d3scale=d3-scale \
      -r d3chroma=d3-scale-chromatic \
      '
        z = d3scale.scaleSequential(d3chroma.interpolateMagma).domain([4, -4]),
        d.features.forEach((f) => {
          f.properties.fill = z(f.properties.rate);
        }),
        d
      ' \
  | ndjson-split 'd.features' \
  > test/saitama_projected_marged_2_simplified_quantized_colors.ndjson
```