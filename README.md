# Coordinate Projection with GHPython 
## GHPythonを使った座標投影

今回のハンズオンの内容は、GrasshopperのPythonを使った地図情報を扱う際に知っておくと便利な内容になっています。メインは座標の変換をうたっていますが、具体的には次のようなことを学びます。

- 緯度経度座標データをGoogle Maps等で採用されているタイル座標に変換する方法
- タイル座標を利用して国土交通省が公開し、API経由でアクセスできる基盤地図情報の取得の仕方
- GeoJson形式で得られる基盤地図情報の仕様
- 緯度経度で表現されているポリゴンの頂点座標をUTM座標に変換（投影）して平面に描写する方法



### Step 1: 緯度経度+ズームレベルをタイル座標に変換

GIS（地理情報システム）系のAPIで場所の指定をする場合、大きく二通りがあります。任意の一点の情報を取得する場合は緯度経度、あるズームレベルでのタイル情報を取得する場合はタイル座標を使います。

- 緯度経度：世界測地系（WGS 84）に基づいた、測量や位置決定などに用いられる座標情報。
- タイル座標：地図画像上の位置であるピクセル座標を256で割ったときの商の整数部で表される座標情報。Google MapsやOpen Street Maps, 国土交通省のタイル情報などで用いられる。

[Pythonでタイル座標⇔緯度経度の変換](https://note.sngklab.jp/?p=72)


<strong>緯度経度 -> タイル座標への変換の計算式</strong>

<img width="579" alt="2021-10-02_00-08-53" src="https://user-images.githubusercontent.com/2336918/135653828-140fa144-96a1-468f-9ea2-9495b4161e0b.png">

```python
import rhinoscriptsyntax as rs

from math import log
from math import tan
from math import pi

if lon is not None and lat is not None and z is not None:
    x = int((lon / 180 + 1) * 2**z / 2)
    y = int(((-log(tan((45 + lat / 2) * pi / 180)) + pi) * 2**z / (2 * pi)))

```

### Step 2: タイル座標から基盤地図情報（ベクトルデータ）を国土交通省のAPI経由で取得

国土交通省は様々なタイルデータを公開しています。例えば次のようなものがあります。

[地理院タイル一覧](https://maps.gsi.go.jp/development/ichiran.html)

[国土地理院ベクトルタイル提供実験](https://github.com/gsi-cyberjapan/vector-tile-experiment)

この中で、今回は次のタイルデータを使ってみようと思います。

[基盤地図情報の GeoJSON タイル](https://github.com/gsi-cyberjapan/experimental_fgd)

このデータは、次のようなURLをGETリクエストを送ることでgeojson形式で指定のタイル座標の建物や街区のアウトラインの座標データを得ることができます。

```
https://cyberjapandata.gsi.go.jp/xyz/experimental_fgd/{z}/{x}/{y}.geojson

{z}: ズームレベル
{x}: タイル座標のX方向の番号
{y}: タイル座標のY方向の番号
```


<strong>基盤地図情報（ベクトル形式のタイルデータ）の取得</strong>

<img width="682" alt="2021-10-02_00-10-06" src="https://user-images.githubusercontent.com/2336918/135653977-30ed6efc-8d50-4f81-bada-75be882fe8b0.png">

```python
import rhinoscriptsyntax as rs
### Rest APIを利用するための.NET系ライブラリをインポート
from System.Net import HttpWebRequest
from System.IO import StreamReader
### JSON形式を扱うための.NET系ライブラリをインポート
import clr
clr.AddReference('System.Web.Extensions')
from System.Web.Script.Serialization import JavaScriptSerializer


if run:
    dict = {"features": []}
    for i in range(t):
        for n in range(t):
            ### 指定したタイルの数だけ基盤地図情報をAPI経由で取得し、featuresというキーを持つDictionaryのリストに結果を追加していきます。
            apiurl = "https://cyberjapandata.gsi.go.jp/xyz/experimental_fgd/{0}/{1}/{2}.geojson"
            uri = apiurl.format(z,x+i,y+n)
            webRequest = HttpWebRequest.Create(uri)
            webRequest.Timeout=1000
            with webRequest.GetResponse() as response:
            
                streamReader = StreamReader(response.GetResponseStream())
                jsonData = streamReader.ReadToEnd()
                ### 得られたJSONテキストデータをDictionaryに変換（デシリアライズ）します。
                js = JavaScriptSerializer()
                dataDict = js.DeserializeObject(jsonData)
                dict["features"].extend(dataDict["features"])
    
    ### 最終的にDictionaryデータをシリアライズしてJSONテキストデータに戻します。
    json = js.Serialize(dict)

```

### Step 3: GeoJsonデータの中の緯度経度情報をUTM座標に変換して図を描く

緯度経度で表される座標はそのままでは平面に描かれた図形の頂点の位置情報として使うことはできません。この緯度経度が採用されていて世界的によく使われる座標系にWGS 84があります。

- 緯度：赤道を基準として南北へそれぞれ90度で表される角度情報
- 経度：イギリスのグリニッジ天文台跡を通る子午線を基準に東西へそれぞれ180度まで表される角度情報

このように角度情報に表される位置情報で、赤道近くと北極・南極近くでは同じ１度でも距離が全然異なってきます。

これを平面に投影して地図でよくみるような図を描くために使われる地図投影手法の一つがUTM（ユニバーサル横メルカトル図法）というものです。これは基準となる経度を決め、任意の点の位置がその経度から東西に何メートル、赤道位置から南北に何メートルの位置にあるかを示すのに使うことができる図法です。世界的によく使われる座標基準はWGS 84と呼ばれるもので、アメリかの東中央あたりの経度が基準になっています。

[More details about UTM Grid Zones](https://www.maptools.com/tutorials/grid_zone_details)

緯度経度からUTM座標に変換するには、緯度経度に応じた歪みによる誤差の吸収をするための計算をする必要があるため、非常に面倒です。ということでこれは専用の座標変換ライブラリを使って対応します。


<strong>GeoJsonの読み込み及び緯度経度->UTM座標変換</strong>

<img width="660" alt="2021-10-02_00-10-25" src="https://user-images.githubusercontent.com/2336918/135654161-2c41a3a9-eba7-4875-9616-ab98853f3a7d.png">

```python
import rhinoscriptsyntax as rs
import Rhino.Geometry as geo

### 座標変換のためにC#のライブラリであるProjNetを使います。先にライブラリの保存先をシステムのパス（ライブラリ参照先）に保存しましょう。
import sys
if libdir not in sys.path:
    sys.path.append(libdir)

import clr
clr.AddReference('ProjNet')
clr.AddReference('GeoAPI.CoordinateSystems')
clr.AddReference('System.Web.Extensions')
from System.Web.Script.Serialization import JavaScriptSerializer
from ProjNet import *
from ProjNet.CoordinateSystems import *
from ProjNet.CoordinateSystems.Transformations import *
from GeoAPI.CoordinateSystems import *
from GeoAPI.CoordinateSystems.Transformations import *

### ProjNetを使った座標変換系の関数を作ります。
### 6経度ずつ分割されたゾーン情報を取得する関数。
def zone(coordinates):
    if 56 <= coordinates[1] < 64 and 3 <= coordinates[0] < 12:
        return 32
    if 72 <= coordinates[1] < 84 and 0 <= coordinates[0] < 42:
        if coordinates[0] < 9:
            return 31
        elif coordinates[0] < 21:
            return 33
        elif coordinates[0] < 33:
            return 35
        return 37
    return int((coordinates[0] + 180) / 6) + 1

### ８緯度ずつ水平分割したときのバンド番号を取得する関数。
def letter(coordinates):
    return 'CDEFGHJKLMNPQRSTUVWXX'[int((coordinates[1] + 80) / 8)]

### 緯度経度をUTM座標に変換する関数。
def LLtoUTM(coordinates):
    z = zone(coordinates)
    l = letter(coordinates)
    ctfac = CoordinateTransformationFactory()

    wgs84 = CoordinateSystems.GeographicCoordinateSystem.WGS84
    utm = CoordinateSystems.ProjectedCoordinateSystem.WGS84_UTM(z, coordinates[1] > 0)
    trans = ctfac.CreateFromCoordinateSystems(wgs84, utm)

    result = trans.MathTransform.Transform(coordinates);
    x = result[0]
    y = result[1]
    return z, l, x, y
    

### geojsonデータからベクトルデータを取得します。
if json is not None:
    bldPolylines = []
    rdgPolylines = []
    
    js = JavaScriptSerializer()
    dataDict = js.DeserializeObject(json)
    
    avepos = geo.Point3d(0,0,0)
    count = 0
    
    features = dataDict["features"]
    for feature in features:
        ### 建物とリッジ線（街区など）の図形情報を取得します。
        type = feature["properties"]["class"]
        coords = feature["geometry"]["coordinates"]
        if "BldL" in type or "RdEdg" in type:
            points = []
            for coord in coords:
                ### 緯度経度情報をUTM座標に変換します。
                z, l, x, y = LLtoUTM((float(coord[0]), float(coord[1])))
                pt = geo.Point3d(x, y, 0)
                points.append(pt)
                count += 1
                avepos += pt
            
            ### 変換したUTM座標のリストからポリラインを描写します。
            polyline = rs.AddPolyline(points)
            if "BldL" in type:
                bldPolylines.append(polyline)
            elif "RdEdg" in type:
                rdgPolylines.append(polyline)
            
    avepos /= float(count)
    
    ### 建物とリッジ線のバウンダリの中心点が原点に来るように移動します。
    movedBldPolylines = []
    movedRdgPolylines = []
    for polyline in bldPolylines:
        mpolyline = rs.MoveObject(polyline, -avepos)
        movedBldPolylines.append(mpolyline)
    for polyline in rdgPolylines:
        mpolyline = rs.MoveObject(polyline, -avepos)
        movedRdgPolylines.append(mpolyline)
        
    bld = movedBldPolylines
    rdg = movedRdgPolylines
```

### 結果

UTM座標に変換すると頂点座標はメートルベースで表現されることなります。点と点の距離が100の場合は100m離れているということです。このような緯度経度を平面座標に投影する手法は地図を扱うことがある場合はかなりの高い確率で実装する必要があり、手順を知っておけば別の言語でも再現が十分に可能なので、役に立つかと思います。

<img width="1200" alt="2021-10-02_02-19-40" src="https://user-images.githubusercontent.com/2336918/135661543-425e3350-84ef-49ef-a8ab-ef1c566a03f8.png">

<img width="708" alt="2021-10-02_02-20-31" src="https://user-images.githubusercontent.com/2336918/135661563-e060a0de-4e37-4f34-9a3c-a4e4b85b529b.png">