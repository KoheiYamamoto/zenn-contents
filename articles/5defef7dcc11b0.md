---
title: "Xamarin.Forms (Android) で Geocoding を実装、Cognitive Search で検索機能強化"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Xamarin.Forms", "Geocoding", "Search", "DB"]
published: false
---

# はじめに

以下のものを準備
- Visual Studio 2022
- Azure アカウント および サブスクリプション（今回使用するのは無料Tier）
- Google アカウント
- 前回、「Xamarin.Forms (Android) で Map を使って現在地を取得し、ピンを追加する」という投稿にて作成した Android アプリ

# 全体像

今回は、店舗（座標）情報に紐づいたクーポンDBがあり、その情報を地図上にマッピングするシナリオを想定している。~~つまり、マップにマップするわけである。~~

![overview.png](/images/5defef7dcc11b0/overview.png "全体像")

ちなみに今回、[Azure Cognitive Search](https://docs.microsoft.com/ja-jp/azure/search/search-what-is-azure-search) という虫眼鏡経由でクーポンDBを読みに行っているわけではあるが、Cognitive Search を簡単に説明すると以下の通りである。

- 検索機能をアプリケーションに簡単に搭載できるようにするサービス
- 一般的に専門知識が必要とされるインデクシング（データベースやストレージを検索可能にするための事前処理）をGUIでの数クリックで自動解決
- DBや文書だけでなく、パワポやエクセル、画像データなどの非構造データを構造データに変換して、検索を可能にしてくれる
  
普段は、組織内に眠るデータ資産のDXに使用することが多いが、今回のように外部サービスとしても組み込むことが可能である。Search を使用すると、自動的に場所情報や、エンティティ（人や組織など）を自動的に抽出してくれるが、今回は、クーポンDBをスキャンし、抽出されたキーワードを活用する。

アプリとして想定している動きは以下の4パターンである。
- 現在地周辺のクーポン情報を表示
- お出かけ予定の場所を検索して、その付近のクーポン情報を表示
- お出かけ予定の場所を検索して、その付近でトレンドなキーワードを表示
- お出かけ予定の場所を検索して、任意のキーワードに関連するクーポンだけをフィルタリングして取得・表示

# 実装
1. DBをデプロイ
2. Cognitive Search をデプロイ・インデクシング
3. 現在地付近のクーポンを表示
4. 場所検索ボタンを追加して、Geocoding により緯度経度を取得・その付近のクーポンを表示
5. 検索場所付近でトレンドなキーワードを表示
6. キーワードでフィルタリングしたクーポンを取得・表示

## DBをデプロイ
今回は、DBとしてAzure SQL DB を使用する。[こちら](https://docs.microsoft.com/ja-jp/learn/modules/deploy-azure-sql-database/)の公式チュートリアルを参考に usernameとパスワードを設定してデプロイする。テーブルのスキーマは以下のように定義する。
### テーブルのスキーマ定義
```
CREATE TABLE COUPON
(
    ID INT PRIMARY KEY NOT NULL,
    NAME NVARCHAR(40) NOT NULL,
    DESCRIPTION NVARCHAR(1024) NOT NULL,
    LAT FLOAT NOT NULL,
    LONG FLOAT NOT NULL,
)
```
### レコード追加
レコードの追加は以下を参考に。1,2,3段落目はそれぞれ東京駅付近、渋谷駅付近、Googleplex（現在地）付近のレコード情報を想定している。
```
insert into [dbo].[COUPON] (id, name, description, lat, long) values (0, N'アーミヤン', N'お食事の方にアイスクリームをご提供', 35.68276366571134, 139.77093758851996);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (1, N'冷野菜しゃぶしゃぶ', N'自家製の赤ワインを一杯サービス', 35.677534875914624, 139.7713653726625);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (2, N'東京ビール園東京駅店', N'オリジナルの赤ワインを一杯まで無料', 35.679139952250516, 139.7620559765365);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (3, N'東京駅ビール園丸の内店', N'お食事の方に赤ワイン1杯をプレゼント', 35.675102379083384, 139.76751531608127);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (4, N'東京駅ビール園皇居店', N'お食事の方に白ワイン1杯を無料サービス', 35.68014931360572, 139.76592640387892);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (5, N'私のフレンチ', N'赤ワインに合うチーズを食前にお持ちします！', 35.67478796934566, 139.76513194775845);

insert into [dbo].[COUPON] (id, name, description, lat, long) values (6, N'豚角', N'ドリンクバーをサービス', 35.65910610009796, 139.7031271060302);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (7, N'マクダナルド', N'丸ごとメロンパンを先着10名様に', 35.65827796397028, 139.7015714248115);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (8, N'GENA', N'ジーンズを10%オフ', 35.66084067970689, 139.6989634558757);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (9, N'べビべビ', N'ベビー用品・赤ちゃん用のジーンズを20%割引！', 35.65736221548825, 139.69829798633705);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (10, N'FUNDI', N'アメリカ直輸入のジーンズを店頭にて1日貸出！', 35.65682150796175, 139.70158096939446);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (11, N'Trende', N'旅行用品・キャンプ道具を無料レンタル', 35.65552379496184, 139.70277881456403);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (12, N'Fashionista', N'指輪などのファッションアイテムを5%ディスカウント！', 35.65729012136291, 139.7057290628521);

insert into [dbo].[COUPON] (id, name, description, lat, long) values (15, N'Big Donuts', N'チョコドーナツ1つプレゼント', 37.4242905468471, -122.08590705229199);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (16, N'San Jose Inn', N'連泊のお客様は20%割引', 37.423591006695645, -122.07932655042501);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (17, N'Sun Resort', N'お子様の宿泊料金 無料!', 37.42011378366949, -122.08621794214399);
insert into [dbo].[COUPON] (id, name, description, lat, long) values (18, N'World Square', N'ファッションアイテム全品5%オフ', 37.420587024697895, -122.07994833012899);
```
作成後のDBは以下のように見えているはず。
![db.png](/images/5defef7dcc11b0/db.png "DBイメージ")

## Cognitive Search をデプロイ・インデクシング
Cognitive Search をデプロイして、👆のDBをインデクシングする。
### Search のデプロイ
[こちら](https://docs.microsoft.com/ja-jp/azure/search/search-get-started-portal)の公式Docsを参考に、Search リソースをデプロイする。(Japan East では、要求リクエストが多いらしく、Japan West にて作成した　2022/5/1現在)

### インデクシング
データのインポートを選択し、先ほどのDBを選択してインデクシングを行う。以下の点に注意されたい。
- エンリッチメントの追加選択にて、テキストに認知技術をすべてチェック。
- ナレッジストアへのエンリッチメントの保存にて、Azure Tableへのプロジェクションを全選択（適当なストレージアカウントが必要）
- データのインポートにて、取得可能カラムと検索可能カラムを全選択。また、keyphrasesをフィルタ可能・ファセット可能に選択。（全部が全部使うわけではないが、抜けがあるとインデクサを作成し直すことになるので、広く選択するようにしている。）

クエリを投げると以下のようにjsondで返される。
![search.png](/images/5defef7dcc11b0/search.png "Searchイメージ")

### Constants.cs を共通プロジェクトに作成
Searchへのクエリを可能にするために、URLとAPIキーを保持する必要があるため、そのためのクラスを追加。
```
namespace PushDemoAndroid
{
    public class Constants
    {
        public const string SearchRequestURL = "<URLを挿入>";
        public const string SearchRequestKey = "<APIキーを挿入>";
    }
}
```

## 現在地付近のクーポンを表示
### Nu Get
以下を共通プロジェクトにインストールする。
- Newtonsoft.Json (ver.13.0.1)

### MainPage.xaml.csを編集
まず、```public partial class MainPage : ContentPage```クラス内にHTTP クライアントのインスタンスを宣言。聞いた話によると、基本的には、アプリケーションの中で1つのインスタンスを使いまわすのがベストプラクティスらしい。（どこで聞いたかは忘れたので、引用しないでほしい）
```
static readonly HttpClient client = new HttpClient();
```
```CurrentLocation_Clicked()```関数を以下のように修正
```
private async void CurrentLocationButton_Clicked(object sender, EventArgs e)
{
    try
    {
        // clear pins
        map.Pins.Clear();
        // get current location
        var request = new GeolocationRequest(GeolocationAccuracy.Medium);
        var location = await Geolocation.GetLocationAsync(request);
        if (location != null)
        {
            // set current location to gpslabel at the bottom of the screen
            GPSlabel.Text = $"{location.Latitude},{location.Longitude}";
            // enable IsShowingUser botton in the map
            map.IsShowingUser = true;
            // move the map to current location
            Position position = new Position(Convert.ToDouble(location.Latitude), Convert.ToDouble(location.Longitude));
            MapSpan mapSpan = new MapSpan(position, 0.015, 0.015);
            map.MoveToRegion(mapSpan);
            // get records from azure search
            HttpResponseMessage res = await client.GetAsync(Constants.SearchRequestURL + "*" + "&api-key=" + Constants.SearchRequestKey);
            string content = await res.Content.ReadAsStringAsync();
            JObject json = JObject.Parse(content);
            // get records nearby
            var nearRecords = GetNearRecords(position, json);
            // add pins nearby
            foreach (JObject item in nearRecords)
            {
                AddPin(Convert.ToString(item.GetValue("NAME")), Convert.ToString(item.GetValue("DESCRIPTION")), new Position(Convert.ToDouble(item.GetValue("LAT")), Convert.ToDouble(item.GetValue("LONG"))));
            }
        }
    }
    catch (Exception)
    {
        Console.WriteLine("Location is not obtained.");
    }
}
```
見てわかる通り、
```
// get current location
var request = new GeolocationRequest(GeolocationAccuracy.Medium);
var location = await Geolocation.GetLocationAsync(request);
```
で現在地を取得し、
```
// get records from azure search
HttpResponseMessage res = await client.GetAsync(Constants.SearchRequestURL + "*" + "&api-key=" + Constants.SearchRequestKey);
string content = await res.Content.ReadAsStringAsync();
JObject json = JObject.Parse(content);
```
でREST API経由で空リクエストをSearchに問い合わせて、全レコードを返してもらっている。その後、
```
// get records nearby
var nearRecords = GetNearRecords(position, json);
```
にて、```position``` 変数（緯度経度）付近に存在するクーポンデータだけをjson arrayを受取り、
```
// add pins nearby
foreach (JObject item in nearRecords)
{
    AddPin(Convert.ToString(item.GetValue("NAME")), Convert.ToString(item.GetValue("DESCRIPTION")), new Position(Convert.ToDouble(item.GetValue("LAT")), Convert.ToDouble(item.GetValue("LONG"))));
}
```
にて、ピンを立てている。

### GetNearRecords() の実装
```position``` 変数付近（座標的に0.005以内であれば、レコードを残すようにしている）
```
 private List<Object> GetNearRecords(Position position, JObject json)
{
    var nearRecords = new List<Object>();
    foreach (JObject item in json["value"])
    {
        if ( (Math.Abs(position.Latitude - Convert.ToDouble(item.GetValue("LAT"))) <= 0.05) && (Math.Abs(position.Longitude - Convert.ToDouble(item.GetValue("LONG"))) <= 0.05))
        {
            nearRecords.Add(item);
        }
    }
    return nearRecords;
}
```
これで現在地付近で探すボタンを押すと、きちんとSearch経由でDBが保持する現在地付近のクーポンが表示されることが確認できる。

（※注意点；章末を参照）

![currentlocation.png](/images/5defef7dcc11b0/currentlocation.png "現在地付近のクーポン")

## 場所検索ボタンを追加して、Geocoding により緯度経度を取得・その付近のクーポンを表示
### MainPage.xamlを以下のように編集
次項と被る部分はあるが、以下のように編集して、「場所を入力」エントリと検索ボタンを追加する。
```
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" xmlns:maps="clr-namespace:Xamarin.Forms.Maps;assembly=Xamarin.Forms.Maps"
             x:Class="PushDemoAndroid.MainPage">

    <StackLayout>
        <Frame BackgroundColor="#2196F3" Padding="24" CornerRadius="0">
            <Label Text="Welcome to Xamarin.Forms!" HorizontalTextAlignment="Center" TextColor="White" FontSize="36"/>
        </Frame>

        <StackLayout Orientation="Horizontal">
            <Label Text="このエリアで人気なキーワード：" HorizontalOptions="Center" TextColor="Black"/>
            <Label Text="N/A" x:Name="Trend" HorizontalOptions="Center" TextColor="Black"/>
        </StackLayout>

        <maps:Map x:Name="map" />

        <StackLayout Orientation="Horizontal">
            <Button Text="現在地周辺で探す"  HorizontalOptions="Center" Clicked="CurrentLocationButton_Clicked"/>
            <StackLayout Orientation="Vertical">
                <Label Text="現在地：" HorizontalOptions="Start" TextColor="Black"/>
                <Label x:Name="GPSlabel" HorizontalOptions="Start" TextColor="Black"/>
            </StackLayout>
        </StackLayout>

        <StackLayout Orientation="Horizontal">
            <Entry x:Name="entry" Placeholder="場所を入力"  HorizontalOptions="FillAndExpand" PlaceholderColor="Olive" />
            <Entry x:Name="keyphrase" Placeholder="キーワードを入力"  HorizontalOptions="FillAndExpand" PlaceholderColor="Olive" />
        </StackLayout>
        <StackLayout Orientation="Horizontal">
            <Button Text="場所とキーワードから検索"  HorizontalOptions="FillAndExpand" Clicked="SearchLocationButton_Clicked"/>
        </StackLayout>
    </StackLayout>

</ContentPage>
```
「キーワードを入力」エントリと「人気なキーワード」については、次項で触れる。

### MainPage.xamls.csにてGeocodingを実装
「場所を入力」エントリに場所名を入れて、検索ボタンをクリックする。その際、```SearchLocationButton_Clicked()``` 関数が呼び出されるので、こちらを実装する。
```
private async void SearchLocationButton_Clicked(object sender, EventArgs e)
{
    try
    {
        // clear pins
        map.Pins.Clear();
        // get coordinates geocoded from entry.Text 
        Geocoder geoCoder = new Geocoder();
        IEnumerable<Position> approximateLocations = await geoCoder.GetPositionsForAddressAsync(entry.Text);
        Position position = approximateLocations.FirstOrDefault();
        if (position != null)
        {
            // move the map to current location
            MapSpan mapSpan = new MapSpan(position, 0.015, 0.015);
            map.MoveToRegion(mapSpan);
            // if keyphrase is not specified
            if (string.IsNullOrEmpty(keyphrase.Text))
            {
                // set current location to gpslabel at the bottom of the screen
                GPSlabel.Text = $"{position.Latitude},{position.Longitude}";
                // get records from azure search
                HttpResponseMessage res = await client.GetAsync(Constants.SearchRequestURL + "*" + "&api-key=" + Constants.SearchRequestKey);
                string content = await res.Content.ReadAsStringAsync();
                JObject json = JObject.Parse(content);
                // get records nearby
                var nearRecords = GetNearRecords(position, json);
                // add pins nearby
                foreach (JObject item in nearRecords)
                {
                    AddPin(item.GetValue("NAME").ToString(), item.GetValue("DESCRIPTION").ToString(), new Position(Convert.ToDouble(item.GetValue("LAT")), Convert.ToDouble(item.GetValue("LONG"))));
                }
            }
        }

    }
    catch (Exception)
    {
        Console.WriteLine("Location cannot be obtained.");
    }
}
```
見てわかる通り、検索ボタンを押すと
```
// get coordinates geocoded from entry.Text 
Geocoder geoCoder = new Geocoder();
IEnumerable<Position> approximateLocations = await geoCoder.GetPositionsForAddressAsync(entry.Text);
Position position = approximateLocations.FirstOrDefault();
```
「場所を入力」エントリ（```entry.Text```）の値がGeocodingされ、結果が座標として ```position``` 変数に格納され、その座標へ画面を移動させる。もし、「キーワードを入力」エントリが空であれば、そのまま Search の REST API を叩いて、レコード情報を落とし、あとは前項と同じ処理フローである。

（※注意点；章末を参照）

![searchlocation_tokyo.png](/images/5defef7dcc11b0/searchlocation_tokyo.png "検索場所付近のクーポン")

## 検索場所付近でトレンドなキーワードを表示
検索場所によって、クーポン内容の分布が異なることは想像がつく。お出かけ前にそのエリアごとにホットなキーワードを可視化するための機能を一部実装する。
### MainPage.xaml.csを編集
前項の ```GetNearRecords()``` の後に1行足す。
```
// get trend keyphrase
Trend.Text = GetNearRecords(nearRecords);
```
これにより画面上で表示しているキーワードのラベルを更新している。
### GetNearRecords()の実装
以下の関数を追加する。
```
 private string getTrendKeyphrase(List<object> nearRecords)
{
    var listKeyphrase = new List<string>();
    foreach (JObject item in nearRecords)
    {
        foreach (var kp in item["keyphrases"])
        {
            listKeyphrase.Add(kp.ToString());
        }
    }
    var mostCommon = (from item in listKeyphrase
                    group item by item into g
                    orderby g.Count() descending
                    select g.Key).First();
    return mostCommon;
}
```
上の関数は、検索場所付近に絞ったクーポン情報を受け取り、最頻キーワードを抽出している。

（※注意点；章末を参照）

![searchlocation_shibuya.png](/images/5defef7dcc11b0/searchlocation_shibuya.png "検索場所付近のトレンド")

## キーワードでフィルタリングしたクーポンを取得・表示
往々にして、場所を検索した上で、ホットなキーワードやその他のキーワードによってフィルタリングをしたいこともあるかと思う。その場合は、「キーワードを入力」エントリ（keyphrase）にもキーワードを入力した上で検索ボタンを押すことで、Searchに対して関連するキーワードを持つクーポンだけを問い合わせることができる。
### SearchLocationButton_Clicked() 関数を編集
既述のSearchLocationButton_Clicked() 関数にキーワードも指定されていた場合の時の挙動を実装する。
```
private async void SearchLocationButton_Clicked(object sender, EventArgs e)
{
    try
    {
        ...
        if (position != null)
        {
           ...
            }
            else // if keyphrase is specified
            {
                // set current location to gpslabel at the bottom of the screen
                GPSlabel.Text = $"{position.Latitude},{position.Longitude}";
                // get records from azure search with keyphrase specified 
                HttpResponseMessage res = await client.GetAsync(Constants.SearchRequestURL + "\"" + keyphrase.Text + "\"" + ",searchFields\"keyphrases\"" + "&api-key=" + Constants.SearchRequestKey);
                string content = await res.Content.ReadAsStringAsync();
                JObject json = JObject.Parse(content);
                // get records nearby
                var nearRecords = GetNearRecords(position, json);
                // add pins nearby
                foreach (JObject item in nearRecords)
                {
                    AddPin(item.GetValue("NAME").ToString(), item.GetValue("DESCRIPTION").ToString(), new Position(Convert.ToDouble(item.GetValue("LAT")), Convert.ToDouble(item.GetValue("LONG"))));
                }
            }
        }
    }
    catch (Exception)
    {
        ...
    }
}
```
違いとしては、Searchにクエリを投げる以下のREST APIのGET文を修正して、返してもらうクーポン情報を絞り込むだけである。
```
// get records from azure search with keyphrase specified 
HttpResponseMessage res = await client.GetAsync(Constants.SearchRequestURL + "\"" + keyphrase.Text + "\"" + ",searchFields\"keyphrases\"" + "&api-key=" + Constants.SearchRequestKey);
```
アプリを実行すると、キーワードによる検索機能もきちんと動作し、関連するクーポンだけが表示されていることが分かる。
![searchlocation_shibuya_filtering.png](/images/5defef7dcc11b0/searchlocation_shibuya_filtering.png "キーワード検索機能")

# まとめ
注意点とデモ動画を以下にまとめる。
## 注意点
「現在地付近のクーポンを表示」、「場所検索ボタンを追加して、Geocoding により緯度経度を取得・その付近のクーポンを表示」、「キーワードでフィルタリングしたクーポンを取得・表示」にて注意点と書いている。これは以下2点の注意点からである。今回はあくまで動くものを見せるデモ環境なので、運用環境なのであれば適切な構成を検討することをお勧めする。

- クーポン情報を取得し、特定の座標付近のレコードだけを抽出する作業をクライアント側で処理しているが、Web APIを立てるべき。レコード数が増えていくとUXが確実に下がるので。
  - [OData 言語による地理空間関数](https://docs.microsoft.com/ja-jp/azure/search/search-query-odata-geo-spatial-functions#examples) も用意されているので、こちらを活用することで Web API も最小化できるかもしれない。今回は、デモ環境なので、サボりました。
- Searchにより抽出・構造化された情報（キーワードなど）は、[ナレッジストア](https://docs.microsoft.com/ja-jp/azure/search/knowledge-store-concept-intro?tabs=portal) と呼ばれる二次利用を想定したDBをSearchが作成してくれるため、実運用の際はこちらへ問い合わせを分けて流したりする方がインデクサへの負荷が軽くなるもしれない。

Search へのクエリの種類などはこちらを参照されたし。
https://docs.microsoft.com/ja-jp/azure/search/search-query-overview

## デモ動画
またもや、ClipChampを使いました。30分で作れるので、是非お試しあれ。

https://youtu.be/ewYRfzSywLM

