

N03-22_11_220101.geojsonをコピーしてsaitama.geojsonにする


20 平面の地図にする
cat test/saitama.geo.json \
  | geoproject 'd3.geoConicEqualArea().parallels([35.4,36.3]).rotate([86, 0]).fitSize([960, 960], d)' \
  > test/saitama_projected.geo.json

21 geojsonをndjson形式に分割

ndjson-cat test/saitama_projected.geo.json \
  | ndjson-split 'd.features' \
  > test/saitama_projected.ndjson


22 csvデータをndjsonに変換する

csv2json -n work/saitama_data.csv \
  | ndjson-map '{ id: d.code, population: +d.population, density: +d.density }' \
  > test/saitama_data.ndjson




23  地図とcsvデータを結合する

ndjson-join 'd.id' 'd.properties.N03_007' \
  test/saitama_data.ndjson \
  test/saitama_projected.ndjson \
  > test/saitama_projected_marged.ndjson

24. 結合したndjsonから人口密度を計算

  cat test/saitama_projected_marged.ndjson \
  | ndjson-map '
      d[1].properties.density = d[0].density,
      d[1].properties.population = d[0].population,
      d[1]
    ' \
  > test/saitama_projected_marged_2.ndjson

25 新しいndjsonからgeojsonを作成

cat test/saitama_projected_marged_2.ndjson \
  | ndjson-reduce \
  | ndjson-map '{ type: "FeatureCollection", features: d }' \
  > test/saitama_projected_marged_2.geo.json




26  地図をgeojson形式からtopojson形式に変換

cat test/saitama_projected_marged_2.geo.json \
  | geo2topo tracts=- \
  > test/saitama_projected_marged_2.topo.json
  

27  topojsonをきれいに整形する

cat test/saitama_projected_marged_2.topo.json \
  | toposimplify -p 1 -f \
  > test/saitama_projected_marged_2_simplified.topo.json




? 28 さらに整形。ファイルサイズを小さくする

cat test/saitama_projected_marged_2_simplified.topo.json \
  | topoquantize 1e5 \
  > test/saitama_projected_marged_2_simplified_quantized.topo.json



29  色を追加する

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

30  色をSVGに追加する
cat test/saitama_projected_marged_2_simplified_quantized_colors.ndjson \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > test/saitama.svg

transform="rotate(-90)"


