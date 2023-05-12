# 地点別データを地図データと組み合わせたsvgを作成する


0. npm install でパッケージをインストール。パスを通す

1. N03-22_11_220101.geojsonをコピーしてsaitama.geojsonにする
2. saitama.geojsonを編集して、saitama.geo.jsonにする
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
3. 平面の地図にする
```
cat test/saitama.geo.json \
  | geoproject 'd3.geoConicEqualArea().parallels([35.4,36.3]).rotate([86, 0]).fitSize([960, 960], d)' \
  > test/saitama_projected.geo.json
```

4. geojsonをndjson形式に分割
```
ndjson-cat test/saitama_projected.geo.json \
  | ndjson-split 'd.features' \
  > test/saitama_projected.ndjson
```

5. csvデータのヘッダーを予め使用するカラムは英語名に変えて置き、ndjsonに変換する
```
csv2json -n work/saitama_data.csv \
  | ndjson-map '{ id: d.code, population: +d.population, density: +d.density }' \
  > test/saitama_data.ndjson
```

6.  地図とcsvデータを結合する
```
ndjson-join 'd.id' 'd.properties.N03_007' \
  test/saitama_data.ndjson \
  test/saitama_projected.ndjson \
  > test/saitama_projected_marged.ndjson
```

6. 結合したndjsonに人口密度と人口を追加する
```
  cat test/saitama_projected_marged.ndjson \
  | ndjson-map '
      d[1].properties.density = d[0].density,
      d[1].properties.population = d[0].population,
      d[1]
    ' \
  > test/saitama_projected_marged_2.ndjson
```
7. ndjsonからgeojsonを作成
```
cat test/saitama_projected_marged_2.ndjson \
  | ndjson-reduce \
  | ndjson-map '{ type: "FeatureCollection", features: d }' \
  > test/saitama_projected_marged_2.geo.json
```



8.  地図をgeojson形式からtopojson形式に変換
```
cat test/saitama_projected_marged_2.geo.json \
  | geo2topo tracts=- \
  > test/saitama_projected_marged_2.topo.json
```

9.  topojsonをきれいに整形する
```
cat test/saitama_projected_marged_2.topo.json \
  | toposimplify -p 1 -f \
  > test/saitama_projected_marged_2_simplified.topo.json
```



10. さらに整形。ファイルサイズを小さくする
```
cat test/saitama_projected_marged_2_simplified.topo.json \
  | topoquantize 1e5 \
  > test/saitama_projected_marged_2_simplified_quantized.topo.json
```


11.  色を追加する
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

12.  色をSVGに追加する
```
cat test/saitama_projected_marged_2_simplified_quantized_colors.ndjson \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > test/saitama.svg
```

13. svgの向きを変える
```
transform="rotate(-90)"
```

14. ブラウザで表示。完成
